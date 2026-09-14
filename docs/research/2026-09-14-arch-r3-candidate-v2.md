---
status: draft
last_verified: 2026-09-14
---

# Architecture candidate v2 (synthesis of round 2)

TL;DR: with the 4096-byte transaction format every proof goes inline, so a stealth payment is one payer transaction and one recipient transaction, with no proof context accounts, no lookup tables, no transient rent, and no window for griefing. The program has four instructions plus a two-instruction registry.

Inputs: arch-r2-candidate-v1, arch-r2-verified, arch-r2-crypto-privacy-skeptic, arch-r2-design-skeptic, arch-r2-ops-cost-skeptic (all 2026-09-14). Marks: [verified] cited in one of those files; [likely] inference; [open] to settle in round 3 or a spike.

## What changed from v1 and why

| Change | Reason | Source |
|---|---|---|
| All proofs inline in one transaction; no context-state accounts, no lookup table | inline offsets resolve under CPI [verified, r1 mechanics 3]; v1 transactions carry 4096 bytes, active on devnet, mainnet expected 2026-09-15 [verified, r2-verified R2-2]; v1 format has no lookup tables anyway | r2-verified R2-2, R2-3; r2 ops 1 |
| `open` also calls DisableNonConfidentialCredits | ConfigureAccount enables public credits unconditionally; one base unit sent to the public balance blocks CloseAccount forever [verified] | r2 crypto A |
| `sweep` does apply, transfer, empty, close, refund in one instruction; no separate `close`, no Swept state | apply resets the credit counter, so any gap between sweep and close is a griefing window [verified]; one transaction has no gap | r2 crypto B; r2 design 5 |
| `apply` instruction, payer-gated | on mints with `auto_approve_new_accounts = false` the issuer's ApproveAccount sits between open and fund, reopening a dust window; payer resets the counter and funds in one transaction | r2-verified R2-6; r1 crypto C1 |
| P holds far less | no transient context rent on either side | r2 ops 2 |
| Every keypair the client creates is derived from `run_secret` | crash resume must never depend on the journal for an address | r2 ops 4 |
| `decryptable_zero_balance` (36 B) added to `open` data | ConfigureAccount needs it; payer computes it under the AES key from S | r2 ops 3 |
| PDA seed `["stealth", E, k]` | free now, a migration later | r2 crypto D4 |
| `open` checks P decompresses (159 CU) | turns a wasted payment into an error | r2 crypto D2 |
| Torsion check by multiply-by-L, about 2,336 CU | blacklist of eight points misses mixed-order points | r2 crypto D1 |
| `run_secret` storage specified | losing it before the payer's rent is reclaimed or an audit is answered is data loss | r2 design 1 |
| Legacy fallback named, not built | if v1 transactions are unusable in LiteSVM or the Rust SDK by week 2, the v1-candidate design (context accounts, five to seven transactions per side) is the fallback | r2-verified could-not-verify |

## Facts that reshape the brief

- USDC is legacy SPL Token with no confidential extension [verified, r2-verified R2-6]. It cannot carry this rail at all.
- PYUSD and USDG are Token-2022 with ConfidentialTransferMint, `auto_approve_new_accounts: false`, same Paxos authority, no auditor key [verified, r2-verified R2-6]. On these mints every confidential account, plain or stealth, needs the issuer's ApproveAccount before it can receive. The stealth mode works there only if the issuer approves program-owned accounts, by policy or by an automated approver.
- The payer can follow the funds it paid through every later hop, as on any public ledger, because account addresses are plain in every instruction; amounts stay hidden. This is not a break: the payer already knows who it paid. What the payer must not learn is the recipient's other income, and that holds as long as the recipient does not merge payer-visible accounts with accounts other payers can see [likely, r2 crypto C reinterpreted].

## Program

One Pinocchio 0.11 program, `no_std`. Accounts carry a one-byte discriminator and a one-byte version; all fields are byte arrays so the zero-copy cast is aligned at 1.

### Accounts

MetaAddress, PDA `["meta", wallet]`, 86 bytes, exact layout as arch-r1-obvious §2. Register checks spend_pub decompresses and `L * spend_pub == identity` via the curve25519 syscalls, and scan_pub is nonzero.

StealthAccount, PDA `["stealth", E, k]`, 144 bytes:

| Offset | Field | Size |
|---|---|---|
| 0 | discriminator = 2 | 1 |
| 1 | version = 1 | 1 |
| 2 | ephemeral_pub E | 32 |
| 34 | one_time_pub P | 32 |
| 66 | mint | 32 |
| 98 | payer | 32 |
| 130 | created_slot LE | 8 |
| 138 | view_tag | 1 |
| 139 | k | 1 |
| 140 | bump | 1 |
| 141 | state: 1 Live | 1 |
| 142 | reserved | 2 |

`payer` is the rent refund address and the signer allowed to call `apply`. The scan filter is `dataSize = 144`, `memcmp(0, [2])`, `memcmp(138, [tag])`. Token account PDA `["ta", E, k]`, 469 bytes, owner field = StealthAccount PDA, allocated by allocate plus transfer-of-deficit plus assign.

P: a system account with no data, funded inside `open`, drained to zero inside `sweep` [verified that a fee payer may end at zero, r2-verified R2-4].

### Instructions

| # | Name | Signers | CPIs, in order | Guards |
|---|---|---|---|---|
| 0 | register_meta | wallet | System | torsion-free spend_pub, nonzero scan_pub |
| 1 | close_meta | wallet | none | PDA from signer |
| 2 | open | payer | System allocate/assign twice; Token-2022 InitializeImmutableOwner, InitializeAccount3, ConfigureAccount (inline proof offset, `maximum_pending_balance_credit_counter = 1`, payer-supplied `decryptable_zero_balance`), DisableNonConfidentialCredits; System transfer to P | PDAs from (E, k) with canonical bump; mint is Token-2022 with ConfidentialTransferMint; not initialized; P decompresses |
| 3 | apply | payer (matches stored payer) | Token-2022 ApplyPendingBalance | state Live |
| 4 | sweep | P (fee payer) | Token-2022 ApplyPendingBalance, Transfer (full available balance, three inline proofs), EmptyAccount (inline zero-ciphertext proof), CloseAccount (rent to payer); close StealthAccount (rent to payer); System transfer of P's remaining lamports to `drain_to` | state Live; signer key == stored P; destination != source |

Data for `open`: E 32, P 32, k 1, view_tag 1, decryptable_zero_balance 36, lamports_for_P 8, bump 1, plus discriminator: 112 bytes. Data for `sweep`: new_decryptable_available_balance after apply 36, new_source_decryptable_available_balance after transfer 36, transfer auditor ciphertexts lo/hi 128, proof offsets 5, plus discriminator: 206 bytes.

Instruction encoders are hand-written against spl-token-2022-interface 3.1.1 [verified discriminants, r1 mechanics] and tested byte-for-byte against the interface crate's builders.

Funding is not an instruction. The payer's Token-2022 Transfer into the new account is a top-level instruction in the same transaction as `open`.

### The payer's transaction, one per stealth recipient

```
1  ComputeBudget SetComputeUnitLimit
2  ZK VerifyPubkeyValidity            (inline, 97 B)
3  ZK VerifyCiphertextCommitmentEquality (321 B)
4  ZK VerifyBatchedGroupedCiphertext3HandlesValidity (545 B)
5  ZK VerifyBatchedRangeProofU128     (1001 B)
6  open                               (offset to 2)
7  Token-2022 Transfer payer -> new account (offsets to 3, 4, 5)
```

Size about 2,900 bytes [likely; v1 cap 4,096, 64 accounts, 64 instructions]. CU about 226k for proofs plus Token-2022 and System work, under 500k [likely]. One signature. Plain recipients: instructions 1, 3, 4, 5, 7.

On approval-required mints: transaction A is instructions 1, 2, 6; the issuer runs ApproveAccount; transaction B is 1, 3, 4, 5, `apply`, 7. Before approval no confidential credit can land and public credits are disabled, so the only dust window is between approval and B, and `apply` inside B absorbs it [likely; confirm `valid_as_destination` checks `approved`, round 3].

### The recipient's transaction, one per stealth payment

```
1  ComputeBudget SetComputeUnitLimit
2  ZK VerifyPubkeyValidity for D      (inline)
3  System create D (P pays), Token-2022 InitializeImmutableOwner, InitializeAccount3 (owner D_owner), ConfigureAccount (D_owner signs, offset to 2), DisableNonConfidentialCredits
4  ZK VerifyCiphertextCommitmentEquality
5  ZK VerifyBatchedGroupedCiphertext3HandlesValidity
6  ZK VerifyBatchedRangeProofU128
7  ZK VerifyZeroCiphertext            (over the source balance after the transfer, precomputed)
8  sweep                              (offsets to 4, 5, 6, 7; drain_to = D_owner)
```

Signers: P and D_owner, both recipient-held, both unseen anywhere else. Size about 3,100 bytes [likely]. The zero-ciphertext proof is over the available balance the account will hold after the transfer, which the client computes homomorphically before sending [likely; round 3 confirms EmptyAccount compares the proof ciphertext to the live balance at execution time].

If P's lamports fall short, the transaction is rejected before inclusion when the fee cannot be paid, and it fails atomically after inclusion when D's rent cannot be paid; only the second burns a fee. The client simulates and checks both bounds before sending (r2 ops 7).

### Money on P

lamports_for_P = D rent 4,155,120 + fee for two signatures 10,000 + priority allowance for about 450k CU at the run's chosen price + retry margin. At a p75 price to be measured (r1 ops spike 4) this is about 0.0045 to 0.006 SOL, of which the D rent stays with the recipient. Payer locked until sweep: StealthAccount 1,893,120 plus token account 4,155,120 lamports, both refunded to `payer` at sweep. The payer never needs to act after its one transaction.

## Keys

crypto.md stands, with: the torsion check specified as multiply-by-L; the nonce regression test (two messages, different R) mandatory; `k` in every seed and in the receipt; BIP-352 cited only for the tweak. Per-run E deferred behind the external review.

## Clients

One Rust workspace: `core` (no_std layouts, seeds, encoders, shared with the program), `crypto` (derivation, test vectors), `client` (run engine, proofgen, journal, scanner), `cli`. Commands: register, pay, scan, sweep, disclose, verify, bench.

Run engine. `run_secret` is a 32-byte key generated per run and stored in an encrypted file under the payer's config directory (age or the OS keychain, decided in week 2); losing it before every rent refund has landed or before an audit is answered is data loss and the CLI says so at creation. Every key the run creates is derived from it: `e_i = HKDF(run_secret, run_id || i || "e")`, and nothing else needs a keypair because there are no context accounts. Journal: one fsynced line per (run_id, i, step) with the signature, last_valid_block_height and outcome; never a secret. Before every send the client reads the StealthAccount PDA: exists means the payment landed (the transaction is atomic), absent after the block height passed means retry. Proofs for the whole run are generated before the first send, with rayon, so no blockhash expires during CPU work. Bounded concurrency, default 8. Priority fee from a live `getRecentPrioritizationFees` sample at p75 with a hard cap and pause-and-alert.

Recipient. `scan` filters our program's accounts by tag, trial-decrypts, and lists payments; `sweep` builds the one transaction above with a fresh D_owner per payment and refuses a destination it has seen before unless overridden.

Disclosure receipt `{E, k, ct_ikm}`; `verify` re-derives the keys, checks the ElGamal pubkey against the closed account's history or the still-open account, and decrypts the amount [verified safe to disclose, r2 crypto D3].

TypeScript: none in the core. Teammates own devnet deployment, the demo script, the status page, and a hand-written Codama IDL if they want a TS client for scan and register [open, team].

## Testing

LiteSVM is the gate; it loads the ZK ElGamal builtin by default [verified, r2-verified R2-5]. Mollusk with the `all-builtins` feature for CU assertions in CI [verified]. Surfpool embedded for `getProgramAccounts` scanning and the recorded demo. Devnet for release smoke, funded incrementally from week 1 because of faucet caps. Chaos harness in week 4: kill the client at every step of a 20-recipient run, assert convergence. Golden-byte tests for every encoder. Adversarial suite as arch-r1-obvious §8 plus: public credit after open is rejected; second confidential credit is rejected; sweep with a partial amount is rejected; P signature test for distinct R.

Cut order if behind: TypeScript anything, N=500 numbers, Mollusk as a separate job, the receipt verifier CLI.

## Spikes, in order

| Id | Question | Exit criterion | When |
|---|---|---|---|
| S1 | Can the Rust SDK (solana-transaction, solana-client) build and send v1 transactions; do LiteSVM and Surfpool execute them; does devnet accept 4096 bytes today | one 3 KB transaction with four ZK verifies lands on LiteSVM and devnet | day 1 |
| S2 | LiteSVM runs ConfigureAccount with an inline proof behind a CPI from a PDA-owned program | test passes | day 1 |
| S3 | Proof generation time for the four proof types, solana-zk-sdk 8.0.0, on the dev machine, N=1 and N=50 with rayon | numbers in a table | day 2 |
| S4 | `getRecentPrioritizationFees` distribution over several days for Token-2022 and the ZK program | p50, p75, p95 | week 1, background |
| S5 | EmptyAccount with a precomputed zero-ciphertext proof in the same transaction as the Transfer | test passes | week 2 |
| S6 | `valid_as_destination` rejects confidential credits before ApproveAccount | source line cited | round 3 |
| S7 | Measured serialized size and CU of both transactions once encoders exist | numbers | week 2 |

## Open, for round 3

| Id | Question | Recommendation |
|---|---|---|
| R3-1 | Is anything in the single-transaction design unsound at the Token-2022 or runtime level: instruction ordering, sysvar reads under CPI, 64-account and 64-instruction caps, CU cap | round 3 verifier with source |
| R3-2 | Does folding empty and close into sweep lose any recovery path a real operator needs (e.g. recipient lost D_owner mid-flow) | round 3 skeptic |
| R3-3 | Positioning after the mint findings: own or partner mints that auto-approve, or issuer-approved accounts | owner decision |
| R3-4 | Registry on-chain: keep (sRFC-42 interop, publish-once demo step, 80 lines) or address book only | keep; owner may cut |
| R3-5 | Unclaimed payouts: should the payer be able to reclaim an account never swept after a long timeout | phase two; it breaks "only the recipient can move funds", state it |
