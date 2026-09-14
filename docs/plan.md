---
status: draft
last_verified: 2026-09-14
---

# Plan

TL;DR: two day-one spikes decide whether the one-transaction design stands, then program, engine, recipient, tests, demo. Open questions have owners and dates; the cut order is written down before the schedule slips.

## Spikes, in order

| Id | Question | Exit criterion | When |
|---|---|---|---|
| S1 | v1 (4096-byte) transactions end to end: build with solana-transaction, run in LiteSVM and Surfpool, send to devnet | one 3 KB transaction with four ZK verify instructions lands in all three | day 1 |
| S2 | ConfigureAccount with an inline proof behind a CPI from a PDA-owned Pinocchio program | LiteSVM test passes | day 1 |
| S3 | Handover and sweep: open sets close authority and hands ownership to P; P then applies, transfers, empties with a precomputed zero-ciphertext proof, and reclaim closes both accounts, all in one transaction with no program call except reclaim | LiteSVM test passes | day 2 |
| S4 | `getRecentPrioritizationFees` distribution over several days for Token-2022 and the ZK program | p50, p75, p95 in a table | week 1, background |
| S5 | Measured bytes and CU of both transactions once encoders exist | numbers in design.md | week 2 |
| S6 | `getProgramAccounts` with the view-tag filter against 1,000 seeded accounts on the RPC provider we use | allowed, latency measured | week 3 |
| S7 | curve25519 syscalls (validate, multiply by L) called from a `no_std` Pinocchio program in LiteSVM | torsion check rejects a small-order point, accepts a valid one, CU measured | week 2; a bad B_spend only harms its owner, so this never blocks |

Done 2026-09-14 (research: arch-r3-verified): proof generation timing; v1 support present in the Rust SDK, LiteSVM and Surfpool; ZK ElGamal builtin loads in LiteSVM by default; the fee payer may drain to zero.

## Phases and exit criteria

| Week | Phase | Exit criterion |
|---|---|---|
| 1 | S1, S2, S3, S4; derivation review requested; encoders for the Token-2022 instructions the program and client build (InitializeAccount3, ConfigureAccount, DisableNonConfidentialCredits, SetAuthority, ApplyPendingBalance, Transfer, EmptyAccount, CloseAccount) with golden-byte tests | spikes green or the fallback chosen; encoders match the interface crate byte for byte |
| 2 | Plain path first, with no program deployed: the engine pays 10 plain recipients on devnet; then program register and open; journal and resume; S5, S7 | plain run on devnet; 10 stealth payments open on LiteSVM; run resumes after a kill at every step |
| 3 | Program: reclaim; recipient scan and sweep with plain instructions; refuse reused destinations; S6 | end-to-end stealth payment on devnet, one transaction per side, rent back with the payer |
| 4 | Chaos harness; adversarial suite; Mollusk CU assertions in CI | attacks rejected; harness converges from every kill point |
| 5 | 50-recipient run on devnet; measurements published; receipt verifier | numbers in design.md replace every [likely] cost |
| 6 | Recorded demo; sRFC-42 write-up; status page | three-minute terminal demo, mixed modes, then a receipt verified by a third party |

Estimate from a cold read (research: arch-r3-fresh-eyes): 45 to 60 engineer-days for one engineer new to Pinocchio and confidential transfers, against about 30 working days in the table. Full scope stands; the work is AI-assisted throughout, and the cut order below is the safety valve if week 1 velocity says otherwise.

## Cut order

1. Status page.
2. Mollusk as a separate CI job (LiteSVM reports CU too).
3. Receipt verifier CLI (ship the format, verify by script).
4. 50-recipient run (ship 10 with the numbers marked as such).
5. Chaos harness (keep the resume test at three kill points).

Never cut: golden-byte tests, the adversarial suite, the nonce regression test.

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| v1 transactions unusable somewhere in the loop | transaction packing changes | S1 on day 1; fallback is research/2026-09-14-arch-r2-candidate-v1 for packing (context accounts, several transactions per side); handover unaffected |
| Derivation flaw found in review | key redesign | review requested in week 1, before sweep is written |
| Hand-written Token-2022 encoders wrong | silent CPI failures | golden-byte tests against spl-token-2022-interface 3.1.1 before any CPI |
| Priority fees under congestion | P underfunded, sweeps stall | S4; floor check before every send; overprovision P |
| Devnet faucet caps | demo blocked | fund from week 1; LiteSVM and Surfpool as the recorded fallback |
| Token-2022 confidential program disabled again (2025 precedent) | demo blocked | same fallback |
| Regulatory reading of single-use addresses | positioning | visible payer as a hard constraint; disclosure paths documented; counsel before "compliant" claims |
| No wallet support | adoption | CLI for the capstone; a wallet SDK is the obvious follow-up |

## Open questions

| Id | Question | Owner | Resolve by | Recommendation |
|---|---|---|---|---|
| Q1 | Domain separation of the derivations from S, and `d` from `b_spend` | Valentyn | week 1 | post the tree to sRFC-42; ask one Anza zk contributor |
| Q3 | Payer funding source: confidential-only, or allow a public deposit path | Valentyn | week 2 | confidential-only in v1 |
| Q5 | FinCEN mixing definition and single-use addresses | Valentyn | before public claims | assume it applies; keep the payer visible |
| Q6 | Named Solana payout platform that wants this | Valentyn | week 2 | ask Zebec, Streamflow, Superteam Earn |
| Q8 | MPC-based consolidation as phase three | Valentyn | after demo | talk to Arcium |
| Q9 | Issuer path for PYUSD and USDG | Valentyn | after demo | one-page spec in the write-up: a policy program whose PDA is the mint's confidential-transfer authority and approves every account, plus the deferred-handover open; do not build |
| Q10 | Never-swept accounts | Valentyn | phase two | after handover the payer cannot reclaim them at all: only P can empty the account. State it as a limitation; a timeout reclaim would need a different custody model and is not planned |
| Q11 | One ephemeral key per run (scan cost versus grouping) | Valentyn | after Q1 | defer until the review has seen the tree |

Resolved 2026-09-14, see decisions.md: Q2 (derivation function published), Q4 (refuse reused destinations by default), Q7 (lookup tables are not the lever; proofs are inline and sends are sequential per source account), Q12 (on-chain registry stays), Q13 (full scope stays).

## Not yet a file

Spec (sRFC candidate), demo script, issuer approver spec. Each becomes a file when it has content.
