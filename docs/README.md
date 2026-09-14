---
status: draft
last_verified: 2026-09-14
---

# Docs index

TL;DR: one flat folder. `brief.md` says what and why, `design.md` says how, `research/` holds evidence, `decisions.md` holds history. Everything else derives from those.

## Structure

| File | Purpose | Canonical for |
|---|---|---|
| `brief.md` | What we build and why, one page, ends with a claims table | scope |
| `landscape.md` | Competitors, positioning, compliance posture, target users | market |
| `design.md` | Components, keys, accounts, instructions, one payment, money, mints | how |
| `client.md` | Batch engine, recipient flow, audit receipt, testing, stack; split from design.md 2026-09-14 | client |
| `crypto.md` | Key derivation, curve checks, reviewer checklist | cryptography |
| `privacy.md` | Threat model, what an observer sees, residual leaks | privacy guarantees |
| `plan.md` | Phases, milestones, risks, open questions with owners | what happens next |
| `decisions.md` | Append-only dated log: what, why, alternatives rejected | history |
| `faq.md` | Questions people ask, answers, short message variants | comms |
| `loi/` | Cohort deliverable: proposal and red-team log | grading |
| `research/` | Dated evidence, immutable, `YYYY-MM-DD-topic.md` | facts |
| `assets/` | `how-it-works.html`, `architecture-simple.html`; both predate the 2026-09-14 architecture and are stale | visuals |

## Reading order

- New teammate: brief → landscape → design → privacy → plan.
- Program developer: design → client → crypto → privacy → decisions.
- LOI writer: brief → landscape → design → faq → research as needed.
- Pitching: brief → faq.

## Rules

1. Facts flow research → brief → design → everything else. Fix upstream first.
2. Research is never edited. Docs never carry history.
3. Two-line TL;DR at the top of every doc. Split a doc that passes ~1,500 words and note it here.
4. `status`: todo | draft | decided | verified. `last_verified`: a date.

## Status

| File | Status |
|---|---|
| brief.md | draft, refined after architecture rounds 2026-09-14 |
| landscape.md | draft |
| design.md | decided 2026-09-14 (handover revision), conditional on spikes S1 and S3; fallback named |
| client.md | decided 2026-09-14 |
| crypto.md | draft |
| privacy.md | draft |
| plan.md | draft |
| decisions.md | draft |
| faq.md | draft |
| loi/part1-proposal.md | todo |
| loi/part2-redteam-log.md | todo |

Architecture rounds of 2026-09-14: `research/2026-09-14-arch-r1-*` (diverge), `-r2-*` (converge), `-r3-*` (verify); candidates v1 and v2 are the round outputs. Research migrated from `~/Projects/mine/solana/capstone/research/` on 2026-09-11; see `research/README.md`. The old `capstone/` folder is now archive only.
