# R3 — design-skeptic on T1 (2026-09-11)

## FATAL
- **F1 LP token is the compliance hole.** Underlying permissioned mint stays gated for free (DAS=frozen + gate on every transfer_checked, trader's own ATA is gated at receive). But the LP mint is a new, ungated claim on the basket: a non-KYC'd party can buy LP on a secondary market and hold economic exposure to a security. If issuer compliance requires KYC'd holders of *exposure*, the LP token reopens the leak one hop removed. Needs answer from tiago18c / compliance-literate reviewer; may require the LP mint itself to be DAS+gate wrapped (a second permissioned-mint problem).
- **F2 "PDA isn't KYC'd" is mostly a non-issue, but the doc never traces why.** Vault on allowlist only lets the pool hold/move; trader's inbound leg fails atomically if not thawed. Property of Token-2022, not of the AMM. Write the trace down and check against sRFC 37 intent.

## SERIOUS
- **S1 "vault re-frozen between simulate and execute" is padding.** Tx atomic; transfer to/from frozen account reverts; residual is retry/UX (integrator). One test, not a design problem.
- **S2 router-adapter is YAGNI.** Pool's own swap already needs meta resolution + thaw; no aggregator routes a zero-liquidity fixture; unverified it targets Jupiter's real `Amm` trait. Cut.
- **S3 AMM-as-vehicle contradiction.** Doc already says value = PR against token-acl + upstream #66 repro. A `compliant-vault` crate + example PR is reviewable by tiago18c today. Verify Turbin3 rubric actually requires a full program before letting it force an oversized AMM. A minimal vault program (init/thaw/deposit/withdraw) satisfies "one use case = one handler".
- **S4 single-leg hook meta resolution is not novel** (`add_extra_accounts_for_execute_cpi` exists). Novel part = N legs in one ix hitting #66.
- **S5 reentrant-hook test needs the actual Sealevel reentrancy rule cited** (self-recursion only) before it's a named test.

## MINOR
- M1 CPI-depth test is mechanical. #66 repro + upstream comment is the single most core-dev-impressive artifact.
- M2 Never say "Jupiter will route through it". "Architected to be routable" at most.

## 30-second reactions
- Anza maintainer: "You built a whole AMM to exercise a bug I filed 20 months ago. Where's the minimal repro / token-acl example, and did you talk to tiago18c about vault-thaw semantics?"
- Jupiter integrations: "No inventory → nothing to route; adapter not worth reviewing until it implements the real Amm trait against a pool with liquidity."

## Reshaped scope
1. `compliant-vault` minimal program: init_vault (permissionless thaw), deposit, withdraw wrapping token-acl gate CPI + single-leg hook resolution. Fills token-acl's missing vault example.
2. One thin consumer: two-instruction constant-product pool on top, framed as fixture for adversarial tests.
3. Cut router-adapter.
4. Headline = adversarial suite: #66 minimal repro filed upstream, CPI-depth test, correctly-sourced reentrancy test, frozen-vault documented as atomic failure mode.
5. Resolve first: does an ungated LP token need its own ACL gating?

**Biggest unresolved question:** ungated LP token = non-KYC'd economic exposure?
