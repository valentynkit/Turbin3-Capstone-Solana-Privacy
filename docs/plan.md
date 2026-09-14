---
status: draft
last_verified: 2026-09-14
---

# Plan

TL;DR: crypto review first, then the rail, then the primitive, measure early, demo a payout run from a terminal. Open questions have owners and dates.

## Phases and exit criteria

| Week | Phase | Exit criterion |
|---|---|---|
| 1 | Derivation review; privacy model final; LOI submitted (due Sep 14) | derivation tree reviewed by one external person |
| 1 to 2 | Vault and plain path: payer client does a batched confidential payout to N pre-configured recipients on LiteSVM, then devnet | N-recipient run works; first cost numbers |
| 2 to 3 | Registry; open + configure + fund with ECDH-derived key | stealth account configured and funded with the recipient offline |
| 3 to 4 | Scan, sweep (P signs, P pays), close; client refuses reused destinations | end-to-end stealth payment on devnet |
| 5 | Adversarial tests; benchmarks; auditor decrypt demo | attacks rejected; numbers published |
| 6 | Surfpool run, status page, demo script, sRFC-42 write-up | three-minute terminal demo of one payout run, mixed modes, then an auditor reading one payment |

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Derivation flaw found in review | key redesign | review before code |
| Proof generation too slow in TypeScript for a batch | client rewrite in Rust | measure in week 2 with @solana/zk-sdk wasm |
| Token-2022 confidential program disabled again (2025 to 2026 precedent) | demo blocked | LiteSVM tests as fallback demo |
| Regulatory reading of single-use addresses (FinCEN 2023 proposal) | positioning | visible payer as a hard constraint; disclosure paths documented; outside counsel before "compliant" claims |
| No wallet support | adoption | CLI for the capstone; a wallet SDK is the obvious follow-up |
| Team below three | scope | drop status page and relayer first |

## Open questions

| Id | Question | Owner | Resolve by | Recommendation |
|---|---|---|---|---|
| Q1 | Domain separation of the two derivations from S | Valentyn | week 1 | post the tree to sRFC-42; ask one Anza zk contributor |
| Q2 | Resolved: derivation function is published (7.0.1 / 0.5.2) | | | |
| Q3 | Payer funding source: confidential-only or allow public deposit | Valentyn | week 2 | confidential-only in v1 |
| Q4 | Sweep destination: refuse reused, or warn | team | week 3 | refuse by default |
| Q5 | FinCEN mixing definition and single-use addresses | Valentyn | before public claims | assume it applies; keep payer visible |
| Q6 | Named Solana payroll or payout platform that wants this | team | week 2 | ask Zebec, Streamflow, Superteam Earn |
| Q7 | Batch size before lookup tables and proof staging bottleneck | teammate | week 2 | measure |
| Q8 | MPC-based consolidation as phase three | team | after demo | talk to Arcium |

## Not yet a file

Spec (sRFC candidate), user stories, architecture diagram deliverable, brand, demo script. Each becomes a file when it has content.
