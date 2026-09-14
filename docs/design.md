---
status: decided
last_verified: 2026-09-14
---

# Design

TL;DR: one Pinocchio program with four instructions, one Rust CLI (client.md). The program sets up a one-time confidential account and hands it to the one-time key; after that the recipient moves funds with plain Token-2022 instructions and our program is never in the funds path. One transaction per side, every proof inline in a 4096-byte v1 transaction. Source-verified in research/2026-09-14-arch-r1 to r5.

Marks: [verified] read in source or on chain, cited; [likely] arithmetic over verified facts; [open] spike pending, see plan.md.

## Components

| Component | Where | Role |
|---|---|---|
| Program | on-chain, ours, Pinocchio 0.11 `no_std` | registry PDAs; creates, configures and hands over one-time accounts; reclaims rent |
| Token-2022 + ZK ElGamal proof program | Solana's, unchanged | confidential balances, proof verification, auditor key; both live on mainnet-beta and devnet since epoch 982 [verified] |
| CLI | off-chain, Rust | payer: register, pay (batch, plain and stealth mixed); recipient: scan, sweep; either: disclose, verify; bench |

No relayer, no indexer, no second program.

## Keys

```
Recipient long-term: b_spend (Ed25519), b_scan (X25519); meta-address B_spend, B_scan
Per payment:         e derived from the run secret, E = e·X, public
Shared secret:       S = ECDH(e, B_scan) = ECDH(b_scan, E)
One-time key:        P = B_spend + t·G, t from S; scalar p = b_spend + t, recipient only
View tag:            1 byte from S, public
Confidential keys:   ElGamal + AES from S via zk-sdk, payer and recipient
Destination owner:   D_owner from the recipient's secret and (E, k), recipient only
```

Recipe and checks: crypto.md.

## Accounts

MetaAddress, PDA `["meta", wallet]`, 86 bytes: discriminator 1, version 1, spend_pub 32, scan_pub 32, scheme_id 2, flags 1, bump 1, reserved 16. Rent 1,489,440 lamports. The wallet is the identity you hand to a payer; register from a fresh wallet if that matters.

Announcement, PDA `["stealth", E, k]`, 69 bytes, rent 1,371,120 lamports:

| Offset | Field | Size |
|---|---|---|
| 0 | discriminator = 2 | 1 |
| 1 | version = 1 | 1 |
| 2 | E | 32 |
| 34 | payer, gets every rent refund | 32 |
| 66 | view_tag | 1 |
| 67 | k | 1 |
| 68 | bump | 1 |

P and the mint are not stored: after open, P is the token account's owner and the mint is in the token account. Scan filter: `dataSize = 69`, `memcmp(0, [2])`, `memcmp(66, [tag])`.

Token account, PDA `["ta", E, k]`, 465 bytes (base plus ConfidentialTransferAccount, no ImmutableOwner), rent 4,127,280 lamports [likely, from the verified per-byte figures]. Owner field: the announcement PDA during open, P after. Close authority: the announcement PDA, so rent can only return to the payer. Allocated with allocate, transfer-of-deficit, assign, so a one-lamport pre-fund cannot block it.

P: a system account with no data, funded inside open, drained to zero inside the recipient's transaction [verified: a fee payer may end a transaction at zero lamports].

Every field is a byte array; the zero-copy cast is aligned at 1. All layouts and encoders live in a `no_std` crate shared by program and CLI.

## Instructions

| # | Name | Signers | CPIs in order | Guards |
|---|---|---|---|---|
| 0 | register_meta | wallet | System | spend_pub decompresses and `L·spend_pub = identity` via curve25519 syscalls, about 2,336 CU [verified]; scan_pub nonzero |
| 1 | close_meta | wallet | none | PDA from signer |
| 2 | open | payer | System allocate + assign for both PDAs; Token-2022 InitializeAccount3 (owner = announcement PDA), ConfigureAccount (inline proof offset, `maximum_pending_balance_credit_counter = 1`, payer-supplied `decryptable_zero_balance`), DisableNonConfidentialCredits, SetAuthority CloseAccount to the announcement PDA, SetAuthority AccountOwner to P; System transfer to P | PDAs from (E, k) with canonical bump; mint is Token-2022 with ConfidentialTransferMint and auto-approves; not initialized; P decompresses (159 CU) |
| 3 | reclaim | none | Token-2022 CloseAccount signed by the announcement PDA, lamports to payer; close the announcement, lamports to payer | token account's pending and available balances are zero (Token-2022 enforces this in CloseAccount); payer account matches the stored one |

Accounts for open: payer, announcement, token account, mint, P, instructions sysvar, System, Token-2022. Data: E 32, P 32, k 1, view_tag 1, decryptable_zero_balance 36, lamports_for_P 8, bump 1.

Accounts for reclaim: announcement, token account, payer, Token-2022.

Funding is not an instruction. The payer's Token-2022 Transfer into the new account is a top-level instruction in the same transaction as open. Sweeping is not an instruction either: the recipient signs Token-2022 directly with P.

Why these shapes [verified unless marked]:
- Inline proof offsets resolve against the top-level instruction index even under our CPI, so no context-state accounts are needed once the transaction can carry the proofs.
- SetAuthority AccountOwner is refused only when ImmutableOwner or CpiGuard is present; it needs the current owner's signature, which the program gives by invoke_signed; the new owner needs no account and no signature. Nothing in ApplyPendingBalance, Transfer, EmptyAccount or CloseAccount is tied to the owner at configure time. The credit cap has no setter after configure.
- Handing the account to P takes our program out of the funds path. Before, a bug in a program-signed sweep could brick funds, and an upgrade authority colluding with a payer that holds the ElGamal key could move them; ImmutableOwner blocked the upgrade authority alone, not the pair. After, the recipient depends on Token-2022 and nothing else, and any wallet that speaks confidential transfers can recover the money with the key P.
- CloseAccount authorises against the close authority when one is set, so pinning it to the announcement PDA makes the rent refund to the payer a guarantee rather than a courtesy, and reclaim is the only way to close both accounts, together.
- ConfigureAccount enables public credits unconditionally; one base unit sent to the public balance would block CloseAccount forever. DisableNonConfidentialCredits before handover closes that; only the owner can re-enable it.
- With the credit counter capped at 1, the payer's own transfer is the only credit that can land before an apply. ApplyPendingBalance resets the counter, so the recipient's apply, transfer and empty must sit in one transaction: there is then no boundary for dust to land in. Atomicity comes from the transaction, not from routing through our program.
- The `expected_pending_balance_credit_counter` field of ApplyPendingBalance is stored, never checked; the client treats it as bookkeeping.
- The AE-encrypted balance cache is not authenticated. The client always decrypts from the ElGamal ciphertexts with its own key.

## One stealth payment

Payer, one transaction, one signature, about 2,700 bytes of 4,096 [likely], about 226k CU of proof verification plus Token-2022 work [verified constants; overhead open]:

```
SetComputeUnitLimit
VerifyPubkeyValidity                              inline, 97 B
VerifyCiphertextCommitmentEquality                321 B
VerifyBatchedGroupedCiphertext3HandlesValidity    545 B
VerifyBatchedRangeProofU128                       1001 B
open                                              offset to the pubkey proof; ends with the account owned by P
Token-2022 Transfer payer -> one-time account     offsets to the three proofs
```

Plain recipients: the same without the pubkey proof and open.

Recipient, one transaction, signed by P and D_owner, about 3,000 bytes [likely], about 231k CU of proofs plus Token-2022 work; no instruction of ours except reclaim:

```
SetComputeUnitLimit
VerifyPubkeyValidity for D
System CreateAccountWithSeed (base D_owner, P pays), InitializeAccount3 (owner D_owner), ConfigureAccount (D_owner signs, offset), DisableNonConfidentialCredits
VerifyCiphertextCommitmentEquality, VerifyBatchedGroupedCiphertext3HandlesValidity, VerifyBatchedRangeProofU128
VerifyZeroCiphertext    over the source balance after the transfer, computed ahead
ApplyPendingBalance, Transfer (full balance to D), EmptyAccount    all signed by P
reclaim                 rent of both accounts to the payer
System transfer of P's remainder to D_owner
```

The zero-ciphertext proof is valid because Transfer updates the source balance by deterministic homomorphic subtraction and EmptyAccount compares the proof against the live balance at execution [verified]. D and D_owner are fresh per payment and derived, so nothing is persisted on the recipient side. A recipient may instead spend the whole balance straight to a counterparty; a partial spend from the one-time account shows the payer the split, because the payer holds that account's key (privacy.md).

Failure model: both transactions are atomic. Payer side: the announcement PDA exists means the payment landed; absent after the last valid block height means retry. Recipient side: the announcement is gone means swept and reclaimed.

## Money

Per stealth payment the payer sends to P: D rent 4,127,280 lamports, fees for two signatures, a priority allowance for the recipient's transaction (about 231k CU of proofs plus Token-2022 work; budget 450k [likely]), a retry margin. About 0.0045 to 0.006 SOL at a fee price still to be measured [open]. The D rent and the remainder end with the recipient; the payer's net cost is fees.

Locked by the payer until the recipient's transaction, then refunded by reclaim: announcement 1,371,120 plus token account 4,127,280 lamports, about 0.0055 SOL. Nothing is locked in proof accounts on either side. Never-swept accounts stay locked and cannot be reclaimed by the payer, since only P can empty them.

Priority fees are the only cost that scales with congestion: the range proof alone is 200,000 CU.

## Fallback

If v1 transactions cannot be used end to end on day 1 (spike S1), the design falls back to research/2026-09-14-arch-r2-candidate-v1 for transaction packing (proof context accounts, several transactions per side) while keeping the handover: the recipient still moves funds with plain instructions and calls our program only for reclaim. Same program state, same keys.

## Confidence

| Part | Confidence | Basis |
|---|---|---|
| PDA-owned setup: program-signed configure, disable, set authority | high | owner validation has no on-curve check; every path cited |
| Handover to P; recipient sweeps with plain instructions | high | SetAuthority and every later authorisation cited (research: arch-r5-handover-verified); not yet run (S3) |
| Inline proofs under CPI | high | runtime index logic cited |
| One transaction per side | medium-high | sizes and caps cited; v1 not yet exercised end to end (S1) |
| Griefing closed | high | counter cap, disabled public credits, atomic recipient transaction, all cited; a mint freeze authority remains |
| Torsion check from Pinocchio | medium | syscalls and CU cited; not yet called from a `no_std` program (S7) |
| Payer-side configure with ECDH-derived key | medium | zk-sdk API verified; domain separation unreviewed |
| Costs | medium | rent exact; priority fees unmeasured |
| Privacy claims | medium | residuals in privacy.md; no formal analysis |
