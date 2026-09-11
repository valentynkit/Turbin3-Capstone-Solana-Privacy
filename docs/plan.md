---
status: draft
last_verified: 2026-09-11
---

# Plan

TL;DR: six weeks, crypto review first, measure early, demo from a terminal. Open questions have owners and dates.

## Phases and exit criteria

| Week | Phase | Exit criterion |
|---|---|---|
| 1 | Crypto review; privacy model written; LOI submitted (due Sep 14) | derivation tree reviewed by one external person; LOI graded |
| 1 to 2 | Registry; open + configure on LiteSVM | PDA-owned account configured with an ECDH-derived key in a test |
| 2 to 3 | Fund path with proofs; CLI pay | confidential transfer into a one-time account on devnet; first cost measurements |
| 3 to 4 | Scan, sweep, close; CLI scan / sweep | end-to-end payment on devnet; sweep authorised by a raw-scalar signature |
| 5 | Adversarial tests; benchmarks; relayer | sysvar spoofing, replay, wrong-P, double fund, poisoned apply-pending all rejected; numbers published |
| 6 | Surfpool run, status page, demo script, sRFC write-up | three-minute terminal demo; post in the sRFC-42 discussion |

## Milestones

_todo: dates once the cohort calendar is known._

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Derivation flaw found in review | redesign of keys | review in week 1, before code |
| Proof generation too slow or too large in TypeScript | client rewrite in Rust | measure in week 2 with @solana/zk-sdk wasm |
| Token-2022 confidential program disabled again (happened 2025 to 2026) | demo blocked | keep LiteSVM tests as the fallback demo |
| Regulatory reading of single-use addresses | positioning | visible sender as a hard constraint; disclosure paths documented |
| Team size below three | scope | drop relayer and status page first |

## Open questions

| Id | Question | Owner | Resolve by | Recommendation |
|---|---|---|---|---|
| Q1 | Domain separation of the two derivations from S | Valentyn | week 1 | post the tree to sRFC-42; ask one Anza zk contributor |
| Q2 | Sender funding source: confidential-only or allow public deposit | Valentyn | week 2 | confidential-only in v1 |
| Q3 | Default sweep destination: fresh stealth account or user choice | team | week 3 | fresh by default in the CLI |
| Q4 | FinCEN mixing definition and single-use addresses | Valentyn | before public claims | assume it applies; keep sender visible |
| Q5 | Named Solana payroll or DAO platform that wants this | team | week 2 | ask Zebec, Streamflow, Superteam Earn |
| Q6 | MPC-based consolidation as phase three | team | after demo | talk to Arcium |
| Q7 | Batch tooling for payroll (phase two) | team | after demo | design once costs are measured |

## Not yet a file

Spec (sRFC candidate), user stories, architecture diagram deliverable, brand, demo script. Each becomes a file when it has content; note it in README.md.
