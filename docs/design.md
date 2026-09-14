---
status: draft
last_verified: 2026-09-14
---

# Design

TL;DR: two small Anchor programs. A registry holds one reusable address per recipient. A stealth program owns every one-time account and drives Token-2022 by CPI. Plain recipients are paid by a direct confidential transfer with no program call. Traced against Token-2022 source (research: token2022-mechanics-verified).

## Components

| Component | Where | Role |
|---|---|---|
| Registry program | on-chain, ours | one meta-address per wallet: spend pubkey (Ed25519), scan pubkey (X25519) |
| Stealth program | on-chain, ours | owns every one-time account; open, configure, fund, sweep, close by CPI; checks the one-time key's signature |
| Token-2022 + ZK ElGamal proof program | Solana's, unchanged | confidential balances, proof verification, auditor key |
| Payer client | off-chain | batch: derive keys, generate proofs, pay plain and stealth recipients |
| Recipient client | off-chain | publish, scan, decrypt, sweep |
| Relayer | off-chain, optional | pays fees for sweeps; not needed in the default flow |

Diagram: assets/how-it-works.html.

## Keys

```
Recipient long-term: b_spend (Ed25519), b_scan (X25519)
Meta-address:        B_spend, B_scan                       public
Per payment:         e random, E = e·X                     E public
Shared secret:       S = ECDH(e, B_scan) = ECDH(b_scan, E) payer and recipient
One-time key:        P = B_spend + t·G, t from S           scalar p = b_spend + t: recipient only
View tag:            1 byte from S                          public
Confidential keys:   ElGamal + AES from S via zk-sdk        payer AND recipient
```

The payer can configure and fund, and can read that one balance. Only the recipient holds p, so only the recipient can move funds. Derivation details: crypto.md.

## Accounts

| Account | Owner | Notes |
|---|---|---|
| MetaAddress | registry | ["meta", wallet]; spend_pub, scan_pub, version |
| StealthAccount | stealth program | ["stealth", E]; E, view_tag, P, mint, token_account, state. Doubles as the announcement: discovery is a getProgramAccounts filter on view_tag |
| Token account | Token-2022 | owner = StealthAccount PDA; ImmutableOwner + ConfidentialTransferAccount |
| P system account | System | holds a little SOL from the payer so P can pay the sweep fee |
| Proof context accounts | ZK proof program | transient |
| Mint | Token-2022 | ConfidentialTransferMint; auditor key optional |

## Instructions

Registry: register, rotate, close.

Stealth program, one atomic state transition each:

| Instruction | Signer | CPI | Guard |
|---|---|---|---|
| open | payer | System, Token-2022 initialize | stores E, view_tag, P, mint; also funds P's system account with fee dust |
| configure | payer | ConfigureAccount, program signs as owner | key-validity proof via context account |
| fund | payer | confidential Transfer in | once: state must be configured |
| sweep | **P as transaction signer and fee payer** | ApplyPendingBalance, then confidential Transfer out; program signs | P must match the stored key; nonce; state becomes swept |
| close | anyone | EmptyAccount (zero-balance proof) + CloseAccount | state must be swept; rent to the sweep destination |

Plain recipients: no instruction of ours; the payer's client builds a Token-2022 confidential transfer directly.

Decisions: all proofs via context-state accounts (our program sits behind a CPI; inline proof offsets are relative to top-level instructions); apply-pending only inside sweep so nobody can poison the decryptable-balance cache; sweep authorised by P signing the transaction itself, not by precompile introspection (simpler, and P paying the fee keeps the recipient's wallet out of the picture); close needs Token-2022's zero-balance proof.

## One payout run

| Step | Who | Txs | Notes |
|---|---|---|---|
| publish | each stealth recipient | 1, once | |
| prepare | payer | 0 | derive keys per stealth recipient; generate all proofs locally |
| plain recipients | payer | ~5 each | confidential transfer |
| stealth recipients: open + configure + fund | payer | ~6 each | key-validity, then transfer proofs |
| scan | recipient | 0 | view-tag filter, trial ECDH, decrypt |
| sweep + close | recipient, fee from P | ~5 | to a fresh self-owned account, never to a reused one |

## Costs

Estimated [likely]: plain ≈ 5 txs and ≈ 223k CU of proof verification; stealth ≈ 10–11 txs across both sides; ≈ 0.013 SOL locked per stealth payment until close, 0.004 standing. Batching amortises client proof generation and lets the run use lookup tables; it does not reduce per-recipient transaction count. Measurement is week 2.

## Tech stack

Rust + Anchor 1.x. Token-2022 confidential extension. ZK ElGamal proof program. zk-sdk key derivation (solana-zk-sdk 7.0.1, @solana/zk-sdk 0.5.2). ed25519-dalek hazmat for raw-scalar signing (client). TypeScript on @solana/kit, @solana-program/token-2022 0.17, @solana-program/zk-elgamal-proof; Codama. LiteSVM + Mollusk, Surfpool, devnet.

## Confidence

| Part | Confidence | Basis |
|---|---|---|
| PDA-owned confidential account, program-signed configure / apply / transfer / close | high | no on-curve check in Token-2022 owner validation; Occult runs it |
| Payer-side configure with ECDH-derived key | medium | zk-sdk API verified; domain separation unreviewed |
| Sweep authorised by P as a transaction signer | high | runtime verifies RFC 8032 algebraically; slnt signs this way |
| Transaction counts and CU | medium-high | sample client + constants |
| Discovery via program-account filter | high | standard RPC |
| Privacy claims | medium | residuals documented, no formal analysis |

## Open questions

1. Payer funding source: require an existing confidential balance (clean) or allow a public deposit path (one visible amount). Recommendation: confidential-only for v1.
2. Should the client refuse sweeps to a reused destination, or only warn. Recommendation: refuse by default, flag to override.
3. Batch client: how many recipients per run before lookup tables and proof staging become the bottleneck. Measure in week 2.
