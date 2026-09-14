---
status: draft
last_verified: 2026-09-14
---

# Architecture candidate v1 (synthesis of round 1)

TL;DR: one Pinocchio program with five instructions, a Rust client, funding done by a plain confidential transfer inside the same transaction as open and configure. Round 1 sources: arch-r1-obvious, arch-r1-creative, arch-r1-mechanics-verified, arch-r1-crypto-privacy-skeptic, arch-r1-ops-cost-skeptic (all 2026-09-14).

Marks: [verified] cited to source in a round-1 file; [likely] inference from verified facts; [open] to be settled in round 2 or by a spike.

## What round 1 settled

| Point | Decision | Source |
|---|---|---|
| Program count | one program; registry is a PDA family inside it, never read on-chain | obvious §1, creative 7 |
| Fund instruction | none; the payer funds with an ordinary Token-2022 confidential Transfer, same code path as plain recipients | obvious §3, creative 1, mechanics claim 1 |
| Proof placement | inline instruction offsets resolve correctly under CPI [verified, mechanics claim 3]; small proofs (pubkey validity, zero ciphertext) go inline, the three Transfer proofs go through context-state accounts because they total 1864 bytes [verified, mechanics claim 4] | mechanics 3, 4; creative 8 |
| Batch resume | deterministic ephemeral keys `e_i = HKDF(run_secret, run_id || i)`, append-only journal, check on-chain state before every step, retry of open that hits AlreadyInitialized is success | obvious §5, creative 12, ops 1 |
| Primary client | Rust CLI; proof generation with solana-zk-sdk, rayon parallel | obvious §5, creative 14, ops spike 1 |
| Dust is not dust | P must hold the sweep's transient proof rent plus the destination account's rent plus fees, about 0.014 SOL, most of which ends in the recipient's hands | obvious §2, ops 4, creative 17 |
| Sweep destination | created and paid by P, never by a wallet the recipient can be identified by | obvious §6, crypto M2 |
| Rent at close | goes to a `rent_refund` address the payer sets at open; close is permissionless given a zero-balance proof | obvious §3, creative 11 |
| Auditor path | self-authenticating disclosure receipt `{E, k, ct_ikm}` verified against the on-chain ElGamal pubkey; mint auditor key is secondary | creative 16 |
| Mint policy | probe `auto_approve_new_accounts` on target mints before anything else; go/no-go for stealth mode on that mint | creative 15 |
| Stack pins | pinocchio 0.11.2, solana-zk-sdk 8.0.0 (released today), @solana/zk-sdk 0.5.2 runs in Node | mechanics 6, 8 |
| ZK ElGamal proof program | live on mainnet-beta and devnet since epoch 982 | mechanics 7, ops 10 |
| Lookup tables | shrink account bytes, not proof bytes; they do not cut the Transfer's context-account transactions | ops 9, mechanics 4 |

## Program

Name placeholder: `payd`. Pinocchio 0.11, `no_std`, one-byte discriminator plus one-byte version on every account, all fields byte arrays so the zero-copy cast is sound (obvious §2).

### Accounts

MetaAddress, PDA `["meta", wallet]`, 86 bytes: spend_pub (Edwards, 32), scan_pub (Montgomery u, 32), scheme_id (2), flags (1), bump (1), reserved (16). Created and closed by the wallet. Register checks spend_pub decompresses and is torsion-free via the curve25519 syscalls and scan_pub is nonzero (obvious §3, crypto M3).

StealthAccount, PDA `["stealth", E]`, about 150 bytes: E (32), P (32), mint (32), rent_refund (32), created_slot (8), view_tag (1), state (1: 1 Live, 2 Swept), bump (1), reserved. It is the announcement: scan is `getProgramAccounts` with `dataSize` and `memcmp` on discriminator and view_tag. The token account address is not stored; it is the PDA `["ta", E]`.

Token account, PDA `["ta", E]`, owner field = StealthAccount PDA, 469 bytes with ImmutableOwner and ConfidentialTransferAccount, rent 0.00416 SOL [verified, mechanics 9]. Allocated with allocate plus transfer-of-deficit plus assign, never `create_account`, so one-lamport pre-funding cannot block it (obvious §2).

P system account: no data. Funded by the payer inside open with a computed amount (section "Money on P"). Drained to the recipient's destination owner in the last recipient transaction.

Proof context accounts: keypair accounts owned by the ZK ElGamal proof program, authority = the party that created them (payer or P), closed in the same transaction that consumes them [verified, mechanics 3].

Why keep the record instead of stuffing E into `decryptable_zero_balance` (creative 4): scanning Token-2022 accounts by mint plus a tag byte means a `getProgramAccounts` over every token account of that mint, millions on mainnet for USDC; scanning our own program's accounts keeps the set to our payments. The record also carries state and rent_refund. Cost 0.002 SOL, recovered at close.

### Instructions

| # | Name | Signers | What it does | Guards |
|---|---|---|---|---|
| 0 | register_meta | wallet | create MetaAddress | torsion-free spend_pub, nonzero scan_pub, PDA from signer |
| 1 | close_meta | wallet | close MetaAddress, rent to wallet | PDA from signer |
| 2 | open | payer | create StealthAccount and token account; InitializeImmutableOwner, InitializeAccount3; ConfigureAccount by CPI with the stealth PDA signing, proof via inline offset to a top-level VerifyPubkeyValidity; `maximum_pending_balance_credit_counter = 1`; System transfer to P | PDAs from E with canonical bump; mint has ConfidentialTransferMint; not initialized; E and P are 32-byte points that decompress [open: whether to check P on-curve] |
| 3 | sweep | P (transaction signer and fee payer) | ApplyPendingBalance then confidential Transfer of the full available balance to `destination`, both by CPI with the stealth PDA signing; state to Swept | state Live; signer key == stored P; destination != source; the three Transfer proofs via context accounts owned by P |
| 4 | close | anyone | EmptyAccount (inline zero-ciphertext proof) then CloseAccount by CPI; close StealthAccount; all rent to rent_refund | state Swept |

Rotate is close_meta plus register_meta. Funding is not an instruction.

Instruction data for the Token-2022 CPIs is hand-encoded against spl-token-2022-interface 3.1.1 discriminants (`[27, 2]` ConfigureAccount, `[27, 8]` ApplyPendingBalance, `[27, 7]` Transfer, `[27, 4]` EmptyAccount, `[9]` CloseAccount) [verified, mechanics table]. Every encoder gets a golden-byte test against the interface crate's builder (ops 5).

### The payer's one transaction per stealth recipient

Preceded by three context-state transactions (equality 320 B, ciphertext validity 544 B, range U128 1000 B; the range proof may need two transactions under the 1232-byte cap, one under 4096). Then one v0 transaction with a per-run lookup table:

1. ComputeBudget SetComputeUnitLimit
2. ZK ElGamal VerifyPubkeyValidity, proof inline (96 B)
3. `open` (creates both PDAs, configures with offset to instruction 2, funds P)
4. Token-2022 confidential Transfer from the payer's account to the new token account, three proofs by context account
5. three CloseContextState, rent back to the payer

There is no public window between configure and fund, so the round-1 critical finding C1 (third-party credit-counter griefing of an empty account) has no target. After the atomic funding, `maximum_pending_balance_credit_counter = 1` makes Token-2022 reject any further credit [open: confirm the counter comparison in `process_transfer` and confirm a rejected credit does not touch the account]. Estimated size under 1232 bytes with the lookup table holding the static program IDs, the mint, the payer's token account and both PDAs [likely; measure].

Payer transactions per stealth payment: 4 to 5 under the 1232-byte cap, 1 to 2 if the 4096-byte transaction size (SIMD-0296 / SIMD-0385) is active [open, time-sensitive: reported for mainnet around 2026-09-15, mechanics "could not verify"]. Plain recipients: 4 to 5 (three contexts plus the transfer).

### The recipient's transactions per stealth payment

1. create destination token account D (owner: fresh keypair D_owner held by the recipient) and ConfigureAccount with inline pubkey validity; P pays fees and rent, D_owner signs configure
2. three context-state transactions for the sweep Transfer proofs, authority P
3. `sweep` plus three CloseContextState
4. `close` (zero-ciphertext proof inline) plus System transfer of P's remaining lamports to D_owner

Six to seven transactions under 1232 bytes, two to three under 4096. Everything is signed by P or D_owner, both recipient-held keys that appear nowhere else.

## Money on P

Per stealth payment the payer transfers to P: D rent 0.00416 plus transient sweep-context rent 0.00854 (recovered to P at close of the contexts, then drained to D_owner) plus fees for about seven transactions at a priority-fee allowance to be set from a measured distribution (ops spike 4), plus a retry margin. About 0.014 SOL leaves the payer; all but fees end with the recipient. The recipient client refuses to start a sweep below a computed floor rather than burn fees on a landed-but-failed attempt (ops 4). If P is still short, the only top-up path links the recipient (crypto M2); the client warns and stops.

Payer locked until close: record 0.002 plus token account 0.00416 plus payer-side transient contexts (recovered in the same transaction). Standing cost after close: fees plus what the recipient keeps.

## Keys and derivation

Unchanged from crypto.md except: the register instruction performs the torsion check (crypto M3); the sweep is always the full available balance in one shot (crypto M4); the raw-scalar signing nonce derivation gets a mandatory regression test that R differs across two messages signed by the same p (crypto C2); the BIP-352 citation is narrowed to the tweak derivation only, and HKDF multi-info independence is named as the actual argument (crypto M5). The k index stays; `k = 0` for the first account per handshake.

One ephemeral key per run (creative 5) is deferred: it cuts recipient scan work but groups every recipient of a run under one E, and needs the external review first.

View tag stays one byte; two bytes (creative 6) is a scan optimisation for a scale we are not at.

## Clients

One Rust workspace: `payd-core` (no_std layouts, seeds, discriminators, errors, shared with the program), `payd-crypto` (derivation tree, test vectors), `payd-client` (run engine, proofgen, journal, scanner), `payd-cli`. Commands: `register`, `pay` (batch, plain and stealth mixed), `scan`, `sweep`, `close` (payer side reap and recipient side), `disclose`, `verify` (receipt verifier), `bench`.

Batch engine: plan, prepare (derive, generate every proof before signing anything), stage (contexts and lookup table warmed one slot ahead), execute (bounded concurrency, sign immediately before send, re-sign on blockhash expiry), reclaim (close contexts, deactivate the table), done. Priority fee from a measured p75 with a hard cap and pause-and-alert, never silent retries (ops 3).

TypeScript: [open, team decision]. Options: none (creative 14); a hand-written Codama IDL generating a thin client for register, scan and the demo (obvious §5, ops 7). The batch engine is Rust either way.

Relayer: removed from the design and the brief (creative B2). P pays its own fees.

## Testing

LiteSVM is the functional gate [open: confirm the ZK ElGamal builtin runs in LiteSVM and Mollusk, ops 6]; Mollusk for CU benches with CI assertions (creative 19); Surfpool embedded for RPC-dependent paths, `getProgramAccounts` scanning, lookup tables and the recorded demo; devnet as release smoke only. Chaos harness: kill the batch client at every step index of a 20-recipient run and assert convergence (creative 13). Adversarial suite as listed in obvious §8, plus: credit after funding is rejected; sweep with a partial amount is rejected; second signature by P reuses no R.

## Costs to publish

Replace every "~10 tx, 0.013 SOL" with a measured table: transactions per payment per side under both transaction-size regimes, CU per instruction from Mollusk, SOL fronted by the payer, SOL ending with the recipient, SOL burned as fees, at N = 1, 10, 50.

## Contested or open, for round 2

| Id | Question | Why it matters |
|---|---|---|
| R2-1 | Does Token-2022 reject a credit that would exceed `maximum_pending_balance_credit_counter`, and is the account untouched on rejection? Can an account with a non-zero pending balance be emptied and closed? | one-time enforcement by the token program; rent recovery after any grief |
| R2-2 | Is the 4096-byte transaction size active on mainnet-beta and devnet today, and in LiteSVM and Surfpool? | transaction counts halve or better; changes the whole batching design |
| R2-3 | Does the combined open + configure + transfer + closes transaction fit 1232 bytes with a lookup table? | if not, open and fund split and C1 returns |
| R2-4 | Can the fee payer P drain to exactly zero lamports in its last transaction? | otherwise 0.00089 SOL strands per payment |
| R2-5 | Is the ZK ElGamal proof program a builtin in LiteSVM and Mollusk? | decides the test loop |
| R2-6 | `auto_approve_new_accounts` on USDC, PYUSD, USDG mainnet mints | stealth mode viability per mint |
| R2-7 | Should open check that P is on-curve? A payer who supplies garbage P only burns its own money | CU vs safety |
| R2-8 | Context-state authority: can P (a plain system account) be the authority for contexts consumed by a CPI whose owner is a PDA? | recipient-side proof plumbing |
| R2-9 | TypeScript client: none, or Codama-generated thin client | team allocation |
| R2-10 | Is a per-payment StealthAccount record the right announcement, or a per-run announcement (one E, N tags) once reviewed? | scan cost at scale, privacy grouping |
