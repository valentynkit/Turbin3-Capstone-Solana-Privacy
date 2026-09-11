# R2 — T2 RWA security-token engine: feasibility + red team (2026-09-11)

## Verdict: Go-with-changes
Compliance/identity/freeze layer is solved (Token ACL mainnet `TACLkU6CiCdkQN2MjoyDkVg2yAH9zkxiHDsiztQ52TP`, Bridgesplit policy_engine, SRWA compliance_modules) — consume via CPI. The open slice: **corporate-actions state machine (announce→record→execute with ScaledUiAmount scheduling)** + **trust-minimised record-date snapshot + crank-less distribution**. Building all seven capabilities from scratch repeats sRFC 00020's scope-creep death.

## Sources
- sRFC 00020 (July 2023, "Dom"): real-estate → general RWA pivot, replies through April 2024, no closure, no impl. RFP #526 deadline Sept 29 2023, one late submission (Rubicon Studio SA), no award.
- Bridgesplit `rwa-token` = the QuillAudits "RWA Token Program": https://github.com/LoopscaleLabs/rwa-token (MIT, last push Oct 2024, "not for active use"); modules asset_controller / policy_engine / identity_registry / data_registry. `revoke_tokens` via Permanent Delegate; `update_interest_bearing_mint_rate` (InterestBearing, not ScaledUiAmount). **No dividend/split/clawback code** despite README.
- https://github.com/SRWA-Cypherpunk/SRWA (Oct–Nov 2025): factory, identity_claims, compliance_modules, controller (hook orchestrator), offering_pool, purchase_order; "dividend_distribution" exists only as a frontend template string.
- Superstate Opening Bell: off-chain SEC-registered register, on-chain tokens as synced pointers. Ondo GM: dividends auto-reinvested, mint/redeem paused around corporate actions. Production issuers sidestep on-chain corporate actions.
- ScaledUiAmount source: `authority, multiplier, new_multiplier, new_multiplier_effective_timestamp` — scheduled flip at record date, no crank. Raydium CLMM allow-lists ScaledUiAmount (token.rs:321-325). InterestBearing exclusivity: unverified in source.
- **On-chain (07-onchain-checks.md): xStocks and Ondo already use ScaledUiAmount + Pausable → splits/halts are not novel.**
- Light Protocol mainnet programs (Lighton6…, compr6C…, SySTEM1…, cTokenmW…), audited; no corporate-actions distributor example; Jito `distributor` is the generic merkle pattern. Integration work, not off-the-shelf.
- Holder snapshot: Token-2022 has no enumerable holder list → indexer required, or (a) issuer commits merkle root + challenge window, or (b) transfer-restriction program keeps per-holder checkpoint ledger (CU cost forever).
- SEC May 15 2025 FAQ: registered transfer agent may use DLT as Master Securityholder File (hybrid on/off-chain OK); staff-level, rulemaking (17ad-12/17ad-30) in flight. EU MiCA/DLT Pilot: primary sources not fetched (unverified).

## Scope (proposed)
One `corporate-actions` Anchor program. Handlers: announce_action (["action", mint, seq]), commit_snapshot_root, dispute_snapshot (optional), execute_split (CPI update_multiplier with effective ts), fund_distribution, claim_distribution (merkle proof + Light compressed bookkeeping), sweep_unclaimed, toggle_pause, init_action_config. Hard: snapshot trust, dispute, compressed claim CPI. Plumbing: the rest.

## Red team
- Core dev: mostly CRUD-with-extra-steps; splits trivial (one scheduled CPI); the only real problem is snapshot + concurrent compressed claims + reverse-split rounding.
- Securities lawyer: issuer-committed merkle root with challenge window is still issuer asserting the list; "trust-minimised" is aspirational, not legal. No issuer will point a shareholder-record obligation at an unaudited student program. Production issuers keep register off-chain.
- Needs T1-style liquidity to matter.

## Kill criteria
No DEX pools ScaledUiAmount → not triggered (Raydium does). ScaledUiAmount incompatible with PermanentDelegate/Pausable → unresolved (xStocks combines all three on-chain → not triggered, per 07). Legally credible snapshot impossible without trusted indexer → likely triggered. Issuer already ships generic on-chain corporate actions → not triggered. sRFC 00020 active → not triggered.

## Who cares
Helius/Light, SF grants, Bridgesplit (inactive), token-acl maintainers, sRFC 00020 repliers.
