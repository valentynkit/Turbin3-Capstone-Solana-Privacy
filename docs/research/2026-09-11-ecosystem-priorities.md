# Solana Ecosystem Research: What's Wanted, What's New (September 2026)

## 1. Solana Foundation / Ecosystem Stated Priorities

**Foundation grants**: milestone-based grants, convertible grants, topic RFPs. Live RFPs (forum.solana.com/c/rfp/10) skew infra/tooling: Test Validator Plugin Framework, Generalized State Compression, Program Verification/Pre-Deployment Analysis, Post-Deployment Monitoring, Alternative Archival Storage, Indexer Tooling, Discriminator Database, and a long-standing **"Unified Security Token/RWA Program"** RFP (open since Oct 2023, still listed). https://forum.solana.com/c/rfp/10

**Colosseum "Request for Startups"** (solana.com/solutions/request-for-startups):
- Stablecoins/Payments/PayFi: "End-to-End Payments" bypassing 3%+ card fees; "Buy Now, Pay Never" (yield-funded subscriptions); P2P marketplace escrow to kill scams/disputes.
- Asset Tokenization: "RWA Aggregated Investment Portal".
- Infrastructure: "Onchain Identity" (privacy-preserving tokenized ID), "Light RPC".

**Superteam 2026**: Superteam USA prioritizing consumer and agentic products; microgrants $200–$10k, emerging markets. https://www.forbes.com/sites/hadleystern/2026/03/12/nicky-scannella-building-solanas-superteam-usa/

**Leadership narrative**: "Internet Capital Markets" (ICM) — decentralized NASDAQ: price discovery, embedded compliance (freeze/confiscate/allowlist/confidential-balance in Token-2022), institutional rails. JPM, State Street, Citi, Franklin Templeton, Visa, PayPal, Western Union POCs/production. RWA ~$5.4B→$19.3B Jan 2025→Mar 2026 (unverified precision); tokenized-equity DEX volume ~95–97% Solana share mid-2026. https://solana.com/news/breakpoint-2026-london-speakers , https://www.odaily.news/en/post/5211602

## 2. Technical Primitives and Maturity (Sept 2026)

| Primitive | Status |
|---|---|
| Token-2022 confidential balances | Live on mainnet, re-enabled epoch 982 (~mid 2026) after ZK ElGamal audit disable. Helius "Rings" devnet-only. https://solana.com/docs/tokens/extensions/confidential-transfer |
| Transfer hooks | Live; composability with DEXs still broken in practice (§3). |
| Scaled UI Amount | Live; multiplier rebasing for splits/dividends without per-holder mint/burn. Cannot combine with InterestBearing. https://solana.com/docs/tokens/extensions/scaled-ui-amount |
| Pausable / Default Account State / Permanent Delegate | Live; MiCA-compliance toolkit (MiCA fully applies July 1 2026). Also abused by rug tokens. |
| ZK compression (Light Protocol) | Mature, ~90M+ compressed accounts; **Helius acquired Light June 2026**, kept OSS. "Light Token Program" (200x cheaper) devnet; mainnet targeted Q1 2026, production still on legacy Compressed Tokens. https://lightprotocol.com/about/ |
| MagicBlock ephemeral rollups | Mainnet, 4 regions, sub-50ms, 1B+ txns. Private ERs (Intel TDX) shipped Private Payments API to mainnet March 2026. https://www.magicblock.xyz/ |
| Solana Attestation Service (SAS) | Live; partners Civic, Sumsub. Reusable KYC passports, jurisdiction gating, registries. https://solana.com/news/solana-attestation-service |
| Solana Permissioned Environments | Live; private SVM instances for banks. https://www.helius.dev/blog/solana-permissioned-blockchains |
| Pinocchio / Codama / Surfpool | Official 2026 dev stack: Anchor 1.1.x or Pinocchio 0.11+, Codama codegen, Surfpool + LiteSVM/Mollusk. https://www.helius.dev/blog/pinocchio |
| Alpenglow | Test cluster May 2026; VAT/BLS SIMDs mainnet July 2026; Transaction V1 activated Sept 9 2026; full consensus mainnet targeted Oct 2026 (100–150ms finality). https://www.coindesk.com/tech/2026/05/11/the-biggest-consensus-overhaul-in-solana-history-is-officially-live-for-testing |
| Firedancer | Full client on mainnet since Dec 2025; ~20%+ validators Q2 2026. |
| Solana Mobile Seeker | Shipped Aug 4 2026, 150K+ preorders; Seed Vault, TEEPIN attestation. |
| Solana Pay / Blinks / Actions | Solana Pay major stablecoin rail (Visa, PYUSD, Gusto payroll pilot). Blinks gated by wallet extensions. **NFC/offline tap-to-pay remains an open GitHub feature request** https://github.com/solana-labs/solana-pay/issues/57 |
| Squads smart accounts / passkeys | $15B+ secured, 40K+ smart accounts; WebAuthn passkeys + secp256k1 signers. |
| Winternitz vaults | Dean Little's WOTS/Keccak vaults; opt-in, single-use per vault; ~zero adoption. |
| xStocks / tokenized equities | Backed-issued SPL tokens, $25B+ cumulative volume, ~95–97% of tokenized-equity DEX volume. Ondo Global Markets leads TVL. Chainlink PoR. https://blog.kraken.com/product/xstocks/25-billion-in-total-transaction-volume |
| sRFCs | Moved to GitHub Discussions `solana-foundation/SRFCs`. Active: sRFC-37 Token ACL Standard (permissioned-token UX, Q1 2026), sRFC-39 Clear Signing, "Solana Agent Protocol". https://github.com/solana-foundation/SRFCs/discussions |

## 3. Open Problems / Developer Pain Points
- **Transfer-hook composability broken for DEXs**: accounts passed into hook become read-only, signer privileges don't carry, hook can't take same-token fee; routers avoid hooked tokens. HUKT hook registry emerging. https://chainstack.com/solana-transfer-hooks-anchor-token-2022/
- **RWA compliance has no unified governance model**: "genuine onchain governance for tokenized securities at scale remains an open problem" (RedStone Tokenization Report Mar 2026). https://blog.redstone.finance/2026/03/26/tokenization-rwa-report-2026/
- **cNFT/compressed-state discoverability**: naive account scans miss compressed assets; indexer dependence.
- **Rent/dust accounts**: 2019-era complaint still manifests (GitHub #6864, #7413).
- **Offline/NFC payments**: Solana Pay #57, no native NFC flow.
- **Blink distribution capped** by wallet-extension gatekeeping.
- **ZK-compression indexer centralization** (Helius-class providers).
- **Quantum-resistant vaults** opt-in, ~zero adoption.

## 4. Unbuilt gaps
(a) hook-aware DEX/router compatibility layer for compliant RWA tokens; (b) RWA aggregation/discovery portal (Foundation asked explicitly); (c) NFC/offline Solana Pay; (d) unified security-token/RWA program (open RFP since Oct 2023); (e) phygital redemption/custody standardization — Phygitals, Collector Crypt, BAXUS each reinvent vault-attestation-redemption with no shared on-chain standard despite $1.6B+ combined volume.

## Under-Exploited Primitive x Domain Matrix

| Primitive | RWA | Tok | Pay | Coll | Concrete idea |
|---|---|---|---|---|---|
| Transfer hooks + hook registry | ✅ | ✅ | — | — | Hook-aware router adapter SDK so Jupiter/Raydium-class routers resolve extra accounts for any registered hook |
| Scaled UI Amount | ✅ | ✅ | — | — | Rebasing treasury/dividend distributor via multiplier update, no mint/burn |
| SAS | ✅ | ✅ | ✅ | ✅ | "Phygital redemption passport" attestation reusable across vault platforms |
| ZK compression / Light Token | ✅ | — | ✅ | ✅ | Compressed-token loyalty / fractionalized-collectible distribution at millions of holders |
| MagicBlock Private ERs | ✅ | — | ✅ | — | Confidential settlement for institutional RWA trades |
| Solana Pay + NFC | — | — | ✅ | — | Tap-to-pay terminal SDK bridging solana: URIs to NFC/HCE |
| RWA Token Program (ACL/Transfer Restrictions/Tokenlock/Dividends) | ✅ | ✅ | — | — | Governance module for tokenized securities (proxy voting/consent) |
| Confidential Balances | ✅ | — | ✅ | — | Payroll/B2B with private amounts + permanent delegate compliance |
| Squads + passkeys | — | — | ✅ | ✅ | Passkey checkout + session-key spending limits for collectibles |
| Winternitz vaults | ✅ | — | — | — | Quantum-resistant cold storage plugin for RWA reserve custodians |

**Overall read**: highest-leverage, least-crowded intersection is **transfer-hook composability for compliant RWA tokens** (multi-year open RFP, documented DEX-breaking bug class, blocks ICM priority). Second: **phygital redemption/attestation standardization**.
