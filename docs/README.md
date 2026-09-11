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
| `assets/` | Diagram sources, pitch page | visuals |

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
| brief.md | todo |
| landscape.md | todo |
| design.md | todo |
| crypto.md | todo |
| privacy.md | todo |
| plan.md | todo |
| decisions.md | todo |
| faq.md | todo |
| loi/part1-proposal.md | todo |
| loi/part2-redteam-log.md | todo |

Migration note: source material currently lives in `~/Projects/mine/solana/capstone/` (BRIEF, ARCHITECTURE, DECISION, research/01-18). It moves here file by file as each doc is written.
