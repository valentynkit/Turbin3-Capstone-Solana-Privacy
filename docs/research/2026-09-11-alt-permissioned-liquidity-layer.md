# R2 — T1 Permissioned-token liquidity layer: feasibility + red team (2026-09-11)

## Verdict: Go-with-changes
Gap is real (no major DEX can hold a mint with TransferHook / PermanentDelegate / DefaultAccountState in its vaults; no reference for permissionless thaw of a protocol-owned vault), but CLOB+RFQ+router+suite is oversized. Narrow to **AMM vault + compliance adapter + adversarial suite**; don't fork Manifest.

## Solved vs open
- sRFC 37 text already *designs* vault thaw: "call permissionless thaw on the same transaction, or asynchronously… A protocol would need direct support if there are token transfers on the same instruction that creates the vault." https://github.com/solana-foundation/SRFCs/discussions/2
- token-acl `ThawPermissionless` (program/src/instructions/thaw_permissionless.rs) does not need the token-account owner to sign; gate keys allow/block by *owner address*, so PDA-owned accounts work. **Zero reference implementation / example / test** for a vault in https://github.com/solana-foundation/token-acl or https://github.com/solana-foundation/token-acl-gate. Gotcha found in code: gate's `can_thaw_permissionless` requires the token account to have `ImmutableOwner` — custom non-ATA vaults must add it.
- Open, confirmed: (1) atomic create+deposit `initialize_pool` pattern needs "direct support", no design given; (2) router mid-tx ATA create+thaw for hooked/ACL legs unaddressed; (3) hook extra-account-meta resolution in a match loop has no prior art; (4) token-2022 #66 heap OOM with ≥2 hooked transfers — open, filed by joncinque Jan 2025, 0 comments. https://github.com/solana-program/token-2022/issues/66
- The dev.to "hook CPI-depth reentrancy" article is 404'd, uncited, not credible — cite the 4-level CPI depth limit itself instead.
- DEX status: Raydium `is_supported_mint()` rejects TransferHook/PermanentDelegate/DefaultAccountState (raydium-cp-swap, raydium-clmm); Meteora DLMM allows hook only if hook authority revoked; Orca gates behind manual Token Badge and its TS SDK drops hook remaining-accounts (orca-so/whirlpools#1372 open). **No DEX auto-thaws its own vault.**
- RFP "Unified Security Token/RWA Program" (forum.solana.com/t/unified-security-token-rwa-program/526): 6 posts, last Oct 2023; sRFC 00020: last April 2024; zero staff-badged posts in either. Stalled. Don't frame as "targeting the RFP".
- Manifest (GPL-3.0, ~26.5k LoC, Pinocchio, no Anchor): detects TransferHook only to refuse; zero frozen-account handling. Don't fork for 6 weeks.
- Closest prior art: https://github.com/fabrknt/accredit (hook program + compliant-registry + Jupiter adapter; solo, 0 stars, Feb 2026, no DEX partnership).

## Scope (proposed)
Programs: `compliant-vault-core` (pool + vault PDAs, deposit/withdraw/swap wrapping Token-2022 ACL + hooks) and `router-adapter` (thin CPI shim resolving extra metas / permissionless thaw for one swap leg).
Handlers: init_pool (no deposit in same ix), thaw_vault (permissionless, CPI token-acl), deposit_liquidity, withdraw_liquidity (handle re-frozen vault), swap (hard: runtime hook meta resolution + thaw-state race), resolve_hook_metas, sync_gate_status, register_hook_program / register_gate_program, emergency_pause.
PDAs: ["pool", mint_a, mint_b], ["vault", pool, mint], ["lp_mint", pool], ["compliance_cfg", mint].
CPIs: token-2022 transfer_checked, token-acl thaw/freeze_permissionless, spl_transfer_hook_interface resolve + hook execute, token-acl-gate can_thaw_permissionless.
Hard: swap + races; adversarial suite (LiteSVM/Mollusk fuzz: frozen-vault races, #66 repro, CPI-depth exhaustion, reentrant hook). Plumbing: everything else.

## Red team
- "Market routes around it": Backpack Securities/Ondo GM trade on ordinary AMMs by not enabling the extensions. (Confirmed on-chain, see 07-onchain-checks.md: hooks armed but null.) Addressable problem may be smaller than it looks — or it's exactly the blocker.
- Securities law: permissionless CLOB for securities hits broker-dealer/ATS rules; every real venue keeps a regulated intermediary.
- "Just an AMM with a thaw ix" — for the scoped version, largely yes; sell it as reference + adversarial findings, not novel CLOB infra.
- CLOB in 6 weeks unrealistic.
- Adoption path: ship as example/PR against token-acl itself; file #66 repro + CPI tests upstream to token-2022.

## Kill criteria
- Major DEX ships permissionless vault thaw → false today.
- sRFC 37 has reference vault impl → false (spec paragraph only).
- #66 fixed → false.
- No real RWA token uses these extensions → **checked on-chain: xStocks + Ondo carry the extensions with hook=null, DAS=initialized** (07-onchain-checks.md). Problem validated as "want it, can't turn it on".
- RFP dead → true; reframe.

## Who cares
tiago18c (sRFC 37 / token-acl), joncinque (#66), Orca eng (whirlpools#1372), fabrknt/accredit author, SF grants DB.
