# Research index

Evidence only. Never edited after the date in the filename; add a new dated file instead. Every claim in the docs points here.

## Core (this project)

| File | What it holds |
|---|---|
| `2026-09-11-feasibility-stealth-confidential.md` | First deep dive: can a sender configure a confidential account for a stealth address; PDA + ECDH answer; scope; red team |
| `2026-09-11-skeptic-stealth-confidential.md` | Adversarial review: sweep re-linking, sender holds the key, payroll framing, cost, sRFC-42 dependency |
| `2026-09-11-token2022-mechanics-verified.md` | Source-verified Token-2022 facts: ConfigureAccount owner checks, key derivation, proof anatomy, CU, rent, slnt, prior art (Occult) |
| `2026-09-11-crypto-derivation-review.md` | Derivation tree with domain separation, Curve25519 checks, signing nonce binding, alternatives, reviewer checklist |
| `2026-09-11-competitors-privacy-verified.md` | Hinkal, Umbra, Helius Rings, Privacy Cash, Light PSP, Arcium: what they hide, trust assumptions, status; comparison table |
| `2026-09-11-compliance.md` | TFR, BSA, FinCEN mixing definition, MiCA, Tornado, auditor key ownership |
| `2026-09-11-demand.md` | Demand signals, target segments, the payroll correction |
| `2026-09-11-srfc-notes.md` | sRFC 37, 40, 42, 43 and RWA classification draft, verified via GitHub |

## Architecture rounds (2026-09-14)

| File | What it holds |
|---|---|
| `2026-09-14-arch-r1-obvious.md` | Conservative full-system draft: one Pinocchio program, byte layouts, Rust client, test pyramid |
| `2026-09-14-arch-r1-creative.md` | Nineteen alternatives ranked by value over verification cost; brief refinements B1 to B7 |
| `2026-09-14-arch-r1-mechanics-verified.md` | Owner checks, discriminants, proof sizes, CU, rent, inline offsets under CPI, ZK program status |
| `2026-09-14-arch-r1-crypto-privacy-skeptic.md` | Credit-counter griefing, nonce reuse, destination policy, torsion, BIP-352 citation |
| `2026-09-14-arch-r1-ops-cost-skeptic.md` | Batch failure modes, priority fees, dust sizing, hand-encoded CPIs, revised weeks |
| `2026-09-14-arch-r2-candidate-v1.md` | Synthesis after round 1: atomic open + configure + fund, context accounts, lookup tables |
| `2026-09-14-arch-r2-verified.md` | Counter semantics, SIMD-0296/0385 status, byte budgets, fee payer to zero, LiteSVM builtins, mint policies on chain, context authority, zk-sdk 8.0.0 |
| `2026-09-14-arch-r2-crypto-privacy-skeptic.md` | Public-credit close block, apply reopening the counter, payer follows D, torsion by multiply-by-L, k in seeds |
| `2026-09-14-arch-r2-design-skeptic.md` | Run secret backup, registry justification, close race, process debt |
| `2026-09-14-arch-r2-ops-cost-skeptic.md` | Lookup table capacity, P margin, missing open field, deterministic context keys, floor check split |
| `2026-09-14-arch-r3-candidate-v2.md` | Synthesis after round 2: everything inline, one transaction per side, sweep folds close |
| `2026-09-14-arch-r3-verified.md` | v1 transaction support across the stack, EmptyAccount and Transfer determinism, approval gating, caps, byte and CU tables, proof timing spike |
| `2026-09-14-arch-r3-fresh-eyes.md` | Cold implementability read: account-list gaps, stale proofs, recipient approval gap, estimate 45 to 60 days |
| `2026-09-14-arch-r3-crypto-privacy-skeptic.md` | Recipient approval gap, AE cache poisoning, D linkage, registry wallet, receipt after close, privacy wording |
| `2026-09-14-arch-r4-consistency.md` | Cross-check of the rewritten docs against the research: numbers, contradictions, stale content, rule compliance |
| `2026-09-14-arch-r4-final-skeptic.md` | Last pass before decided: freeze authority caveat, estimate versus schedule, torsion spike |
| `2026-09-14-arch-r5-e2e-simulation.md` | Cold step-by-step simulation of a three-recipient run, sweep, audit and failure cases; no source contradictions |
| `2026-09-14-arch-r5-handover-verified.md` | SetAuthority on a configured confidential account, later authorisations, close authority, sRFC-42 precedent |
| `2026-09-14-arch-r5-redesign-search.md` | Alternatives A to H ranked; handover adopted, D kept, announcement shrunk, approver as a policy program |

## Context (how we got here)

| File | What it holds |
|---|---|
| `2026-09-11-idea-funnel-round1.md` | 19 scored ideas across the four cohort domains |
| `2026-09-11-idea-funnel-round2.md` | Four finalists, multi-axis verdict, why this one |
| `2026-09-11-hackathon-winners.md` | Colosseum winners 2024–2026, what judges reward, saturated categories |
| `2026-09-11-ecosystem-priorities.md` | Foundation RFPs, Colosseum requests, primitive maturity in Sept 2026 |
| `2026-09-11-competitors-four-domains.md` | RWA, tokenization, payments, collectibles landscape |
| `2026-09-11-hard-open-problems.md` | Eleven hard problems ranked |
| `2026-09-11-turbin3-expectations.md` | Cohort deliverables and what "praised" capstones look like |
| `2026-09-11-onchain-checks-rwa-mints.md` | xStocks and Ondo mint extensions read from mainnet |

## Alternatives we did not choose

| File | What it holds |
|---|---|
| `2026-09-11-alt-permissioned-liquidity-layer.md`, `-skeptic.md`, `-tech.md` | Token ACL / hook-aware vault and exchange |
| `2026-09-11-alt-payment-channels.md` | Dispute layer for SF payment channels |
| `2026-09-11-alt-security-token-engine.md` | Corporate actions on Token-2022 |
