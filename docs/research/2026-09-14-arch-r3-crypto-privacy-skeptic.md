---
status: draft
last_verified: 2026-09-14
---

# R3 — adversarial crypto/privacy review of candidate v2 (2026-09-14)

TL;DR: atomicity closes both round-2 criticals for real, source-verified this round via Token-2022's own `maximum_pending_balance_credit_counter = 1` cap, which blocks third-party credits after funding regardless of transaction boundaries. The new critical is different: on the two approval-required mints the brief names as viable (PYUSD, USDG), the recipient's own sweep transaction cannot fund D, because D's `ConfigureAccount` sets `approved = false` and nothing in `sweep` gets it approved.

Sources read beyond the assigned docs: `solana-program/token-2022` `main`, `program/src/extension/confidential_transfer/processor.rs` (`process_apply_pending_balance`, `process_empty_account`, `process_transfer`, `process_source_for_transfer`, `process_destination_for_transfer`) and `interface/src/extension/confidential_transfer/mod.rs` (`ConfidentialTransferAccount::valid_as_destination`, `closable`, `approved`).

## CRITICAL

**On approval-required mints, the recipient's own sweep transaction cannot fund D — v2 only fixed the payer's side of this problem.** [verified]
Target: candidate's "recipient's transaction" sequence, step 3 and step 8; R3-3.
`process_configure_account` sets `approved = confidential_transfer_mint.auto_approve_new_accounts` (`processor.rs:289`); `valid_as_destination` (`mod.rs:161`) checks `approved()` first and errors if false. The candidate's own "Facts that reshape the brief" gives PYUSD and USDG `auto_approve_new_accounts: false`. Step 3 creates and configures D; step 8's `sweep` transfers into D in the same transaction. On these mints D is `approved = false` at that point, `process_destination_for_transfer` rejects it, and the sweep fails atomically, every time, because D's address doesn't exist for the issuer to approve until the recipient submits this very transaction, and nothing gets an `ApproveAccount` in between. The candidate solved this for the payer's `open` (transaction A/B, `apply` absorbs the gap) but never revisited it for the recipient's sweep. Stealth mode is inoperable on both mints the candidate names as launch targets; only a hypothetical auto-approving mint works.
Cheapest fix: either the issuer runs a standing out-of-band `ApproveAccount` job over newly created program-owned accounts, or D cannot be created fresh per payment on these mints and must be a pre-registered, pre-approved account instead — which reopens consolidation. This blocks R3-3; it decides whether stealth mode ships at all for PYUSD/USDG.

## MAJOR

**`apply`'s cached AE balance is unchecked on-chain; the payer holds the same key and can poison it.** [verified]
Target: v2's `apply` instruction, "payer-gated."
`process_apply_pending_balance` (`processor.rs:1196-1246`) copies `new_decryptable_available_balance` from instruction data straight into account state, no proof, no check against `available_balance`. Payer and recipient derive `ct_ikm` from the same `S` (payer via `ECDH(e, B_scan)`, recipient via `ECDH(b_scan, E)`), so they hold the identical AE key, not just the same ElGamal key. A payer who calls `apply` can write a validly-encrypted but false balance, e.g. zero when the true pending amount is nonzero, to make a naive client think nothing arrived. Recovery is practical: the client can always decrypt `available_balance`/`pending_balance_lo/hi` directly with the shared ElGamal key, and Token-2022's own 16-bit lo / 32-bit hi split is exactly what makes that discrete-log recovery tractable (baby-step-giant-step per half, standard practice for confidential balances). Nothing in crypto.md or v2 says the client must do that recomputation whenever a payer-triggered `apply` may have run; right now the cache looks authoritative.
Cheapest fix: state explicitly that `decryptable_available_balance` is untrusted whenever `apply` was payer-signed, and that `scan`/`sweep` always recompute from ciphertexts, never display the cached AE value as ground truth.

**D and D_owner are directly linked to P inside the sweep transaction itself, and nothing funds a recipient's future spending from D without a new link.** [verified against candidate's instruction table]
Target: item 3; carries R2 finding C forward.
`sweep`'s CPI list has P creating D (step 3) and, at the end, draining P's leftover lamports to `drain_to = D_owner`, both cleartext account metadata in the transaction the payer can already find (it funded P and knows P's address from its own `open`). So D_owner is linked to P, and P to the payer, in one atomic, permanently public step — the same finding R2 made; v2 adds no technical fix, only the existing "sweep to a fresh self-owned account" guidance. What still isn't answered: D_owner gets only a few thousand lamports of dust, not enough for repeated future fees. Real spending out of D needs SOL from somewhere, and topping up D_owner from any identifiable source (an exchange withdrawal, another recipient-controlled wallet) reintroduces exactly the correlation the construction exists to avoid, one hop later and off the program's radar.
Cheapest fix: state explicitly that D_owner's SOL for its first outbound transfer must come from a pool the recipient funds out-of-band and reuses across many D_owners, not an individually-sourced top-up, and flag this as open, not solved.

**"The payer already knows who it paid" holds for payroll, breaks for grants and donations, because the registering wallet is what gets shared to pay at all.** [verified against candidate's register_meta account layout]
Target: item 4; "Facts that reshape the brief" paragraph.
`register_meta`'s PDA is `["meta", wallet]` — the meta-address a recipient publishes to be paid *is* their real on-chain wallet address, permanently linked to `B_scan`/`B_spend` the moment anyone registers. Payroll: true, the payer already has an identity-to-wallet mapping. Grants: the moment a contributor known only by handle hands over a meta-address, the payer learns the real wallet that registered it, before any payment and before D exists. The candidate's framing is honest for payroll and overstated for pseudonymous payers: the leak is through registration, not D. privacy.md's artifact table half-acknowledges this ("MetaAddress: public, once; it is the recipient") but the candidate's narrative paragraph doesn't carry the caveat forward.
Cheapest fix: recommend registering from a throwaway wallet with no other history, funded from a source not otherwise tied to the registrant, and say so directly next to the "payer already knows" claim rather than only in the artifact table.

**The disclosure receipt has no way to find the transaction once the account is closed.** [verified against Token-2022 source and v2's receipt format]
Target: item 7; receipt format `{E, k, ct_ikm}`.
After `sweep`'s `CloseAccount`, `getAccountInfo` on the token account and the StealthAccount PDA returns nothing — but the ElGamal pubkey and every ciphertext needed to verify are still present as public inputs baked into the ZK proof instructions' own data within the sweep transaction, not derived from current account state (confirmed: `process_source_for_transfer` checks the *live* `available_balance` against the value the client committed to in the proof, `processor.rs:914-926` — everything the verifier needs was already committed in that transaction). The gap is discovery: the receipt carries no transaction signature, so `verify` must query `getSignaturesForAddress` on the PDA derived from `E`, which only works if the RPC endpoint retains history back to that slot. Most public RPC nodes prune well short of that.
Cheapest fix: add the sweep transaction's signature to the receipt, and state in crypto.md that `verify` needs an archival RPC or indexer, not a standard pruned one, once the account is closed.

**The recipient side has no described crash-safety model, unlike the heavily specified payer side.** [likely]
Target: "Clients" section, run engine paragraph, which only covers the payer.
v2 gives the payer a `run_secret`, a journal, and an explicit data-loss warning. The recipient's `sweep` command "builds the one transaction above with a fresh D_owner per payment" with no stated derivation scheme, no journal, no statement about what happens if the process dies after the transaction lands but before D_owner's keypair is persisted. If D_owner is a bare random keypair in memory, that crash permanently loses the swept funds — worse than the payer's case, where funds are merely locked, not gone.
Cheapest fix: give the recipient side the same treatment as the payer: derive D_owner deterministically from a stored recipient secret and payment index, and persist before broadcasting, mirroring the payer's `run_secret` pattern.

## MINOR / NOTE

**The griefing vector round 2 found is closed, and source now shows a third leg is closed too.** [verified]
`valid_as_destination` (`mod.rs:161-172`) rejects any confidential credit once `pending_balance_credit_counter` would exceed `maximum_pending_balance_credit_counter`, which v2 pins at 1. The payer's own funding transfer is the only credit ever allowed to land before an `apply`, and `sweep` does `apply` and the draining `Transfer` as CPIs inside one atomic instruction, so no transaction boundary exists for a third party's dust to land: not between `open` and funding (atomic), not between funding and sweep (counter already at max, any attempt fails outright), not between apply and transfer inside sweep (same instruction). This is a property of Token-2022's own state machine plus v2's `max = 1` choice, not just careful sequencing — worth pinning in crypto.md's reviewer checklist alongside the torsion check.

**E is visible before confirmation; a squatter could race the PDA.** [note]
E appears in cleartext in the payer's pending transaction; Solana has no private mempool, so an observer could try to land a competing account at the same seed first, making `open`'s "not initialized" check fail. Cost is a wasted fee and a retry with new E; no funds at risk since funding is atomic with `open`. Generic to Solana, not this design.

**A mint's freeze authority can block sweep.** [note]
`is_frozen()` gates `apply`, `transfer`, and `empty`. Same centralization the candidate already accepts via Paxos-issued mints; worth one line in privacy.md's adversary table.

**Unclaimed-payout reclaim, if ever added, must be worded as a phase-two break of "only the recipient can move funds."** [item 5]
Recommend: a distinct, clearly labeled instruction gated by a long timeout (a year or more), documented as breaking recipient-exclusivity for the unclaimed tail, never merged into `sweep` or `close`. privacy.md's griefing analysis gets a row for it the day it is designed, not after.

## Round-2 resolution table

| Id | Status | Why |
|---|---|---|
| A | resolved | `open` now CPIs `DisableNonConfidentialCredits`; combined with `non_confidential_transfer_allowed()`, a plain SPL credit is rejected after `open` completes. |
| B | resolved | `sweep` folds apply, transfer, empty, close into one atomic instruction; `valid_as_destination`'s counter cap (verified above) means no third-party credit can land in the window R2 found. |
| C | partly | No new construction fix, guidance only. New residual: no documented answer for where SOL to spend from D later comes from without a new link. |
| D1 | resolved | v2 states the torsion check as multiply-by-L explicitly. |
| D2 | resolved | v2 states `open` checks P decompresses. |
| D3 | partly | Still open pending external HKDF review; this round adds an unrelated residual — the receipt's missing transaction signature and archival-RPC dependency. |
| D4 | resolved | PDA seed is now `["stealth", E, k]`. |

## What survived

- Third-party griefing by extra credits, confidential or not, after the recipient's account is funded is now closed by construction, confirmed against Token-2022 source rather than assumed from sequencing alone.
- Payer cannot forge or block a sweep by any signing means, and cannot drain P: both require `p`, which the payer never has, confirmed unchanged.
- The precomputed zero-ciphertext proof plan for `EmptyAccount` is sound: the resulting balance after a full-amount transfer is a deterministic function of values already known to the client before submission, confirmed by reading `process_source_for_transfer`.
- Fresh P, D, and D_owner per payment still block cross-payment correlation for every party except the specific payer of that specific payment.

## privacy.md wording

- "The account's AE-encrypted balance cache is not authenticated on-chain. The payer, who holds the same AE key as the recipient, can write any value there through `apply`. Client code always decrypts the account's balance from the ElGamal ciphertext with its own key; the cache is a hint, never ground truth."
- "Sweeping into a fresh self-owned account does not end the funding problem: that account needs SOL to spend from later, and funding it without linking it back to the payer-funded fee account is unsolved. Treat the swept account as burned for both tokens and fees until a funding path is chosen."
- "Registering a meta-address makes the registering wallet public and permanent, and that wallet is exactly what gets handed to a payer in order to be paid at all. For payroll, where the payer already knows the identity behind the wallet, this costs nothing. For a grants or donations flow, where the payer starts out knowing only a handle, register from a wallet with no other history, or the meta-address itself, not the one-time account, is what deanonymizes the recipient."
- "A disclosure receipt for a closed account is still verifiable, because the proof instructions in the sweep transaction carry the ElGamal pubkey and ciphertexts as public inputs, independent of current account state. It requires an archival RPC or indexer, not a standard pruned one, and the receipt should carry the transaction signature so verification does not depend on address-history retention."
