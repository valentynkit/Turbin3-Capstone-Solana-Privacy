---
status: draft
last_verified: 2026-09-14
---

# Arch R3 — candidate v2 single-transaction verification (source-verified, 2026-09-14)

TL;DR: every mechanic candidate v2 leans on holds against current source — EmptyAccount checks the live balance, Transfer subtracts homomorphically and deterministically, approval gating and DisableNonConfidentialCredits behave exactly as assumed, and v1 transactions are already supported end to end in the Rust SDK, LiteSVM, and Surfpool (the only real gap is mainnet-beta feature activation, not library support). Both candidate transactions fit the 4096-byte v1 cap with roughly 1.1-1.4 KB of margin, and CU totals sit at a quarter to a fifth of the 1.4M cap using sourced proof-verification constants; the only unsourced number left is Token-2022's own per-instruction processing overhead.

Sources: `solana-program/token-2022` `main` `program/src/extension/confidential_transfer/processor.rs`, `mod.rs`, and `program/src/processor.rs` (fetched 2026-09-14); `spl-token-2022-interface` 3.1.1 (local cache) `extension/confidential_transfer/{instruction.rs,mod.rs}`; `solana-zk-elgamal-proof-interface` 0.1.3 (local cache) `proof_data/zero_ciphertext.rs`; `solana-message` 4.5.0/4.4.1/4.1.6 (local cache) `versions/mod.rs`; `solana-transaction` 4.1.6/4.3.0 (local cache) `sanitized.rs`; `solana-rpc-client` 4.2.2 (local cache) `rpc_client.rs`; `litesvm` crates.io (latest 0.16.0, not the locally cached 0.10.0) `crates/litesvm/src/lib.rs`; `txtx/surfpool` `main` `crates/core/src/surfnet/svm.rs` and `Cargo.toml`; `anza-xyz/agave` `master` `program-runtime/src/execution_budget.rs`, `svm/src/account_loader.rs`, `svm/src/rent_calculator.rs`, `svm/src/transaction_account_state_info.rs`, `programs/zk-elgamal-proof/src/lib.rs`; `anza-xyz/solana-sdk` `master` `instructions-sysvar/src/lib.rs`, `message/src/sanitized.rs`, `account-view/src/lib.rs`; `anza-xyz/svm` `master` `transaction-context/src/{transaction.rs,instruction_accounts.rs,transaction_accounts.rs}`; `pinocchio` `sdk/src/lib.rs`; `solana-foundation/solana-improvement-documents` `main` `0385-transaction-v1.md`; `solana-zk-sdk` 8.0.0 crate tarball (local build in `/tmp/zkbench`, outside the repo).

## Claims table

| Id | Question | Verdict | Source | Note |
|---|---|---|---|---|
| Q1a | v1 message variant exists in the SDK | VERIFIED | `solana-message-4.5.0/src/versions/mod.rs:63-66`: `pub enum VersionedMessage { Legacy(...), V1(v1::Message) }` | Present as far back as `solana-message` 4.2.4 / `solana-transaction` 4.1.6, not just the newest release. |
| Q1b | solana-rpc-client sends v1 today | VERIFIED | `solana-rpc-client-4.2.2/src/rpc_client.rs:5190` | Test builds and sends `VersionedMessage::V1` through `send_transaction`; whether a given cluster accepts the bytes depends on the SIMD-0296/0385 feature gate, not the client. |
| Q1c | LiteSVM parses v1 | VERIFIED | LiteSVM `main`, `crates/litesvm/src/lib.rs:1212-1230` (`sanitize_transaction_no_verify_inner`) | Delegates generically to `SanitizedTransaction::try_create` with no Legacy/V0-only branch; latest release is 0.16.0, not the 0.10.0 cached locally, so the local dev dependency needs a bump before this can be relied on in CI. |
| Q1d | Surfpool parses v1 | VERIFIED | `surfpool` `main` `crates/core/src/surfnet/svm.rs` (`estimate_fee_for_message` matches `VersionedMessage::V1(_)` explicitly, alongside `Legacy`) | Latest tag `v1.5.0` is behind `main` by 51 commits; recheck this file on the tag actually pinned before depending on it. |
| Q1 conclusion | "If any of these is missing, fall back to context-state accounts" | REFUTED | all of the above | Every layer (solana-message, solana-transaction, solana-rpc-client, LiteSVM, Surfpool) already carries v1-aware code today. The real blocker is mainnet-beta feature activation (`txv1aq4pp281K9um3tnPgkfX8UqtFT6wcVW3hNezGLL`, inactive on mainnet-beta as of epoch 1034, active on devnet, per r2-verified R2-2) — a timing gap, not a missing-library gap. Devnet testing and a mainnet legacy-format fallback both need to stay in the plan until that feature flips, but not because any SDK or test harness lacks v1 support. |
| Q2 | Instructions sysvar / CPI index resolution under v1 | VERIFIED | `agave/svm/src/account_loader.rs:652-668` (`construct_instructions_account`, generic over `SVMMessage`); `solana-sdk/message/src/sanitized.rs:71-187` (`SanitizedMessage::V1` shares one `program_instructions_iter` body with Legacy/V0); `solana-sdk/instructions-sysvar/src/lib.rs:287-309` (`get_instruction_relative`, pure byte-blob parsing); `agave/transaction-context/src/transaction.rs:407-458` (`TransactionContext::push`) | None of these four paths branch on message version. SIMD-0385 itself (fetched) says nothing about the instructions sysvar — it is purely a wire-encoding change. Inline `ProofLocation::InstructionOffset` resolution is unaffected by v1. |
| Q3a | EmptyAccount compares proof against live `available_balance` | VERIFIED | `processor.rs:379-395`, `process_empty_account` | `if confidential_transfer_account.available_balance != proof_context.ciphertext { return Err(TokenError::ConfidentialTransferBalanceMismatch) }` — reads the account's current on-chain field at execution time, no snapshot. |
| Q3b | Transfer updates source `available_balance` by deterministic homomorphic subtraction | VERIFIED | `processor.rs:911-924`, `process_source_for_transfer` | `ciphertext_arithmetic::subtract_with_lo_hi(&available_balance, &lo, &hi)`, checked against the client-supplied `new_source_ciphertext` and written back. Fully deterministic; the client already computes this exact value to build its equality proof. |
| Q3c | Sound to precompute the EmptyAccount proof before sending | VERIFIED, conditional | same as above | Sound only if nothing else touches `available_balance` between proof generation and EmptyAccount's execution in the same transaction — true for the exact ApplyPendingBalance -> Transfer -> EmptyAccount -> CloseAccount order in `sweep`, fragile to reordering. Worth a line in decisions.md. |
| Q4a | `valid_as_destination` rejects a Transfer into `approved = false` | VERIFIED | `mod.rs:161-162` calls `approved()`; `mod.rs:118-124` returns `TokenError::ConfidentialTransferAccountNotApproved` if false | |
| Q4b | ConfigureAccount succeeds and leaves `approved = false` on a mint with `auto_approve_new_accounts = false` | VERIFIED | `processor.rs:288`: `confidential_transfer_account.approved = confidential_transfer_mint.auto_approve_new_accounts;` then `Ok(())` at line 306 | No error branch on this value; ConfigureAccount always succeeds. |
| Q4c | ApproveAccount requires only the mint's CT authority signer | VERIFIED | `processor.rs:340-347`: signer check is `authority_info.is_signer && *authority_info.key == confidential_transfer_mint_authority`, else `MissingRequiredSignature` | No owner/token-account signer check anywhere in `process_approve_account`. Discriminant 3 confirmed in `instruction.rs:109,127`. |
| Q5a | DisableNonConfidentialCredits discriminant | VERIFIED | `instruction.rs:340,365,384,406`, enum position 12 (0-indexed) | Matches r1 mechanics' table. |
| Q5b | Signer = owner only (PDA via invoke_signed works) | VERIFIED | `processor.rs:1281-1296`, `process_allow_non_confidential_credits` uses `Processor::validate_owner` | Same owner/delegate/multisig pattern as every other CT instruction, no extra signer. |
| Q5c | Callable immediately after ConfigureAccount in the same transaction | VERIFIED | `processor.rs:1281-1305` reads only the extension (created by ConfigureAccount at `processor.rs:284`) and validates the owner; no other precondition | No field ConfigureAccount writes blocks this call. |
| Q5d | Base SPL Transfer into `allow_non_confidential_credits = false` fails | VERIFIED | `processor.rs:533-537` calls `confidential_transfer_state.non_confidential_transfer_allowed()`; `mod.rs:140-146` returns `TokenError::NonConfidentialTransfersDisabled` if the flag is false | |
| Q6 | ApplyPendingBalance on zero pending balance is a harmless no-op | VERIFIED | `processor.rs:1196-1248`, `process_apply_pending_balance` | No guard rejects a zero-pending call; it recomputes `available_balance += 0`, resets an already-zero counter. Unconditional sweep call is safe. |
| Q7a | Base CloseAccount gates on `closable()` | VERIFIED | `closable()`: interface `mod.rs:127-135`; call site `processor.rs:1350-1352` | Requires `pending_balance_lo/hi` and `available_balance` all zero ciphertexts. CloseAccount re-checks this condition itself; it does not call EmptyAccount internally, so EmptyAccount must run first as a separate instruction (as `sweep` does). |
| Q7b | ImmutableOwner affects closability | REFUTED | `processor.rs:1298-1388`, no `ImmutableOwner` reference in `process_close_account` | ImmutableOwner blocks owner reassignment, not closure; irrelevant to CloseAccount eligibility. |
| Q7c | CloseAccount destination for lamports can be any account, owner signs independently | VERIFIED | `processor.rs:1298-1302,1379-1388` | Only constraint is `source != destination` (plus an incinerator special case); the owner/authority signer check is independent of who receives lamports, so a PDA owner via `invoke_signed` can direct lamports to P. |
| Q8a | v1 caps: 64 accounts, 64 instructions | VERIFIED | `0385-transaction-v1.md` (fetched): "fails sanitization if `NumInstructions > 64`" and "fails sanitization if `num_addresses > 64`"; "this new v1 transaction format notably does not include address lookup tables" | Matches r2-verified R2-2. |
| Q8b | CU cap 1.4M | VERIFIED | `agave/program-runtime/src/execution_budget.rs:26`: `MAX_COMPUTE_UNIT_LIMIT: u32 = 1_400_000` | |
| Q8c | Max CPI depth 4 | PARTLY | `agave/program-runtime/src/execution_budget.rs:8,10`: `MAX_INSTRUCTION_STACK_DEPTH: usize = 5` (top-level + 4 CPI hops); `MAX_INSTRUCTION_STACK_DEPTH_SIMD_0268: usize = 9` behind a separate feature gate | "4 levels of CPI" is the correct reading of the current default constant, but a newer SIMD-0268 gate raises it to 9 — cite the constant name, not a bare "4," if this goes into a doc that outlives the gate. Design's program -> Token-2022 chain is depth 2 either way, well inside both. |
| Q8d | Token-2022 does not CPI to the ZK program for inline proofs | VERIFIED (double-sourced) | `spl-token-confidential-transfer-proof-extraction-0.6.1/src/instruction.rs:11,99`: `use solana_instructions_sysvar::get_instruction_relative`, called directly | `InstructionOffset` proofs are resolved by reading the instructions sysvar, never by a CPI into the ZK ElGamal proof program. Corroborated independently by the Q2 agent's own read of the same file. |
| Q8e | Per-instruction data size cap distinct from the tx cap | REFUTED | Full `solana-message-4.5.0/src/` tree searched, no such constant found | Only the overall transaction byte cap (4096 under v1) constrains the 1001-byte range-proof instruction. |
| Q8f | Per-transaction cap on the number of ZK verify instructions | REFUTED | `agave/programs/zk-elgamal-proof/src/lib.rs` reviewed in full, no counter found | The practical ceiling is the 1.4M CU budget and the 64-instruction cap, not a dedicated proof-count check. |
| Q9a | Pinocchio can close its own PDA after prior CPIs in the same instruction | VERIFIED | `solana-sdk/account-view/src/lib.rs:322-346` (`AccountView::close`/`close_unchecked`, re-exported by pinocchio `sdk/src/lib.rs:361-362`) | Doc notes data zeroing finalizes "at the end of the instruction... or at the next CPI call" — close-after-CPI is an expected pattern, matching `sweep`'s CPIs-then-close-own-PDA ordering. |
| Q9b | Realloc/ownership ordering pitfall | PARTLY | `agave/transaction-context/src/instruction_accounts.rs:329-357` (`is_owned_by_current_program`, `can_data_be_changed`, `can_data_be_resized`) | No CPI-ordering restriction exists; the only gate is current ownership/writability at the moment of the resize call. Since the StealthAccount PDA stays owned by our program throughout and the prior CPIs act on a different account (the token account), there is no conflict — but this is a "no restriction found," not a citation of an explicit allow-rule. |
| Q9c | Post-close "revival" guard | PARTLY | `agave/svm/src/rent_calculator.rs:188-207` (`transition_allowed`); `agave/svm/src/transaction_account_state_info.rs:105-125` | No check named "revival"; the general rent-state comparison (pre-tx vs post-tx per account) always allows a transition to `Uninitialized` (0 lamports). Since nothing in `sweep` or later instructions references the closed StealthAccount PDA again, the design is safe, but this is inferred from the absence of a blocking rule plus one allow-rule, not a single dedicated citation. |
| Q10 | Byte and CU totals for both transactions | see tables below | | |
| Q11 | Proof generation timing spike | see results below | | |

## Byte table

Compact-array (shortvec) encoding: 1 byte for lengths/data <128, 2 bytes for 128-16383. Signature = 64B + 1B list-length prefix per signer. `[sourced]` = struct size confirmed from source this round or in r1/r2. `[likely]` = candidate's own field layout for an instruction that has not been encoded yet (open, open_D, sweep); counted the same way r2-verified counted the original `open` estimate.

### Payer's transaction (register/open + fund)

| # | Instruction | Accounts | Data (B) | Ix bytes (prog+acc+data) | Source |
|---|---|---|---|---|---|
| 1 | SetComputeUnitLimit | 0 | 5 | 8 | [sourced, r2-verified] |
| 2 | VerifyPubkeyValidity | 0 | 97 | 100 | [sourced, r1 mechanics claim 4] |
| 3 | VerifyCiphertextCommitmentEquality | 0 | 321 | 325 | [sourced] |
| 4 | VerifyBatchedGroupedCiphertext3HandlesValidity | 0 | 545 | 549 | [sourced] |
| 5 | VerifyBatchedRangeProofU128 | 0 | 1001 | 1005 | [sourced] |
| 6 | open | 8 | 112 | 123 | [likely, candidate's own field layout] |
| 7 | Token-2022 Transfer (3 inline proofs) | 4 | 169 | 177 | [sourced data size, r1 mechanics; account count reasoned] |

10 unique account keys (payer, ComputeBudget program, ZK ElGamal proof program, System program, Token-2022 program, mint, StealthAccount PDA, token account PDA, instructions sysvar, P).

Message = header(3) + accountkeys(1+10*32=321) + blockhash(32) + instructions(1+2287=2288) = 2644B. Signatures = 65B (1 signer). **Total = 2709B**, cap 4096B, **margin 1387B (34%)**. Close to the candidate's own "about 2,900 bytes" estimate; both land well under cap either way.

### Recipient's transaction (sweep)

| # | Instruction | Accounts | Data (B) | Ix bytes | Source |
|---|---|---|---|---|---|
| 1 | SetComputeUnitLimit | 0 | 5 | 8 | [sourced] |
| 2 | VerifyPubkeyValidity for D | 0 | 97 | 100 | [sourced] |
| 3 | open_D (create D, InitializeImmutableOwner, InitializeAccount3, ConfigureAccount, DisableNonConfidentialCredits) | 7 | ~38 | ~48 | [likely, not spec'd in candidate at field level] |
| 4 | VerifyCiphertextCommitmentEquality | 0 | 321 | 325 | [sourced] |
| 5 | VerifyBatchedGroupedCiphertext3HandlesValidity | 0 | 545 | 549 | [sourced] |
| 6 | VerifyBatchedRangeProofU128 | 0 | 1001 | 1005 | [sourced] |
| 7 | VerifyZeroCiphertext | 0 | 193 | 197 | [sourced, `zero_ciphertext.rs`: context 96B + proof 96B = 192B + 1B discriminant] |
| 8 | sweep | 8 | 206 | 218 | [likely, candidate's own field layout] |

11 unique account keys (P, D_owner, ComputeBudget program, ZK ElGamal proof program, D token account, System program, Token-2022 program, mint, instructions sysvar, StealthAccount PDA, source token account).

Message = header(3) + accountkeys(1+11*32=353) + blockhash(32) + instructions(1+2450=2451) = 2839B. Signatures = 129B (2 signers: P, D_owner). **Total = 2968B**, cap 4096B, **margin 1128B (28%)**. Close to the candidate's own "about 3,100 bytes" estimate.

Note: this arithmetic uses legacy-style compact-array/header encoding as a proxy for v1's exact wire layout, since SIMD-0385's byte-level field encoding beyond the account/instruction count caps was not independently re-derived this round (see Could not verify). The account/instruction count caps themselves are sourced and both transactions clear them with room to spare (7-8 of 64 instructions, 10-11 of 64 accounts).

## CU table

Proof-verification constants sourced from `agave/programs/zk-elgamal-proof/src/lib.rs:20-32` (r1 mechanics claim 5, re-cited by the Q8 agent this round).

### Payer's transaction

| Item | CU | Source |
|---|---|---|
| VerifyPubkeyValidity | 2,600 | sourced |
| VerifyCiphertextCommitmentEquality | 6,400 | sourced |
| VerifyBatchedGroupedCiphertext3HandlesValidity | 16,400 | sourced |
| VerifyBatchedRangeProofU128 | 200,000 | sourced |
| Proof subtotal | 225,400 | matches candidate's own "about 226k for proofs" estimate almost exactly |
| `open` (2x System allocate/assign, InitializeImmutableOwner, InitializeAccount3, ConfigureAccount, DisableNonConfidentialCredits, System transfer) | [open] | not re-derived; r1 mechanics carries an unverified ~31k estimate for Token-2022's own per-instruction processing on top of proof verification |
| Token-2022 Transfer's own processing (distinct from its 3 proofs, already counted above) | [open] | same caveat |
| **Total** | **~225,400 + [open]**, likely 250-280k | well under the 1.4M cap and under candidate's stated "under 500k" |

### Recipient's transaction (sweep)

| Item | CU | Source |
|---|---|---|
| VerifyPubkeyValidity (D) | 2,600 | sourced |
| VerifyCiphertextCommitmentEquality | 6,400 | sourced |
| VerifyBatchedGroupedCiphertext3HandlesValidity | 16,400 | sourced |
| VerifyBatchedRangeProofU128 | 200,000 | sourced |
| VerifyZeroCiphertext | 6,000 | sourced |
| Proof subtotal | 231,400 | |
| `open_D` (System create, InitializeImmutableOwner, InitializeAccount3, ConfigureAccount, DisableNonConfidentialCredits) | [open] | same caveat, likely comparable to `open`'s overhead |
| `sweep` (ApplyPendingBalance, Transfer, EmptyAccount, CloseAccount, StealthAccount close, System transfer) | [open] | more CPIs than `open`, likely the larger of the two [open] terms |
| **Total** | **~231,400 + [open]**, likely 280-320k | well under the 1.4M cap |

Both totals leave 4-5x headroom under the CU cap even before any [open] overhead is pinned down; Token-2022's own processing cost would have to be roughly 4x the r1 estimate to threaten the 1.4M ceiling, which is not plausible for a handful of state writes and ciphertext arithmetic operations.

## Spike 11 — proof generation timing

Ran to completion, no fix cycles needed. Wrote only to `/tmp/zkbench`, nothing in the repo. `solana-zk-sdk = 8.0.0` (resolves `solana-zk-elgamal-proof-interface = 1.0.0`, `curve25519-dalek = 4.1.3`). No separate `spl-token-confidential-transfer-proof-generation` dependency was needed: `solana_zk_sdk::zk_elgamal_proof_program` re-exports free-function builders (`build_pubkey_validity_proof_data`, `build_ciphertext_commitment_equality_proof_data`, `build_batched_grouped_ciphertext_3_handles_validity_proof_data`, `build_batched_range_proof_u128_data`) that do exactly what was asked.

Mean ms per proof, 20 iterations, release build, single-threaded:

| Proof | Mean ms |
|---|---|
| PubkeyValidityProofData | 0.035 |
| CiphertextCommitmentEqualityProofData | 0.232 |
| BatchedGroupedCiphertext3HandlesValidityProofData | 0.405 |
| BatchedRangeProofU128Data | 16.316 |

The range proof dominates cost by 40-80x over the sigma proofs. N=50 (200 proofs total, `std::thread::scope` with a 10-thread pool, `available_parallelism()` on this machine): **202.5ms total wall-clock** (serial estimate would be ~850ms; ~4.2x speedup, consistent with range-proof-bound work). At this rate, generating proofs for a 20-recipient run (80 proofs, ignoring the zero-ciphertext proof needed per sweep) is on the order of 100ms of CPU-bound work, not a bottleneck next to network round trips.

## Consequences for candidate v2

- **v1 SDK/tooling readiness is not the risk the candidate flagged.** Line 27's "legacy fallback named, not built" framing treats missing library support as the trigger condition, but every layer (solana-message, solana-transaction, solana-rpc-client, LiteSVM, Surfpool) already carries v1-aware code today. The actual trigger is mainnet-beta's feature activation timing, which the candidate should track directly (poll the feature account) rather than infer from library gaps that do not exist. The local litesvm dependency (0.10.0 cached) is stale against the 0.16.0 release and should be bumped before S1 runs.
- **S6 is now settled, confirming line 94's assumption.** `valid_as_destination` rejects confidential credits before `ApproveAccount`, `ConfigureAccount` succeeds unconditionally and just leaves `approved = false`, and `ApproveAccount` needs only the mint authority's signature. The "no dust window before approval, `apply` absorbs the gap after" design holds exactly as candidate v2 describes for approval-required mints (R3-3 in the candidate's open list is now unblocked on the mechanics side; the remaining question is a positioning/ownership decision, not a technical one).
- **The `sweep` ordering (ApplyPendingBalance -> Transfer -> EmptyAccount -> CloseAccount) is sound but order-dependent, not order-independent.** The client can precompute the EmptyAccount zero-ciphertext proof before sending only because nothing between proof generation and EmptyAccount's execution touches `available_balance` except the one Transfer whose result is itself deterministic and precomputable. Any future reordering or inserted instruction between Transfer and EmptyAccount in the same transaction breaks this — worth a one-line invariant in decisions.md, not just an implicit ordering in the instruction table.
- **CPI depth and instruction/account caps are not a constraint at any plausible scale.** Both transactions use 7-8 of the 64-instruction cap and 10-11 of the 64-account cap under v1, sit at depth 2 of a 4-deep (or 8-deep, under SIMD-0268) CPI budget, and use roughly a quarter of the CU cap even before Token-2022's own processing overhead is pinned down. R3-1 (candidate's own open question) is answered: nothing found unsound at the Token-2022 or runtime level for this design.
- **Byte margins are healthy but the exact v1 wire-encoding wasn't independently re-derived.** Both transactions land within about 30% of the 4096-byte cap using legacy-style compact-array arithmetic as a proxy for v1's field encoding. The account/instruction count caps (the only caps SIMD-0385 actually specifies) are cleared with room to spare regardless of the exact byte-level format, so this doesn't change the go/no-go, but S7 (measured size once encoders exist) should still run rather than trusting this estimate for the final byte budget.

## Could not verify

- SIMD-0385's exact byte-level wire encoding for v1 message headers and account/instruction serialization beyond the 64/64 count caps and the "no lookup tables" statement — the byte tables above use legacy-style shortvec/header arithmetic as a proxy. Re-check against the SIMD text's binary layout section, or against a real encoder's output, before trusting the byte totals to the last hundred bytes.
- Token-2022's own per-instruction processing CU overhead on top of proof verification (the "[open]" rows in both CU tables) — still not re-derived from source or measured; r1's ~31k estimate for a single instruction carries forward unverified. Spike: `simulateTransaction` a real `open` and a real `sweep` on devnet once the program exists, diff against the proof-verification subtotal.
- The `open_D` sub-instruction inside `sweep`'s recipient transaction has no field-level data-size spec in candidate v2 (unlike `open` and `sweep`, which do); the 38-byte estimate here is this report's own guess, not sourced from the design or from an existing encoder. Flag for the same re-check S7 already calls for on `open`.
- Whether the Surfpool `main` behavior cited for Q1d (`VersionedMessage::V1` handled explicitly in `svm.rs`) is present in the actually-pinned release tag (`v1.5.0`, 51 commits behind `main`) — only `main` was checked.
- Exact CPI depth applicable if SIMD-0268 (`MAX_INSTRUCTION_STACK_DEPTH_SIMD_0268 = 9`) is active on the target cluster at ship time versus the default of 5 — doesn't change this design's go/no-go (depth 2 either way) but should be the cited constant, not a bare "4," in any doc that outlives the gate's activation status.
