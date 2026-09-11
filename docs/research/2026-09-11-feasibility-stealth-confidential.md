# R2 — P2 Silent payments × confidential balances: feasibility + red team (2026-09-11)

## Verdict: Go-with-changes
Crux: can a sender configure a confidential Token-2022 account for a stealth address whose key it doesn't hold? **No for a raw Ed25519 stealth key** — `ConfigureAccount` needs PubkeyValidityProof + owner signature (`validate_owner`); `ConfigureAccountWithRegistry` needs an `ElGamalRegistry` PDA whose `CreateRegistry` also needs the wallet signature. Source: https://github.com/solana-program/token-2022/blob/main/program/src/extension/confidential_transfer/processor.rs , https://github.com/solana-program/token-2022/blob/main/confidential/elgamal-registry-interface/src/instruction.rs
**Yes with a protocol change**: make the stealth owner a **PDA** (program signs via invoke_signed) and derive the ElGamal keypair from the ECDH shared secret via `derive_confidential_keys_from_ikm` (zk-elgamal-proof/zk-sdk/src/encryption/derivation.rs, accepts arbitrary IKM). Sender computes `e·B_scan`, recipient computes `b_scan·E` → same IKM → same ElGamal key; sender generates PubkeyValidityProof and configures the PDA-owned account in the same tx as the payment. Sweep requires recipient to prove knowledge of spend key for `P = B_spend + t·G` (Ed25519 precompile + sysvar introspection). **Cryptographic soundness of dual-party-derivable ElGamal key: UNVERIFIED — needs review. This is the one genuinely novel piece.**

## Solved vs open
- sRFC 42 specifies derivation (Ed25519 spend key, X25519 scan key, HKDF-SHA256 tweak, 1-byte view tag, bech32m meta-address); reference https://github.com/susruth/slnt (MIT, unaudited, test vectors). Candidate program IDs return null on mainnet → devnet only. Repo: one week of commits (2026-05-31→06-05) then silent; discussion has 1 comment (liveness objection, unanswered).
- Token-2022 confidential transfer is production: Configure → Deposit → ApplyPendingBalance → Transfer → Withdraw; mint-level auditor ElGamal pubkey with consistency proof. https://solana.com/docs/tokens/extensions/confidential-transfer
- SF compliance framing: "Auditor keys provide read access; they do not authorize transfers" https://github.com/solana-foundation/solana-com/pull/1976
- Nobody combines per-payment unlinkability with amount-hiding, Solana or EVM: Helius Rings (pooled anonymity set), Arcium CSPL (MPC balances, no stealth), Umbra/Fluidkey (amounts public), MagicBlock Private Payments (TEE).

## Scope (proposed)
Programs: `stealth-registry` (meta-address publication) + `confidential-pinboard` (announcement + PDA-owned stealth account lifecycle, CPIs into Token-2022).
Handlers (~11): register_meta_address; post_announcement (ephemeral pub + view tag, no amount); init_stealth_pda (bind PDA to P — hard); create_stealth_ata; configure_confidential_via_ecdh (hard, the crux); deposit_confidential (must be confidential→confidential Transfer, see leak below); apply_pending_balance (crank); sweep_stealth_account (hard: discrete-log ownership proof for a never-signing account); close_stealth_account; batch_payout (ALTs; proof context-state accounts); set_auditor_key.
Off-chain: Rust/TS scanner (view-tag filter + trial ECDH), ElGamal rederivation, sweep/consolidation, batch proof gen, CU/latency benchmarks.

## Red team
- Mostly plumbing except PDA-owner + ECDH-IKM + sweep proof. Don't oversell.
- Not redundant: confidential hides amount but keeps linkable address; sRFC 42 leaves amount public.
- Leaks: (1) announcement + fresh PDA account + Configure + Deposit adjacent in time = timing correlator; (2) `Deposit` shields a *public* balance — so a new stealth account must receive via confidential→confidential Transfer, i.e. the sender must already hold a confidential balance (its own treasury conversion is a public, amount-revealing event, decoupled in time).
- Batch payroll cost UNVERIFIED: each Transfer needs equality + ciphertext-validity + range proofs, likely context-state accounts (create→verify→close) → several txs per recipient; 500-person payroll could be thousands of txs. Benchmark first.
- Rent: each PDA + confidential-extension ATA is rent-heavy; thousands of one-time accounts bleed SOL; needs permissionless close crank. Privacy-vs-cost tension vs Rings/CSPL.
- Regulatory: auditor key reveals amount, not beneficiary identity → Travel Rule not satisfied for pseudonymous B2B; fine for payroll (employer knows employee).
- sRFC 42 momentum is the weakest link: dormant 3+ months, not mainnet, unaudited.
- Wallets: Phantom confirmed no confidential-balance support; others unverified.

## Kill criteria
sRFC 42 on mainnet → no. Author engagement → no. Crux solvable → conditionally (soundness unverified). Existing combo shipped → none. Batch CU tractable → unverified/at risk. Rent tractable → at risk. Wallet support → no. Regulatory framing → none specific.

## Who cares
susruth (sRFC 42), SF confidential-balances/docs team, Helius Rings, Arcium, MagicBlock, Squads Altitude, Colosseum judges (Vanish precedent).
