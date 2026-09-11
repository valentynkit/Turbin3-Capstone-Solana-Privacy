---
status: draft
last_verified: 2026-09-11
---

# Design

TL;DR: two small Anchor programs. A registry holds one reusable address per recipient. A stealth program owns every one-time account and drives Token-2022 by CPI: configure, fund, sweep, close. Token-2022 and its proof program are used unchanged. Everything below is traced against Token-2022 source (research: token2022-mechanics-verified); confidence is noted per part.

## Components

| Component | Where | Role |
|---|---|---|
| Registry program | on-chain, ours | one meta-address per wallet: spend pubkey (Ed25519), scan pubkey (X25519) |
| Stealth program | on-chain, ours | owns every one-time account (PDA); creates, configures, funds, sweeps, closes by CPI; verifies the recipient's one-time-key signature |
| Token-2022 + ZK ElGamal proof program | on-chain, Solana's | confidential balances, proof verification, auditor key; unchanged |
| Sender CLI | off-chain | derives the one-time account and keys, generates proofs, submits payment |
| Recipient CLI | off-chain | publishes the address, scans, decrypts, generates sweep proofs, signs with the one-time key |
| Relayer (optional) | off-chain | pays fees for a recipient's sweep; holds no keys |

Diagram: assets/architecture-simple.html.

## Keys

```
Recipient long-term: b_spend (Ed25519), b_scan (X25519)
Meta-address:        B_spend, B_scan                       public
Per payment:         e random, E = e·X                     E published
Shared secret:       S = ECDH(e, B_scan) = ECDH(b_scan, E) sender and recipient only
One-time key:        P = B_spend + t·G,  t from S           scalar p = b_spend + t: recipient only
View tag:            1 byte from S                          public, cheap filter
Confidential keys:   ElGamal + AES from S via zk-sdk        sender AND recipient
```

Consequence: the sender can configure and fund, and can read that one account's balance, but cannot sweep. Only the recipient holds p. Nobody without b_scan can link E, P, or the account to B_spend. Exact derivation and checks: crypto.md.

## Accounts

| Account | Owner | Notes |
|---|---|---|
| MetaAddress | registry | seeds ["meta", wallet]; spend_pub, scan_pub, version |
| StealthAccount | stealth program | seeds ["stealth", E]; E, view_tag, P, mint, token_account, state. Doubles as the announcement: discovery is a getProgramAccounts filter on view_tag |
| Token account | Token-2022 | owner = StealthAccount PDA; ImmutableOwner + ConfidentialTransferAccount |
| Proof context accounts | ZK proof program | transient, created and closed around each transfer |
| Mint | Token-2022 | ConfidentialTransferMint, auditor key optional |

## Instructions

Registry: register, rotate, close.

Stealth program, one atomic state transition each:

| Instruction | Signer | CPI | Guard |
|---|---|---|---|
| open | sender | System, Token-2022 initialize | stores E, view_tag, P, mint |
| configure | sender | ConfigureAccount, program signs as owner | key-validity proof via context account |
| fund | sender | confidential Transfer in | once only: state must be configured |
| sweep | any fee payer + Ed25519 precompile instruction signed by P | ApplyPendingBalance, confidential Transfer out, program signs | sysvar address, program id, exact message, nonce |
| close | anyone | EmptyAccount (zero-balance proof) + CloseAccount, program signs | state must be swept; rent to the address named in the sweep |

Design choices [decided]: all proofs through context-state accounts, because our program sits behind a CPI and inline proof offsets are relative to top-level instructions; apply-pending only inside sweep so nobody can poison the decryptable-balance cache; close requires the zero-balance proof Token-2022 demands for confidential accounts.

## One payment

| Step | Who | Txs | Notes |
|---|---|---|---|
| publish | recipient | 1, once | |
| open + configure | sender | 1 to 2 | key-validity proof context |
| fund | sender | about 4 | equality, validity, range proofs; range proof staged across transactions |
| scan | recipient | 0 | filter by view tag, trial ECDH, decrypt |
| sweep + close | recipient or relayer | about 5 | proofs for the outgoing transfer, signature by P |

Sequence diagram: assets/pitch.html, "One payment".

## Costs

Estimated, not measured [likely; from the Foundation's sample client and program constants]:
- about 10 transactions per payment, 5 per side;
- about 223k compute units of proof verification per confidential transfer (equality 6.4k, validity 16.4k, 128-bit range 200k), two transfers per payment;
- about 0.013 SOL locked per payment, 0.004 of it standing until close, the rest reclaimed with the proof context accounts.

Measurement is a week-2 deliverable; numbers move to "verified" then.

## Tech stack

Rust + Anchor 1.x. Token-2022 confidential extension. ZK ElGamal proof program. zk-sdk key derivation (solana-zk-sdk 7.0.1, @solana/zk-sdk 0.5.2). ed25519-dalek hazmat for raw-scalar signing, client side. Ed25519 precompile + instructions sysvar. TypeScript client on @solana/kit, @solana-program/token-2022 0.17, @solana-program/zk-elgamal-proof; Codama codegen. LiteSVM + Mollusk for unit and adversarial tests, Surfpool for integration, devnet for demo.

## Confidence per part

| Part | Confidence | Basis |
|---|---|---|
| PDA-owned confidential account, program-signed configure / apply / transfer / close | high | no on-curve check in Token-2022 owner validation; Occult runs it |
| Sender-side configure with ECDH-derived key | medium | zk-sdk API verified; domain separation unreviewed |
| In-program raw-scalar Ed25519 sweep authorization | high | precompile verifies the algebra; standard sysvar pattern |
| Transaction counts and CU | medium-high | sample client + constants |
| Discovery via program-account filter on view tag | high | standard RPC |
| Privacy claims | medium | residuals documented, no formal analysis |

## Open questions

1. Sender funding source: require an existing confidential balance (clean), or allow a public deposit path (one visible amount)? Recommendation: require confidential source for v1; document the deposit path.
2. Should sweep destination be forced to be a fresh stealth self-payment to blunt consolidation linking? Recommendation: offer it as the default in the CLI, don't enforce on-chain.
