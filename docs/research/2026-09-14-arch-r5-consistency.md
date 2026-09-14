---
status: draft
last_verified: 2026-09-14
---

# Arch R5 — consistency check on the handover revision

TL;DR: the docs carry the handover shape correctly almost everywhere, with one real self-contradiction (design.md's fallback paragraph says the recipient's sweep needs no program instruction, which the same document's own transaction listing and reclaim instruction refute) and one stale citation (brief.md's 64/64 cap claim still points to research that modeled the pre-handover shape). Numbers (69 bytes, 465 bytes, 86 bytes, and their rents) all check out against the stated rent formula and against arch-r1-mechanics-verified's byte breakdown.

## 1. Leftover previous-shape language

Checked for: a sweep instruction of ours, program-signed sweep, ImmutableOwner on the one-time account, a state byte, a 144-byte announcement, P or mint stored in the announcement, `drain_to` as instruction data, sweep-and-close folded into one instruction, rent refund as a courtesy, five or six program instructions.

None of these survive as present-tense claims in brief.md, design.md, client.md, crypto.md, privacy.md, plan.md, or faq.md. Specific checks:

- No 144-byte announcement, no `drain_to`, no state byte, no ImmutableOwner on the token account anywhere outside decisions.md's historical entries (which are allowed to describe the past).
- design.md line 79 explicitly turns "rent refund as a courtesy" into "a guarantee rather than a courtesy" — this is the rule stated correctly, not a leftover.
- design.md's instruction table lists exactly four program instructions (register_meta, close_meta, open, reclaim), matching its own TL;DR ("four instructions") and brief.md's scope line ("register, close registration, open, reclaim"). No fifth or sixth instruction anywhere current.
- decisions.md's older 2026-09-14 entry ("Architecture rounds...") still describes a `sweep` program instruction and a program-signed CPI chain. That entry is dated history of a since-superseded round, and decisions.md is append-only by the repo's own rule, so this is correct, not a bug.

One real problem found, which is a leftover in substance if not in exact wording:

**design.md line 128** (the Fallback section): "the design falls back to research/2026-09-14-arch-r2-candidate-v1 for transaction packing (proof context accounts, several transactions per side) while keeping the handover: **the recipient's sweep still needs no program instruction**."

This is false against the rest of the document. design.md's own recipient transaction listing (lines 103-112) includes `reclaim` as an instruction, and reclaim is one of the program's four instructions (design.md line 67, table row 3). decisions.md's newest entry says "our program's only later role is reclaim." client.md line 30 says "only the rent reclaim at the end needs our program." plan.md's S3 exit criterion says "one transaction with no program call except reclaim." arch-r5-handover-verified.md's own consequences section is explicit that a program call to close the announcement PDA is still required because only the owning program can debit a program-owned PDA. The "no program instruction" sentence in design.md's Fallback paragraph appears to be a holdover from an earlier point in the redesign (before reclaim was reintroduced to close the announcement) and was never updated when reclaim was added back. See Must fix.

## 2. Numbers

Rent formula `(128 + len) × 3480 × 2`, checked against every stated size:

- MetaAddress, 86 bytes: (128+86)×3480×2 = 1,489,440. Matches design.md exactly.
- Announcement, 69 bytes: (128+69)×3480×2 = 1,371,120. Matches design.md exactly. Field list (discriminator 1, version 1, E 32, payer 32, view_tag 1, k 1, bump 1) sums to 69.
- Token account, 465 bytes: (128+465)×3480×2 = 4,127,280. Matches design.md exactly. Cross-checked against arch-r1-mechanics-verified's sourced breakdown for the pre-handover account (base 165 + type byte 1 + ImmutableOwner TLV 4 + ConfidentialTransferAccount TLV 299 = 469 bytes, 4,155,120 lamports): removing exactly the 4-byte ImmutableOwner TLV gives 465 bytes and 4,127,280 lamports, which is precisely what design.md claims and precisely what dropping ImmutableOwner should produce. The [likely] tag on this one row (design.md line 54) is appropriately conservative but the arithmetic is sound.
- `open`'s instruction data (E 32, P 32, k 1, view_tag 1, decryptable_zero_balance 36, lamports_for_P 8, bump 1, +1 discriminant = 112 bytes) matches arch-r3-verified's sourced 112-byte estimate for `open` even though `open` gained two SetAuthority CPIs under the handover revision — both new CPIs reuse data already present in the instruction (P is already passed; the announcement PDA is derived on-chain), so no growth in `open`'s own byte count. This is a case where a number that looks like it should have moved didn't, correctly.
- CU figures (register_meta ≈2,336 CU) match crypto.md's breakdown (159 + 2,177 = 2,336) exactly.

Recipient's instruction list against the byte budget: counting design.md's own listing (lines 103-112) as top-level instructions gives SetComputeUnitLimit, VerifyPubkeyValidity, CreateAccountWithSeed, InitializeAccount3, ConfigureAccount, DisableNonConfidentialCredits, VerifyCiphertextCommitmentEquality, VerifyBatchedGroupedCiphertext3HandlesValidity, VerifyBatchedRangeProofU128, VerifyZeroCiphertext, ApplyPendingBalance, Transfer, EmptyAccount, reclaim, System transfer — **15 top-level instructions**, well under 64. Unique accounts: P, D_owner, D, the one-time (source) token account, the announcement PDA, mint, the stored payer, instructions sysvar, System, Token-2022, ComputeBudget, ZK ElGamal proof program — **about 12**, well under 64. Splitting the old single CPI'd `sweep` instruction into four top-level Token-2022 calls plus `reclaim` adds per-instruction envelope overhead (program-id index, account count, data-length prefix) roughly four to five times over, on the order of tens of bytes, not hundreds; arch-r3-verified's old recipient-transaction total (2,968 of 4,096 bytes, 1,128 bytes of margin) had enough headroom to absorb this comfortably. So the claim plausibly holds, but no research file has recomputed the exact new total — S5 in plan.md ("measured bytes and CU... once encoders exist") is still open, which is the correct place for this, not a doc bug.

Given that, **brief.md line 90**'s claim — "A stealth payment fits one 4096-byte transaction per side, under the 64-account and 64-instruction caps | likely | arch-r3-verified" — cites research that verified the pre-handover shape (single CPI'd `sweep`, ImmutableOwner present, D built inside `sweep`). The conclusion is very likely still true (see arithmetic above) but the citation no longer matches the architecture it's attached to. See Must fix.

## 3. Cross-document contradictions

Checked: who owns the token account when; who can close it; where rent goes; what the payer can and cannot do after open; never-swept accounts; the issuer path; the D default and whole-balance option.

All consistent across brief, design, client, crypto, privacy, plan, decisions:

- Ownership: PDA during open, P after (design.md, crypto.md line 56, decisions.md all agree).
- Close authority: pinned to the announcement PDA (design.md, r5-handover-verified, client.md's reclaim description all agree); rent always returns to the stored payer.
- Payer's post-open limits: brief.md section 6, crypto.md's concerns table, and privacy.md's residual-leaks section all state the same thing — the payer holds that one account's decryption keys, can't move funds, can't block the sweep, and a partial spend (not a full sweep) shows it the split.
- Never-swept accounts: design.md, decisions.md's consequence line, and plan.md's Q10 all agree the payer can never reclaim them, only P can empty them.
- Issuer path: client.md's mint paragraph and plan.md's Q9 agree on the same two-step flow (deferred-handover open, policy-program approver, not built in v1).
- D default / whole-balance option: design.md line 114, client.md's `--spend-to` line, and decisions.md's rejected-alternatives note all agree D is the default and a whole-balance spend to a counterparty is the documented alternative.

The one contradiction found is internal to design.md, already covered in section 1: the Fallback paragraph's "no program instruction" claim against the same document's own instruction listing, reclaim row, and confidence table.

## 4. faq.md and README.md against the handover design

No staleness found. faq.md's "Doesn't the payer know the recipient's key?" answer ("Accounts are one-time, swept into a fresh self-owned account, then closed") is compatible with the handover shape and doesn't claim program custody. The top-level README.md's status line ("one Pinocchio program and one Rust CLI are planned; a stealth payment is one transaction per side") is generic enough to still hold. Neither file mentions a sweep instruction, ImmutableOwner, or instruction counts that would need updating.

## 5. Rules

Front-matter (`status` + `last_verified`) and a TL;DR are present in every file read. No em dashes, no AI-tell words (checked for comprehensive/robust/seamless/leverage/delve/cutting-edge/game-changing/revolutionize/synergy and found none), no emoji. Terminology: "payer" and "recipient" used consistently in every current-state doc; the three uses of "sender" are all inside decisions.md's dated 2026-09-11 historical entries, which decisions.md's own append-only rule permits.

Word counts (`wc -w`, includes table markup so slightly over true prose count):

| File | Words | Against ~1,500 budget |
|---|---|---|
| decisions.md | 1,587 | over, but append-only logs are expected to grow |
| design.md | 1,722 | over, and this is after client.md was already split out of it on 2026-09-14 |
| brief.md | 1,552 | just over |
| crypto.md | 1,160 | under |
| plan.md | 1,118 | under |
| client.md | 934 | under |
| privacy.md | 855 | under |
| faq.md | 699 | under |
| docs/README.md | 428 | under |

design.md is the one worth a second look: docs/README.md's own rule 3 says to split a doc that passes ~1,500 words, and design.md already went through one split today and is still 15% over. Worth another trim or split before the next round, not urgent.

## 6. First-time-reader clarity in design.md

The recipient's transaction paragraph (design.md line 101) says: "Recipient, one transaction, signed by P and D_owner, about 3,000 bytes [likely], about 231k CU of proofs plus Token-2022 work; **no instruction of ours except the last**:" followed by a list ending in:

```
reclaim                 rent of both accounts to the payer
System transfer of P's remainder to D_owner
```

`reclaim` — the one instruction that is "ours" — is second-to-last, not last; the actual last line is a plain System transfer. A first-time reader following the list top to bottom would look for our instruction at the end and find someone else's. This should read "no instruction of ours except reclaim" or the list should be reordered so reclaim is genuinely last. Same underlying issue as the Fallback contradiction in section 1: the sentence hasn't been checked against the list it introduces.

The `open` row of the instruction table (design.md line 66) packs five Token-2022 CPIs and two System calls into one semicolon-and-comma-separated cell. It's accurate (checked against arch-r5-handover-verified's ordering requirement — SetAuthority CloseAccount must precede SetAuthority AccountOwner, since the PDA can only sign while it's still the owner, and the row has them in that order) but dense enough that a reader has to parse carefully to see the two SetAuthority calls are new. Not wrong, just worth a line break or a short lead-in sentence if this table is touched again.

## Must fix

1. design.md line 128 (Fallback section): remove or correct "the recipient's sweep still needs no program instruction." It contradicts design.md's own recipient transaction listing (which includes `reclaim`), the reclaim instruction row in the same document, decisions.md's newest entry, client.md's reclaim description, and plan.md's S3 exit criterion, all of which agree a program call for `reclaim` is still required. Suggested fix: "the recipient's sweep still needs no program instruction to move funds; reclaim remains the one program call, to close the announcement."
2. design.md line 101: fix "no instruction of ours except the last" — `reclaim` is second-to-last in the list that follows, not last. Either reword to name reclaim directly or move `reclaim` to the end of the list (after the System transfer of P's remainder), whichever matches the actual required ordering.
3. brief.md line 90: the citation for "under the 64-account and 64-instruction caps" points to arch-r3-verified, which modeled the pre-handover shape (single CPI'd sweep, no separate reclaim, ImmutableOwner present). The conclusion is very likely still true (see section 2's arithmetic: 15 instructions, ~12 accounts, comfortable margin under the old 2,968/4,096-byte total), but no research file has re-verified it for the handover shape specifically. Either recompute and cite the new totals (folds naturally into S5, already open in plan.md) or soften the brief.md claim's sourcing note to flag it as carried over pending S5.

## Nice to fix

- design.md's "Why these shapes" bullets (lines 75-83), especially line 78 ("Before, a bug in a program-signed sweep could brick funds... After, the recipient depends on Token-2022 and nothing else..."), retell the before/after history that decisions.md's newest entry already holds almost verbatim. Per the repo's own rule ("docs describe the present; decisions.md holds the why and the history"), this could be trimmed to a present-tense statement of the guarantee with a pointer to decisions.md for the history.
- design.md is 1,722 words, over budget again after today's split into client.md. Consider trimming the Instructions section's densest cells or moving the "Why these shapes" bullets to decisions.md per the point above.
- The word "sweep" is overloaded: it names a CLI command (client.md), a general recipient action (plan.md, privacy.md), and, in decisions.md's historical entries, a now-removed program instruction. Current docs use it consistently enough that a careful reader untangles it, but a glossary line in docs/README.md or client.md ("sweep: the CLI command; it builds one transaction of plain Token-2022 instructions plus reclaim, no program instruction of its own moves funds") would remove the ambiguity a first-time reader hits in design.md's Fallback line.
- design.md's `open` instruction-table cell (line 66) is dense; a short lead sentence or line break separating "sets up the account" from "hands it to P" would help a first-time reader see the two new SetAuthority calls are the handover mechanism, not incidental cleanup.
