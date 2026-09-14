---
status: draft
last_verified: 2026-09-14
---

# Arch R2 — design skeptic review of candidate v1

TL;DR: the candidate is unusually well cross-checked internally — I re-derived its rent and transaction-size arithmetic and found no errors, and it correctly closes round 1's critical griefing finding (C1) by making open+configure+fund atomic. The real gaps are two adopted round-1 ideas that dropped their own stated costs, one likely premature build (on-chain registry), and a process gap: round 1's four "brief refinements" sections never got consolidated, so brief.md and plan.md are already stale against this candidate.

How to read: **[verified]** checked by reading the candidate and its cited sources; **[likely]** strong inference; **[open]** the candidate itself already flags this, noted only where the framing needs a correction.

## Major

**1. Deterministic run seed (idea 12, adopted) drops its own stated cost: the run_secret becomes a real secret with no backup story.** [verified against candidate "Batch resume" row and creative §12]
Candidate: "deterministic ephemeral keys `e_i = HKDF(run_secret, run_id || i)`, append-only journal, check on-chain state before every step." Creative's own writeup of this idea says plainly: "Losing the run secret loses the ability to reap rent or answer an audit for that run. It becomes a real secret to back up." The candidate's Clients section lists a `keyring` module with no design detail, and nowhere states how `run_secret` is persisted, whether it survives past `Reclaim`, or whether the demo's audit-decrypt criterion (brief §8) depends on a run_secret that is only ever held in memory. If the process that ran the demo payout exits before someone writes `run_secret` down, the "auditor decrypting one payment" success criterion for that run becomes unreproducible. Fix: one sentence in the Clients section on where `run_secret` lives (encrypted file, `keyring`-crate-backed, whatever) and a note that losing it before Reclaim is a silent data-loss event, not just an inconvenience.

**2. On-chain registry is kept without addressing round 1's own free alternative.** [likely]
The "round 1 settled" table's only registry entry is "Program count: one program; registry is a PDA family inside it" (obvious §1, creative 7), which settles *how many programs* but not *whether the registry needs to be on-chain at all*. Creative idea 7 explicitly offered a second option — "a 64-byte bech32m string exchanged out of band like an IBAN, with no chain state at all" — rated confidence high, verify cost zero, the cheapest win in round 1's whole ranking table. The candidate builds `register_meta`/`close_meta`, an 86-byte account, and a torsion-free curve check on every registration without ever weighing that against the free alternative for a capstone demo. This may be the right call if the product's actual pitch is "pay any wallet without an out-of-band exchange," but that argument isn't made anywhere in the candidate — it's inherited by omission. Fix: one line stating why on-chain lookup earns its keep for the demo, or cut it and use a local address book per idea 7's own exit criterion ("the demo runs without a registry; add one only if the demo actually needs it").

## Minor

**3. Test-pyramid fallback from round 1 didn't carry forward.** [verified against candidate "Testing" section and obvious §8]
Obvious's own test section ends with "If the schedule slips, Mollusk goes first, since LiteSVM also reports CUs." The candidate keeps all four harnesses (Mollusk, LiteSVM, Surfpool, devnet) as flat, undifferentiated scope with no priority order, in a project where R2-5 (does the ZK ElGamal proof program even run in Mollusk/LiteSVM) is still open and week 1 is already loaded with hand-encoding six CPI instruction builders (ops finding 5, not repeated here but its schedule pressure is real). Fix: restate the cut order explicitly, since the candidate otherwise reads as if all four are equally load-bearing.

**4. Chaos harness is scoped but not budgeted.** [verified against candidate "Testing" section and creative §13]
Creative's own accounting: "About 100 lines... a day of work that produces no demo footage." The candidate cites it ("kill the batch client at every step index of a 20-recipient run and assert convergence (creative 13)") as flat scope with no week attached and no schedule section exists in this document to attach it to. Not wrong to keep — idea 13 ranks high-value — but for a six-week solo-programs-engineer plan, a day with no demo output is worth a line saying when it happens relative to the two devnet-dependent open questions (R2-1, R2-5) it partly tests.

**5. Permissionless `close` can race the recipient's own bundled close-and-drain transaction.** [likely, new]
Recipient step 4 is "`close` (zero-ciphertext proof inline) plus System transfer of P's remaining lamports to D_owner," both instructions in one transaction. `close`'s guard is only `state == Swept` — anyone can call it once a sweep lands, and nothing about doing so benefits an outside caller (rent goes to `rent_refund`, the payer, not the caller), so this isn't an incentivized attack, but an unrelated crank or another client instance racing the recipient's own client will make that instruction fail with a wrong-state error, and since Solana transactions are atomic, the bundled P-lamports drain fails with it. No design doc addresses "close already happened, retry just the drain." Fix: split the recipient's step-4 transaction build so the drain doesn't depend on `close` succeeding in the same transaction, or have the client re-check state before building it.

**6. R2's ten open questions have no owner or recommendation column.** [verified against CLAUDE.md:12 and the candidate's "Contested or open" table]
The repo's own convention: "Open questions stay open with a recommendation and an owner in `docs/plan.md`; do not silently resolve them." The candidate's table has Id/Question/Why-it-matters only. None of R2-1 through R2-10 have made it into `docs/plan.md`, which still shows the pre-round-1 phase table and open-questions list (Q1-Q8), several of which round 1 already answered (Q4 "refuse by default" is now settled; the relayer Q-adjacent risk-table line "drop status page and relayer first" refers to a component round 1 removed entirely). This is process debt, not an architecture flaw, but it means the round-1/round-2 work is currently only legible by reading five research files, not the docs a new contributor is told to start from.

## Note

**7. StealthAccount layout is approximate where the program needs it exact.** [verified against candidate account section]
"about 150 bytes... reserved" versus round 1's own precise 190-byte, offset-by-offset table for the same account family (before the `token_account` field was dropped). Not a defect — the candidate is a synthesis document, not the spec — but worth pinning to exact offsets before anyone starts the zero-copy cast, since `assert_no_padding!` and the fixed-offset scan filter (`dataSize`, `memcmp` on `view_tag`) both depend on the final number, not "about."

**8. TypeScript's role is left open in a project the owner constraints name as staffed by TypeScript teammates.** [likely]
R2-9 defers TS entirely ("none" is a live option). Brief §9 already gives teammates non-SDK work (devnet deployment, status page, demo), so this may resolve fine on its own, but the candidate doesn't say that explicitly — it reads as an unresolved team-fit question rather than a covered one. One sentence connecting R2-9's "none" branch to brief §9's existing non-coding scope would close this.

## What survived

The rent and dust arithmetic checks out by hand: sweep-context rent (0.00201144 + 0.00357048 + 0.002958 = 0.00853992, candidate's "0.00854") and the destination token account rent (0.00415512, candidate's "0.00416") both reproduce exactly from the cited per-proof rent table. The atomic open+configure+fund transaction genuinely closes round 1's C1 (third-party credit-counter griefing) — no residual window exists once those three steps share one transaction, which the candidate states but is worth confirming holds on inspection. Proof placement, discriminants, and CU figures trace cleanly to the cited mechanics-verified file with no transcription errors found.

## What to cut for the capstone, in order

1. The on-chain registry, unless the candidate states why "pay any wallet without prior exchange" is load-bearing for the demo (finding 2). Cheapest possible cut: two instructions, one account family, one curve check.
2. The chaos harness, if week 1's spikes (R2-1 through R2-5) eat into week 2 — it produces no demo footage and depends on the batch client already existing (finding 4).
3. Mollusk as a separate CI job, folded into "LiteSVM already reports CUs" if schedule slips (finding 3) — this was obvious's own stated fallback, not a new cut.
4. Devnet as anything but release smoke — already scoped that way; don't let it creep into the PR-gate loop given the airdrop-cap risk ops-skeptic already flagged.

Nothing here argues the shape is wrong. The program's three-and-a-bit instructions, the no-fund-instruction decision, and the credit-counter-as-invariant design are all sound and already reasoned past round 1's objections.
