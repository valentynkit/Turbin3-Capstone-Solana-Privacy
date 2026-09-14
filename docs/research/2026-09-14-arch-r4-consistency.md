---
status: draft
last_verified: 2026-09-14
---

# Arch R4 — consistency and accuracy review of the rewritten docs (2026-09-14)

TL;DR: the rewrite holds together well; most numbers trace to r2/r3-verified and the ten fresh-eyes gaps are mostly closed. Two real problems: the issuer-approval-mint fallback in client.md calls for a "payer-gated apply" step that does not exist in the four-instruction program, and design.md's fallback deadline for spike S1 ("end of week 1") contradicts plan.md's own spike table ("day 1").

## 1. Numbers checked against research

| Claim | Doc | Value | Research | Match |
|---|---|---|---|---|
| register_meta torsion check CU | design.md, crypto.md | 2,336 CU (159 + 2,177) | arch-r2-crypto-privacy-skeptic.md:34,37 (`curve25519_edwards_validate_point_cost` 159, `curve25519_edwards_multiply_cost` 2,177) | match, but source is r2-crypto-privacy-skeptic, not r2/r3-verified as the doc's "Source-verified in research/2026-09-14-arch-r1 to r3" banner implies for this specific number |
| Payer tx proof CU subtotal | design.md | ~226k CU | arch-r3-verified.md: 225,400 | match |
| Recipient tx proof CU subtotal | design.md | ~231k CU | arch-r3-verified.md: 231,400 | match |
| Range proof CU | design.md, plan.md | 200,000 CU | arch-r3-verified.md sourced constant | match |
| Payer tx bytes | design.md | ~2,700 B of 4,096 | arch-r3-verified.md: 2,709 B computed | match, [likely] mark appropriate |
| Recipient tx bytes | design.md | ~3,000 B | arch-r3-verified.md: 2,968 B computed | match |
| Token account size/rent | design.md, client.md | 469 B, 4,155,120 lamports | arch-r1-mechanics-verified.md:61 (469B -> 4,155,120/0.00415512) | match (source outside the four assigned files but present in research/) |
| StealthAccount size | design.md, client.md | 144 bytes | offsets in design.md sum to 144; client.md `dataSize = 144` scan filter | internally consistent |
| MetaAddress size | design.md | 86 bytes | offsets sum to 86 | internally consistent |
| ZK ElGamal program live since | design.md | "epoch 982" | arch-r1-mechanics-verified.md:22 (epoch 982, ~early June 2026) | match, source outside the four assigned files |
| v1 tx feature activation | decisions.md | "mainnet expected 2026-09-15" | arch-r2-verified.md R2-2 (mainnet `activated_at: None` as of epoch 1034; public reporting ~2026-09-15) | match, correctly hedged as expected not confirmed |
| Proof gen timing | client.md | range 16 ms, others <0.5 ms, 50-recipient set 0.2s | arch-r3-verified.md Spike 11: range 16.316ms, others 0.035-0.405ms, N=50 total 202.5ms | match |
| Priority allowance | design.md | "about 450k CU at the run's price" | arch-r3-candidate-v2.md:115 carries the same figure forward | traces to the candidate doc, not reconciled against the ~226k/231k totals shown a few lines earlier in the same doc (see section 7) |
| Payer cost to fund P | design.md | "0.0045 to 0.006 SOL" | arch-r3-candidate-v2.md:115, same range | match |
| Payer cost, brief.md phrasing | brief.md | "fronts about 0.006 SOL, refunded... sends the recipient about 0.005 SOL" | design.md's two separate figures: 0.006 SOL locked+refunded (StealthAccount+token account), 0.0045-0.006 SOL sent to fund P | brief collapses a range into a single point (0.005) and doesn't carry design's "[open]" qualifier into its own number, though the paragraph as a whole is tagged [likely, unmeasured] |
| Token-2022 instruction encoder count | plan.md | "six Token-2022 instructions" | design.md's own instruction table names eight distinct Token-2022 instruction types (InitializeImmutableOwner, InitializeAccount3, ConfigureAccount, DisableNonConfidentialCredits, ApplyPendingBalance, Transfer, EmptyAccount, CloseAccount) | mismatch, see Must fix |
| PYUSD/USDG auto_approve_new_accounts | brief.md, client.md, decisions.md | false, issuer must ApproveAccount | arch-r2-verified.md R2-6 (mainnet-beta getAccountInfo, 2026-09-14) | match across all three docs |

## 2. Contradictions between docs

| Topic | Doc A | Doc B | Verdict |
|---|---|---|---|
| Spike S1 deadline | design.md Fallback: "by the end of week 1 (spike S1)" | plan.md spike table: S1 exit "day 1"; plan.md Risks: "S1 on day 1" | contradiction, see Must fix |
| Token-2022 encoder count | plan.md: "six Token-2022 instructions" | design.md instruction table: eight distinct instruction types used | contradiction, see Must fix |
| Issuer-approval "apply" step | client.md Mints: "a payer-gated apply plus the transfer" | design.md/decisions.md: program has exactly four instructions (register_meta, close_meta, open, sweep), no standalone apply | client.md's approval-mint fallback names a step that isn't in the instruction set, see Must fix |
| Instruction count vs table | design.md TL;DR "four instructions"; decisions.md "four instructions (register_meta, close_meta, open, sweep)" | design.md's own table has exactly four rows | consistent (fresh-eyes gap 9, now resolved) |
| Rent recipient | design.md: StealthAccount field "payer, gets every rent refund"; Money: "Locked... refunded at sweep... about 0.006 SOL" to payer | design.md Money: "D rent... ends with the recipient" | consistent, two distinct pots correctly kept separate |
| Fallback transaction count | design.md Fallback: "five to seven transactions per side" | plan.md Risks: "five to seven transactions per side" | consistent |
| Mint policy / issuer path | brief.md, client.md, decisions.md | all three: auto-approve mints work unmodified, PYUSD/USDG need an approver, documented not built | consistent wording across all three |
| CLI subcommand list | brief.md Scope: "register, pay, scan, sweep, disclose, verify, bench" | design.md Components: "register, pay... scan, sweep... disclose, verify; bench" | consistent, 7 subcommands both places |
| "sender" vs "payer/recipient" terminology | decisions.md uses "sender" 3x (lines 58, 59, 65) | CLAUDE.md: "payer" and "recipient" in current-state docs | decisions.md is history, not current-state, per docs/README.md's "Docs never carry history" split — arguably exempt, flagged as borderline in section 6 |

## 3. [verified] marks

Spot-checked marks in design.md, crypto.md, client.md against the four assigned research files plus the wider research/ folder where a mark pointed outside them:

- Every [verified] mark checked traces to an actual research finding (arch-r1/r2/r3-verified, or a skeptic file that itself cites runtime source line numbers). None found asserting something the cited or inferable research contradicts.
- Marks whose source sits outside the four assigned files (epoch 982, the 2,336 CU torsion cost, the 469B/4,155,120 rent figure) are still real citations within research/, just not in r2-verified/r3-verified specifically — acceptable under design.md's blanket "Source-verified in research/2026-09-14-arch-r1 to r3" banner, but a reader following only the four assigned files would not find them.
- design.md's "priority allowance for about 450k CU" carries no [verified]/[likely]/[open] mark at all, unlike the two CU subtotals a few lines above it that do carry marks. Given r3-verified's own CU tables leave Token-2022's own processing overhead as "[open]" and total 250-320k likely, a 450k allowance is plausible as a safety margin but is presented as a bare fact. Should carry a mark.
- crypto.md's "[open: reviewer to confirm using the signing secret as HKDF input is acceptable...]" and design.md's Confidence table are the right pattern; no claim found stating something as settled that the confidence table itself marks medium or lower.

## 4. Stale content

Checked brief.md, design.md, client.md, crypto.md, privacy.md, plan.md, faq.md, README.md for: Anchor, two programs, fund instruction, context-state accounts as default, lookup tables, relayer, "fee dust", ~10 transactions, 0.013 SOL, TypeScript, Codama, teammates/team split.

None found. Every occurrence of these terms in current-state docs is a negation ("no relayer", "no lookup tables", "no context-state accounts") describing what the design does not do, correctly distinguishing the current shape from the rejected one. The stale numbers (~10 transactions, 0.013 SOL) and stale stack (Anchor, TypeScript) appear only in decisions.md's "Rejected:" lines, which is exactly where CLAUDE.md's history/present split says they belong. faq.md and README.md are both clean.

## 5. Fresh-eyes blocking gaps (arch-r3-fresh-eyes.md)

| # | Gap | Answered where | Status |
|---|---|---|---|
| 1 | Instructions sysvar missing from `open`/`sweep` account lists | design.md Accounts for open / Accounts for sweep both list "instructions sysvar" | resolved |
| 2 | Mint account missing from `sweep` | design.md Accounts for sweep lists "mint" | resolved |
| 3 | `apply`'s instruction data unspecified | design.md folded apply into sweep; sweep's Data lists "new decryptable balance after apply 36" etc | resolved (by removing the standalone instruction, not by specifying it) |
| 4 | Batch-pregenerated Transfer proofs invalid once source balance changes | client.md Batch engine: "sends are sequential per payer source account... the three transfer proofs and the zero-ciphertext proof are generated just before each send" | resolved, matches fresh-eyes' own recommended fix |
| 5 | Recipient sweep can't fund D on approval-required mints | client.md Mints: "the flow splits on both sides... create D, issuer approves, then sweep... needs an approver... documented, not built" | partially resolved — documented at the narrative level, but the "payer-gated apply" step it names isn't a defined program instruction (see Must fix) |
| 6 | D_owner derivation and recipient resume/journal story unspecified | crypto.md: `d = HKDF-SHA512(..., ikm = b_spend, info = "d-owner" || E || k)`, deterministic | derivation resolved; no explicit resume/journal statement for the recipient side (payer gets one, recipient doesn't), though determinism implicitly avoids "funds gone on crash" |
| 7 | MetaAddress byte layout referenced, not given | design.md Accounts: full offset table for MetaAddress (86 bytes) | resolved |
| 8 | Resume can't distinguish "A landed, B pending" on approval-required mints | not addressed anywhere found | still open |
| 9 | TL;DR instruction count (six) vs table (five) | design.md TL;DR "four instructions", table has four rows | resolved, numbers now agree |
| 10 | brief.md stale scope/cost vs design | brief.md now says "one Pinocchio program... register, close registration, open, sweep"; 0.013 SOL and ~10 tx removed, documented in decisions.md | resolved |

## 6. CLAUDE.md rules

| Rule | Result |
|---|---|
| Front-matter status + last_verified | present on brief, design, client, crypto, privacy, plan, decisions, faq, landscape; **missing on docs/README.md** |
| Two-line TL;DR | present on all nine docs checked, docs/README.md included |
| Under ~1,500 words | brief 1,496; design 1,406; client 596; crypto 1,138; privacy 831; plan 1,001; decisions 1,210; faq 699; README 417 — all within budget, brief is at the edge |
| No em dashes | none found in any doc |
| No AI-tell words | none found (checked comprehensive, robust, seamless, leverage, utilize, cutting-edge, holistic, synergy, paradigm, unlock, elevate, game-changing, revolutionize, innovative, boast) |
| No emoji bullets | none found |
| "payer"/"recipient" terminology | held everywhere in current-state docs; decisions.md uses "sender" 3x, arguably exempt as history rather than current state, worth a decision either way |

## 7. Unclear on first read

- design.md instruction table, sweep row: "Signers: P, fee payer". A first-time reader can't tell if this means two separate signers or "P, acting as fee payer" (one signer, one role) — crypto.md later clarifies P pays the fee, but the table alone reads as ambiguous, especially since the worked example a few lines down says "signed by P and D_owner" (different pairing).
- design.md Money: "a priority allowance for about 450k CU at the run's price" sits right after two CU subtotals of ~226k and ~231k with no line connecting the 450k figure to either total or explaining the margin.
- design.md "why these shapes" bullet: "The `expected_pending_balance_credit_counter` field of ApplyPendingBalance is stored, never checked; the client treats it as bookkeeping." States a fact without saying why a reader should care (it means the field can't be relied on as an on-chain guard) — the "so what" is implicit.
- client.md Mints: "the flow splits on both sides (open, issuer approves, then a payer-gated apply plus the transfer; create D, issuer approves, then sweep)". Dense parenthetical covering two different multi-step flows in one sentence; a reader has to reconstruct which steps belong to the payer's side and which to the recipient's, and "apply" here reads as a program instruction that doesn't otherwise exist in the doc set.
- design.md "One stealth payment" pseudocode: "open offset to the pubkey proof" — terse enough that a reader unfamiliar with inline proof offsets won't parse this as "open's instruction data contains an offset pointing at instruction #2."

## Must fix before decided

1. client.md's issuer-approval-mint fallback names a "payer-gated apply" step that has no corresponding program instruction — the shipped design has exactly four instructions (register_meta, close_meta, open, sweep) with apply folded into sweep. Either define the fifth instruction the approval-mint path needs, or rewrite the client.md sentence to describe what actually gets called.
2. design.md's Fallback section ("by the end of week 1") contradicts plan.md's spike table and Risks section (both say spike S1 exits "day 1"). Pick one deadline and match it in both docs.
3. plan.md's "encoders for the six Token-2022 instructions" doesn't match design.md's own instruction table, which names eight distinct Token-2022 instruction types. Recount and fix the number, or clarify that "six" means something narrower (e.g. only the inline-proof variants needing hand-rolled encoders) and say so.
4. docs/README.md has no front-matter `status`/`last_verified` block, unlike every other file in docs/. Add one or note the exemption in docs/README.md's own Rules section.
5. Fresh-eyes gap 8 (resume can't distinguish "transaction A landed, B pending" from "both landed" on approval-required mints) is still unanswered anywhere in client.md or plan.md.

## Nice to fix

- brief.md's Cost bullet collapses two different design.md figures (0.006 SOL refunded-at-sweep lock; 0.0045-0.006 SOL sent to fund the recipient's account) into single-point numbers (0.006, 0.005); consider carrying the range instead of a point estimate for the second figure.
- design.md's "450k CU" priority allowance has no confidence mark and isn't reconciled against the ~226k/231k CU subtotals shown just above it in the same section.
- State explicitly (crypto.md or client.md) that D_owner's deterministic derivation from `b_spend`, `E`, `k` gives the recipient side the same crash-safety property the payer's journal gives the payer side — fresh-eyes gap 6 is resolved in substance but not in words.
- design.md's sweep row "Signers: P, fee payer" could read "Signers: P (fee payer)" to remove the two-signers-or-one ambiguity against the later "signed by P and D_owner" example.
- Decide whether decisions.md's three "sender" uses need fixing to payer/recipient, or whether docs/README.md's rules should say explicitly that decisions.md, as history, is exempt from the terminology rule.
