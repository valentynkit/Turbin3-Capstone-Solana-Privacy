---
status: draft
last_verified: 2026-09-14
---

# Round 3 fresh eyes: could I build this from the document

TL;DR: no, not without going back to the author. The account lists per instruction are incomplete (the Instructions sysvar and the mint are missing from CPIs that need them), and two load-bearing assumptions do not survive contact: batch proof pregeneration against a payer source balance that concurrent sends mutate, and a recipient sweep that assumes no approval step on the two mints the document itself names as realistic.

## 1. The five instructions and the registry pair

The document gives signers, CPI order, and guards per instruction, but never a full ordered account list with writable/signer flags. That is the first gap, and it is not a detail: an implementer cannot write `try_from(&*accounts)` without it.

**register_meta / close_meta.** Accounts are inferable (wallet signer, MetaAddress PDA, System program for register; wallet signer and PDA for close). The MetaAddress byte layout is not in this document, only a pointer to "arch-r1-obvious §2" [open, not in scope]. Guards (torsion-free spend_pub via multiply-by-L, nonzero scan_pub) are checkable in principle, but the document never says which syscall surface performs the curve arithmetic, and Pinocchio's own reference material has no mention of curve25519 syscalls [checked: pinocchio.md has no "curve25519" section]. That is new ground for whoever implements it.

**open.** CPI list is System (allocate/assign twice, transfer to P) and Token-2022 (InitializeImmutableOwner, InitializeAccount3, ConfigureAccount, DisableNonConfidentialCredits). ConfigureAccount with an inline proof offset needs the Instructions sysvar account so the CPI can resolve the offset into the VerifyPubkeyValidity instruction sitting elsewhere in the same transaction. That account is never listed anywhere in the document, for `open` or for `sweep`. Also missing: a guard tying the `P` account address passed in to the `P` value in instruction data — without it nothing stops the two from diverging.

**apply.** No instruction data is specified at all. ApplyPendingBalance needs `expected_pending_balance_credit_counter` and a 36-byte `new_decryptable_available_balance`; the document lists data sizes for `open` and `sweep` but skips `apply` entirely.

**sweep.** The mint account is never mentioned in this instruction's discussion, but Token-2022's confidential Transfer needs it for the mint's auditor ElGamal pubkey used in the range/validity proofs. Same missing-sysvar problem as `open`, worse here because four inline proofs feed three different CPIs (Transfer, EmptyAccount) in one instruction.

Also: the TL;DR claims "four instructions plus a two-instruction registry" (six), but the instruction table has five rows total (register_meta, close_meta, open, apply, sweep). Internal miscount.

## 2. The two transactions, client-side

Payer's transaction: `SetComputeUnitLimit` has no companion `SetComputeUnitPrice`, despite the run engine section describing a live priority-fee sample with a hard cap — that instruction is missing from every transaction skeleton shown. VerifyPubkeyValidity at step 2 is over the ElGamal pubkey derived from `ct_ikm`, not the Ed25519 `P` checked inside `open`; the document never disambiguates the two "keys named P" for a reader who has not internalized crypto.md's derivation tree.

The serious gap: Transfer's three proofs (steps 3-5) are generated against the payer's *current* on-chain confidential balance ciphertext. The run engine says proofs for the whole run are generated once, upfront, with rayon, "before the first send." But each transfer changes the payer's source ciphertext, and the next transfer's equality/range proof is only valid against the balance state at the moment it lands, not at the moment it was computed. Combined with "bounded concurrency, default 8" sends in flight against the same payer source account, this cannot work as written: proof N+1 computed before send N lands does not match the state send N will have produced. Either sends against one source account must be strictly serial (contradicting the concurrency setting), or proofs must be regenerated per-send against fresh state (contradicting "generated before the first send").

Recipient's transaction: which key signs what is stated (P signs as fee payer/authority, D_owner signs ConfigureAccount for D), but D's key derivation is not. Blockhash timing across the two-transaction approval-required case (transaction A, external ApproveAccount, transaction B) is not addressed: nothing says how the client detects approval landed before building B.

## 3. The client run engine

The single-transaction, deterministic-derivation, PDA-existence-check resume model is sound for the common case: since `e_i` is derived from `run_secret`, a crash before send is a no-op on resume, and "wait for last_valid_block_height to pass before declaring retry" avoids a double-send race. That part works.

Two places I would get stuck. First, the batch-proof problem from section 2: on resume after a crash mid-run, does the client regenerate proofs against the then-current source balance, or replay the ones computed before the crash? The document only describes the upfront generation, not a retry-time regeneration path, and the two are not the same thing once other sends have landed in between.

Second, the approval-required mint path has no resume story at all. The single check "StealthAccount PDA exists means landed" cannot distinguish "transaction A landed, B still needed" from "both landed," because both leave the PDA present. The document flags the two-transaction split as a mechanism (line 94) but never revisits it in the run engine's resume logic.

## 4. Recipient flow, b_spend and b_scan to funds in D

Scan: filter by dataSize/tag (given), then for each candidate recompute `t` and check `P` matches (inferable from crypto.md, not restated here, fine). Sweep: build one transaction that creates D, configures it, and drains the stealth account into it.

The document is silent on how D_owner is generated (deterministic from the recipient's own keys, or fresh random needing its own local storage) and on any crash-resume story for the recipient side, unlike the payer's explicit journal.

The larger problem: the document states plainly that on PYUSD and USDG, "every confidential account, plain or stealth, needs the issuer's ApproveAccount before it can receive" (line 32). D is a confidential account, freshly created per payment, with reuse explicitly refused by policy. The recipient's sweep transaction creates, configures, and funds D all in one atomic instruction sequence with no room for an external, asynchronous ApproveAccount step. As written, sweep cannot work on the two mints the document itself calls out as the realistic ones. The payer side got a two-transaction fix for exactly this problem; the recipient side got none.

## 5. Where I would push back

**Folding apply, transfer, empty, close, and the lamport drain into one `sweep` instruction.** The stated reason is real (a credit-counter reset window is a griefing surface), but that concern is addressed once the account is drained by the transfer; EmptyAccount and CloseAccount do not need to be in the same instruction to keep that property. Simpler: `sweep` does apply+transfer only; a separate, non-atomic `close` handles empty+close+refund. Less CPI weight per instruction, and it restores the recovery path R3-2 already asks about.

**Batch proof pregeneration.** Section 2's finding is a correctness problem, not a style one, but the fix is a simplification: generate the account-creation proof (VerifyPubkeyValidity, no shared state) upfront and in parallel; generate each Transfer's three proofs immediately before that specific send, serially per payer source account. Less clever, more correct.

**Raw Ed25519 nonce-bound signing via ed25519-dalek hazmat**, worked around with a nonce-seed label. This is inherent to the scheme (a derived one-time signing key has to sign somehow) so I would not remove it, but I would push for exactly one signing function used by every code path, including retries, so nonce derivation can never diverge by accident.

**Betting the whole single-transaction architecture on a chain feature landing the day after this document is dated**, before spike S1 has run. Not wrong to plan around it, but the document reads as settled where it is still one unexecuted spike away from its own fallback.

## 6. Contradictions with brief.md and crypto.md

brief.md §7 scope: "two Anchor programs." This document: one Pinocchio program, no_std. Direct contradiction on both framework and count; expected during design work, but brief.md is canonical per CLAUDE.md and has not been updated.

brief.md's claims table: "~10 transactions... per stealth payment." This candidate collapses that to one transaction per side (two total, up to roughly four on approval-required mints). brief.md is stale against the very design it is meant to summarize.

brief.md: "about 0.013 SOL locked per payment." Summing this document's own numbers (StealthAccount 1,893,120 + token account 4,155,120 + D's rent 4,155,120 lamports) gives about 0.0102 SOL, short of brief's figure even before subtracting the parts that get refunded. Soft mismatch, not checked further [likely].

No contradiction found against crypto.md; this document's additions (torsion check method, nonce regression test, k in every seed) are consistent extensions, not changes.

## 7. Estimate

One strong Rust engineer, new to Pinocchio and to Token-2022 confidential transfers: program 15-20 days (no_std zero-copy patterns, Token-2022 CPI account ordering and proof-offset mechanics, hand-written encoders tested byte-for-byte, curve syscalls with no reference material to lean on), client 15-20 days (zk-sdk proof integration, HKDF derivation tree, journal and concurrency logic, scanner, CLI), tests 10-15 days (LiteSVM, Mollusk CU assertions, Surfpool, chaos harness, golden-byte and adversarial suites). Call it 45-60 engineer-days, nine to twelve weeks, assuming S1 and S2 pass as hoped.

Top three things that blow this: the 4096-byte transaction format not working end to end on LiteSVM, Surfpool, and devnet by the time S1 runs, which forces the named context-account fallback and is close to a second architecture; the batch-proof-versus-concurrent-sends conflict from section 2 forcing a run-engine redesign mid-build; and the approval-required-mint gap on the recipient side forcing a second sweep transaction flow after the single-transaction one is already built and tested.

## Blocking gaps

1. No Instructions sysvar account in the account list for `open` or `sweep`, despite both CPI-ing inline-proof instructions that need it.
2. Mint account missing from `sweep`'s account discussion, needed by Transfer for the auditor ElGamal pubkey.
3. `apply`'s instruction data (expected credit counter, new decryptable balance) is unspecified, unlike `open` and `sweep`.
4. Batch-pregenerated Transfer proofs are invalid once the payer's source balance changes; concurrency of 8 against one source account guarantees this happens.
5. Recipient sweep is one atomic transaction but D needs issuer ApproveAccount on PYUSD/USDG per this document's own mint findings; no two-transaction path exists on the recipient side.
6. D_owner's derivation and recovery are unspecified; no resume/journal story for the recipient flow.
7. MetaAddress's byte layout is referenced, not given, in this document.
8. Resume logic cannot distinguish "transaction A landed, B pending" from "both landed" on approval-required mints.
9. TL;DR instruction count (six) does not match the instruction table (five).
10. brief.md's scope ("two Anchor programs") and cost/transaction-count figures are stale against this candidate; needs reconciliation before either doc is decided.
