# R3 — T1 technical design facts (source-verified, 2026-09-11)

## Token ACL (token-acl @bfbaf589, token-acl-gate @c525fa71, mainnet TACLkU6…)
- `ThawPermissionless` (disc 6, idempotent 9): only `authority` signs; `token_account_owner` is a plain AccountInfo (pubkey compare). Anyone can pay to thaw anyone. `FreezePermissionless` (disc 7/10) signed by MintConfig PDA, gated by `enable_permissionless_freeze`.
- ImmutableOwner check lives in the **gate** (`token-acl-gate/program/src/instructions/can_thaw_permissionless.rs:28-30`), not core; custom gates need not enforce it.
- Gate CPI de-escalates everything to readonly/non-signer (`interface/src/onchain.rs`), plain `invoke`.
- `MintConfig` PDA ["MINT_CONFIG", mint]: bump, enable_permissionless_thaw/freeze, mint, freeze_authority (human), gating_program; mint's real freeze authority reassigned to the PDA.
- ABL gate keys by token-account owner pubkey (`WalletEntry` ["wallet_entry", list_config, wallet]); PDA-owned vault allow-listed identically via `AddWallet`.
- `token_acl` metadata key (underscore) in TokenMetadata additional_metadata = gate program (clients/rust/src/metadata.rs). Wallet auto-detect unverified/reported broken in Phantom.
- ABL program GATEzzqxhJnsWF6vHRsgtixxSB8PaQdcqGEVTEHWiULz: CreateList, Add/RemoveWallet, SetupExtraMetas, SetupFreezeExtraMetas, DeleteList, CanThaw/CanFreezePermissionless.

## Transfer hooks
- Extra metas PDA ["extra-account-metas", mint] under hook program. `add_extra_accounts_for_execute_cpi` does no on-chain fetch; client must pre-supply all accounts. Typical 2–8 extras.
- **Issue #66 root cause fixed upstream**: quadratic heap growth in `ExtraAccountMetaList::add_to_cpi_instruction`, fixed by solana-program/libraries PR #199 (merged 2026-08-26), `spl-tlv-account-resolution >= 0.11.2`. Issue itself still open with 0 comments. → T1's "#66 repro" artifact is moot; only a "close the issue with a pointer to the fix" comment remains.
- Anchor 1.2 `token_2022_extensions::transfer_hook` wraps only initialize/update, not invoke.
- CPI depth: `MAX_INSTRUCTION_STACK_DEPTH = 5` (top + 4 CPIs); SIMD-0268 (→9) not active on mainnet. AMM→token-2022→hook→hook's CPI uses 4/5: hooks must be single-hop.

## Orca / Meteora
- Orca whirlpools calls `add_extra_accounts_for_execute_cpi` in custom wrappers; `RemainingAccountsInfo` slices per mint; TokenBadge gates hook/permDelegate/DAS/pausable mints. Issue #1372 open (TS SDK drops hook accounts). License: Apache-2.0 only through 2025-02-26, custom non-commercial after.
- Meteora DLMM: hook allowed only if program + authority revoked; otherwise manual badge.

## Coherence
- token-2022 frozen check symmetric (processor.rs:363 source, :523 dest); DAS applies to every InitializeAccount, no PDA exemption. Pool vault passes the same gate once; traders per-account. Hook `execute` sees source/dest/owner so a hook *could* exempt a vault — no canonical implementation. LP-mint permissioning: no prior art found either way.

## Jupiter
- `jup-ag/jupiter-amm-interface` `Amm` trait: static `get_accounts_to_update`; hook metas must be resolved by the adapter's own `get_swap_and_account_metas` (offchain resolve) and declared via `RemainingAccountsInfo` slices (`AccountsType::TransferHookA/B`). No dynamic-resolution callback. No canonical extension policy; Trigger rejects hook mints unless whitelisted.

## Testing
- LiteSVM/Mollusk expose `heap_size`, `compute_unit_limit`, `max_instruction_stack_depth`; Mollusk loads token-2022 ELF (`mollusk-svm-programs-token`, extensions feature) + other programs. token-2022 `clients/rust-legacy/tests/transfer_hook.rs` (1280 lines) has adversarial fixtures (_fail.so, _downgrade.so, _sentinel_amount.so). No public audit reports specific to hooks.
- Surfpool: `surfnet_setTokenAccount` only targets ATAs; `surfnet_setAccount` raw-byte overwrite lets you mutate a forked xStocks mint's hook program / freeze authority by hand-rolling TLV bytes. Lazy mainnet fetch, no clone call needed.

## Implication for the decision
T1's most core-dev-impressive artifact (#66 repro) evaporated in Aug 2026; remaining novelty = token-acl vault example + hook-aware pool + LP-gating design question. Confirms P2 as #1.
