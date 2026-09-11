# Conventions for AI sessions

_todo_

- `docs/brief.md` is canonical for scope; `docs/design.md` for how. Everything else derives or evidences.
- `docs/research/` is evidence: never edit, only add dated files.
- Docs describe the present; `docs/decisions.md` holds the why and the history.
- Every doc: front-matter `status` (todo | draft | decided | verified) and `last_verified`, then a two-line TL;DR. Under ~1,500 words.
- Committed text is human voice: no em dashes, no AI-tell words, no emoji bullets.
- Read `docs/README.md` first for the reading order per role.
- Every new crate gets `license = "MIT OR Apache-2.0"`, every new package.json gets `"license": "MIT OR Apache-2.0"`. No per-file license headers.
