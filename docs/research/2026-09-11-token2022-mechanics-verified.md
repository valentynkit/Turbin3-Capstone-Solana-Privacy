# R3 — P2 technical design facts (source-verified, 2026-09-11)
Sources: solana-program/token-2022 @a3e696e, solana-program/zk-elgamal-proof @06c299c, susruth/slnt, solana-foundation/Confidential-Balances-Sample, agave precompiles, Wormhole fix commit 274bb7c.

## ConfigureAccount + PDA owner
- Accounts: token account (w), mint, instructions sysvar or proof context, owner (signer). `process_configure_account` (processor.rs:220-276) → `Processor::validate_owner` (processor.rs:2253-2292): only checks key match + is_signer (or multisig). **No on-curve check anywhere. PDA via invoke_signed works.** Same for ApplyPendingBalance (processor.rs:1196-1246) — needs invoke_signed too.
- `PubkeyValidityProofData` = Schnorr PoK of the ElGamal secret; not tied to owner identity.
- `decryptable_zero_balance` (AES-128-GCM-SIV, 36B) stored unchecked.
- Key derivation: `zk-sdk/src/encryption/derivation.rs`: HKDF-SHA512 extract(salt "solana-conf-bal/v1", ikm) → expand info "ae" (16B) and "elgamal" (64B mod L). `derive_confidential_keys_from_ikm(&[u8])` takes raw bytes. Ships `pda_wallet_public_seed(program_id, wallet_pda, mint, token_account)` explicitly for PDA-owned accounts. Added 2026-07-24 (commit af8e9f6) — may be main-only, not yet in crates.io v7.0.1 (UNVERIFIED).
- `maximum_pending_balance_credit_counter` default 65536.

## Client tooling (Sept 2026) — mature
- `@solana/zk-sdk` npm 0.5.2 (2026-08-27): wasm, `ConfidentialKeys.fromIkm(bytes)`, proof data types.
- `@solana-program/zk-elgamal-proof` 0.4.0; `@solana-program/token-2022` 0.17.0 (2026-09-10) with full confidential instruction-plan builders (confidentialTransferHelpers.ts, 2055 lines). TS client feasible; Rust CLI not required. Browser range-proof speed unbenchmarked.

## Transfer anatomy
- Plain Transfer: equality (6,400 CU) + batched 3-handle validity (16,400) + **range U128** (200,000) ≈ 222,800 CU + 3×3,300 closes; token logic ~31k on top. TransferWithFee ≈ 410k CU (U256 range).
- U128 range proof ~1.4KB > 1232B tx limit → staged via spl-record account; reference client = **5 transactions per Transfer** (equality ctx, validity ctx, record create+chunk, chunk+range ctx, Transfer+closes).
- Auditor: mint `auditor_elgamal_pubkey` must equal proof's 3rd handle (processor.rs:683-705) — structurally forced when set.

## Sweep + raw-scalar Ed25519
- RFC 8032 verify is algebraic; agave precompile → `ed25519_dalek::verify_strict`; raw-scalar signatures accepted. `ed25519-dalek hazmat::ExpandedSecretKey`/`raw_sign` (warning: Double Public Key Signing Oracle — bind hash_prefix to scalar).
- slnt does raw-scalar signing (stealth_signing.rs) but as a **top-level tx signer**, never in-program. In-program precompile + sysvar introspection is unbuilt for this use → real gap.
- Checklist (Wormhole lesson): check sysvar address == instructions::id(); introspected program_id == ed25519 program; exact pubkey/msg/sig match; fixed relative index.

## Rent/size
- Token acct + ImmutableOwner + ConfidentialTransferAccount = 469B ≈ 0.00416 SOL. Proof ctx accounts: pubkey-validity 65B (0.00134), equality 161B (0.00201), validity 385B (0.00357), range 297B (0.00296) — all reclaimable via CloseContextState.
- Per stealth payment ≈ 0.0127 SOL locked, 0.0042 standing until close.

## slnt
- Ed25519 spend / X25519 scan; HKDF-SHA256 salt "slnt-v1-derive"; view tag = SHA-256(…)[0]; Pinboard `post` stateless Anchor event {scheme_id u16, ephemeral_pub 32, view_tag u8, metadata ≤64B}; Registry PDA ["meta", registrant, scheme_id_le], 101B. MIT. Test vectors. Active 2026-05-20→06-05 only.

## Prior art for PDA-owned confidential accounts (novelty softens)
- `Occult-fi/occult` (confidential batch-auction AMM, pool-PDA-owned vaults, invoke_signed configure/deposit/apply), `pupplecat/gacha-sol`, `kilogold/confidential_escrow` (README only), `harsh4786/Confidential-Escrow` (old API). Zero PDA-owner examples in token-2022 tests or SF samples. Claim: hardening/documenting/generalizing + ECDH-derived keys + in-program raw-scalar sweep (untouched by prior art).

## Per-payment tx sequence (12 txs incl. sweep)
1 init PDA + create ATA (+0.00416) · 2 announce · 3 configure (inline PubkeyValidity) · 4-6 proof ctx for funding Transfer (+0.00854 transient) · 7 confidential Transfer + closes (~254k CU) · 8 apply pending (recipient/relayer) · 9-11 sweep proof ctx · 12 sweep: ed25519 precompile + sysvar verify + invoke_signed Transfer + closes + close account (reclaim).
