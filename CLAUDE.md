# Conventions for AI sessions

Project: private payments on Solana (stealth addresses + Token-2022 confidential balances, sender visible, no pool). Design phase; no code yet. Owner: Valentyn, who does the programs and cryptography and no frontend.

- `docs/brief.md` is canonical for scope; `docs/design.md` for how. Everything else derives or evidences.
- `docs/research/` is evidence: never edit, only add dated files.
- Docs describe the present; `docs/decisions.md` holds the why and the history.
- Every doc: front-matter `status` (todo | draft | decided | verified) and `last_verified`, then a two-line TL;DR. Under ~1,500 words.
- Committed text is human voice: no em dashes, no AI-tell words, no emoji bullets.
- Read `docs/README.md` first for the reading order per role.
- Every claim in a doc traces to a file in `docs/research/` or an on-chain check; mark anything else unverified.
- Open questions stay open with a recommendation and an owner in `docs/plan.md`; do not silently resolve them.
- Do not add competitor notes marked LOCAL or anything from private conversations to this repo.
- Every new crate gets `license = "MIT OR Apache-2.0"`, every new package.json gets `"license": "MIT OR Apache-2.0"`. No per-file license headers.
