---
status: draft
last_verified: 2026-09-14
---

# Arch R4 — final adversarial pass before "decided" (2026-09-14)

TL;DR: the architecture itself holds; three rounds of review already closed the griefing, atomicity and approval-mint problems this pass went looking for. What is left is one internal contradiction in a "verified" claim and a schedule that the plan's own numbers do not support, plus one spike gap on a cryptographic guard nobody has exercised.

Scope: re-read brief.md, design.md, client.md, crypto.md, privacy.md, plan.md, decisions.md against the six hunt targets in the prompt, cross-checked against arch-r3-crypto-privacy-skeptic, arch-r3-fresh-eyes and arch-r3-verified so as not to re-file what those already closed (approval-mint sweep gap, missing sysvar/mint accounts, D_owner derivation, receipt signature, batch-proof-vs-concurrency conflict — all now fixed in the current docs and not repeated here).

## MAJOR

**brief.md's "No third party can block a funded payment" claim is contradicted by its own cited source.**
Target: `brief.md` claims table, row "No third party can block a funded payment: one credit allowed, public credits disabled, sweep and close atomic | verified | arch-r3-crypto-privacy-skeptic".
That exact research file's own MINOR/NOTE section says: "A mint's freeze authority can block sweep... `is_frozen()` gates `apply`, `transfer`, and `empty`." A mint authority is a third party, and freezing is blocking. privacy.md's adversary table already carries this caveat ("Mint authority... freeze authority... a property of the mint, not of the rail"), but the claims table — the document CLAUDE.md names as canonical for scope, meant to be read on its own — states the stronger, false version without qualification.
Fix: reword the row to "No third party other than the mint's freeze authority can block a funded payment" or split it into two rows, one for griefing (true, unconditional) and one for freeze (true, mint-dependent).

**The plan's own effort estimate is 1.5-2x the six-week budget, and the cut order does not close that gap.**
Target: `plan.md`, "Estimate from a cold read... 45 to 60 engineer-days" versus the six-week (≈30 working-day, solo) Phases table, and the Cut order section immediately below it.
45-60 days against 30 available is a 50-100% overrun before any cut. The cut order removes a status page, moves Mollusk to a separate job, ships the receipt verifier format-only, drops 50 recipients to 10, and trims the chaos harness to three kill points — plausibly 10-15 days of savings combined, which closes the gap only at the low end of the estimate and not at all at the high end. Nothing in the cut order touches the two largest cost drivers arch-r3-fresh-eyes named (program work: no_std zero-copy, curve syscalls, Token-2022 CPI ordering; client work: HKDF tree, journal, scanner). A plan about to be marked decided should either extend the window, name a scope cut large enough to matter (e.g., ship plain-mode only for the demo and push stealth to a follow-on week), or state explicitly why the 45-60 figure no longer applies.
Fix: add the arithmetic to plan.md and pick one of the three responses above, on the record.

**No spike covers the curve25519 syscall FFI that `register_meta`'s torsion check depends on.**
Target: `plan.md` spikes S1-S6; `design.md` instruction table, `register_meta` guard: "spend_pub decompresses and `L·spend_pub = identity` via curve25519 syscalls, about 2,336 CU [verified]."
arch-r3-fresh-eyes already flagged this as new ground: "Pinocchio's own reference material has no mention of curve25519 syscalls." That finding was never closed — S2 spikes `ConfigureAccount` under CPI, which never touches this code path; no other spike names `register_meta` or the curve syscalls at all. The CU cost is sourced from Agave's cost model, which says nothing about whether the raw FFI declarations needed to call `sol_curve_validate_point`/`sol_curve_group_op` from a `no_std` Pinocchio program compile and behave as expected. This is the only cryptographic guard against small-subgroup spend keys, and it sits in week 1's first program instruction, ahead of S1 and S2's own results.
Fix: fold a one-hour FFI smoke test into S2, or add it as S0 on day 1, before register_meta is written for real.

## Verdict

The design is sound where the last three rounds already stress-tested it: atomicity closes the griefing windows, the approval-mint gap is honestly scoped out rather than hidden, and D_owner's deterministic derivation gives the recipient side the same crash safety the payer side already had. What is left before "decided" is bookkeeping, not architecture: fix the one claims-table overstatement, put a number on the schedule gap and choose a response, and give the curve25519 syscall path the same day-one attention S1 and S2 already get. None of the three blocks the shape of the design; all three are cheap to close before the label changes.
