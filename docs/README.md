# Docs index

TL;DR: one flat folder. `brief.md` says what and why, `design.md` says how, `research/` holds evidence, `decisions.md` holds history. Everything else derives from those.

## Structure

| File | Purpose | Canonical for |
|---|---|---|
| `brief.md` | What we build and why, one page, ends with a claims table | scope |
| `landscape.md` | Competitors, positioning, compliance posture, target users | market |
| `design.md` | Components, keys, accounts, instructions, one payment, costs | how |
| `crypto.md` | Key derivation, curve checks, reviewer checklist | cryptography |
| `privacy.md` | Threat model, what an observer sees, residual leaks | privacy guarantees |
| `plan.md` | Phases, milestones, risks, open questions with owners | what happens next |
| `decisions.md` | Append-only dated log: what, why, alternatives rejected | history |
| `faq.md` | Questions people ask, answers, short message variants | comms |
| `loi/` | Cohort deliverable: proposal and red-team log | grading |
| `research/` | Dated evidence, immutable, `YYYY-MM-DD-topic.md` | facts |
| `assets/` | `how-it-works.html` (current explainer), `architecture-simple.html` | visuals |

## Reading order

- New teammate: brief → landscape → design → privacy → plan.
- Program developer: design → crypto → privacy → decisions.
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
| brief.md | draft, re-centred 2026-09-14 |
| landscape.md | draft |
| design.md | draft, sweep corrections 2026-09-14 |
| crypto.md | draft |
| privacy.md | draft |
| plan.md | draft |
| decisions.md | draft |
| faq.md | draft |
| loi/part1-proposal.md | todo |
| loi/part2-redteam-log.md | todo |

Research migrated from `~/Projects/mine/solana/capstone/research/` on 2026-09-11; see `research/README.md`. The old `capstone/` folder is now archive only.
