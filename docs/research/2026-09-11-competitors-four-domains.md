# Solana RWA / Tokenization / Payments / Collectibles — Competitive Landscape (Sept 2026)

*Numbers from DefiLlama, press releases, project docs, trade press as linked. "(unverified)" where no primary source.*

## 1. Real World Assets

**Tokenized commodities (lithium/rare earth/uranium).** No mature Solana-native project. **xU3O8** (uranium, Cameco-custodied) on Tezos/Etherlink ~$8.5M mcap (https://chain.link/article/tokenized-uranium-rwa); **Raredex.io** (rare earths) on Arbitrum. No lithium token anywhere. Solana white space, but underlying model (custodian + attested vault receipt) is unremarkable engineering.

**Carry-trade/sovereign debt.** **Etherfuse** (Solana CETES) small TVL (~MXN 65.4M), composable with Orca/Kamino/Jupiter, Mexico regulatory sandbox (https://solanacompass.com/projects/etherfuse). **Ondo** $3.5B protocol TVL, Solana minority; USDY ~$2.15B, OUSG ~$370M (https://defillama.com/protocol/ondo-finance). **OpenEden TBILL** ~$115–247M (unverified).

**Private credit.** **Maple** $2.9B TVL mostly Ethereum; **Credix** Solana-only ~$13.5M TVL, CESCE/Munich Re insurance on LatAm receivables; **Collaterize** generic launchpad, no verifiable traction (https://crypto.news/collaterize-launches-rwa-tokenization-launchpad-on-solana/); **Huma** $17B+ cumulative PayFi volume, $38M raise; **Centrifuge** on Solana since May 2025; **Goldfinch** wound down (~$1.6M TVL).

**Real estate/casks.** **Parcl** synthetic perps, TVL collapsed ~$1.1M. **Homebase** real legal-wrapper engineering (Metaplex token with KYC/freeze/reissue, Reg D lockup) for one $246,800 deal. **Baxus** whisky, $5M seed, >$20M cumulative trades, molecular-tag auth pilot.

**RWA gaps:** (1) oracle pricing for illiquid physical assets — technical + market-structure. (2) settlement speed vs TradFi redemption cycles — plumbing + custodian-bound. (3) on-chain compliance — increasingly solved via transfer hooks. (4) legal-wrapper binding — regulatory. (5) secondary liquidity — distribution.

## 2. Tokenization

**Equities.** **xStocks/Backed** $35B+ cumulative, ~95% Solana DEX share, thin bearer tokens, Robinhood Chain overtaking daily volume (https://genfinity.io/2026/07/06/tokenized-stocks-on-solana-explode-past-5-77b-q2-2026/). **Superstate** $1B+ AUM, Parallel Markets KYC, Solana minor (USCC $4.4M). **Securitize (BUIDL)** ~$550–615M on Solana, transfer-agent-of-record, public via SPAC (NYSE: SECZ). **Franklin Templeton Benji** ~$726–828M, SEC no-action letter. **Ondo Global Markets** $1B TVL fastest ever, Solana leg ~$248M, Token-2022 transfer-hook eligibility enforcement (https://www.coindesk.com/business/2026/01/21/ondo-finance-brings-200-tokenized-u-s-stocks-and-etfs-to-solana). **Remora, Jarsy (Base), Republic** thin wrappers.

**Commodity vaults with PoR.** **PAXG** on Solana is LayerZero-wrapped, monthly KPMG attestation. **XAUT** >$3.3B mcap, quarterly attestation. **Chainlink PoR on Solana** 165 integrations, periodic not continuous (https://chain.link/blog/quarterly-review-q2-2026).

**SPV/PE.** **Libre Capital** (Nomura/Brevan Howard) token-extension eligibility gating for feeder funds (Hamilton Lane SCOPE $556M). **Sologenic** XRPL-native.

**Compliance tooling.** Transfer hooks are the Solana-specific differentiator; "RWA Token Program" emerging (https://www.quillaudits.com/research/rwa-development/non-evm-standards/solana-rwa-token-program). **SAS** mainnet 2026, very early. **Bridge (Stripe)** orchestration, may route to Tempo L1.

**Tokenization gaps:** (1) atomic settlement between off-chain registries and tokens — technical + regulatory. (2) real-time PoR vs periodic — technical, solvable. (3) transfer-hook composability with DeFi — technical. (4) cross-chain whitelist portability — early. (5) secondary liquidity for SPV/PE — regulatory.

## 3. Payments

**Regional stablecoins.** **Etherfuse MXNe** real bond collateral, thin liquidity. **BRZ** negligible. **Turkish lira** stablecoins huge ($3.4B via Zodia 2025) but not Solana-native. **cNGN** SEC-regulated Nigeria, trivial circulation.

**Merchant POS.** **Helio** acquired by MoonPay $175M, $1.5B+ processed, 6,000+ merchants. **Sphere** $38B+ committed volume, SphereNet private pilot. **Sling Money** $20M, 150+ countries. **Squads Altitude** $18M, $200M+ processed. **Slash** $100M Series C. **Solana Pay** thin QR/link/NFC spec, ~700 merchants via Cryptwerk.

**Offline/mesh/state channels — thin on Solana.** No Solana-native BLE-mesh/offline project found. Bitcoin **LNMesh** (academic), Ethereum **MeshWallet** (BLE relay, online settlement) both punt on offline double-spend (https://arxiv.org/abs/2304.14559). **Solana Private Channels / x402 payment channels** are custodial-escrow batching, not trustless bilateral channels. "1M payments/sec" x402 benchmark is voucher-signing throughput; actual daily x402 volume ~$28K, ~half wash (https://cryptoslate.com/solanas-million-payments-a-second-ai-system-can-leave-sellers-unpaid-even-after-they-deliver/).

**Confidential transfers.** Primitive live, no production payroll found — **Zebec** ($500M+ annual payroll) and **Streamflow** ($1.4B TVL) use transparent SPL.

**Payments gaps:** (1) true offline finality — technical, unsolved anywhere. (2) confidential-transfer usability/integration layer — technical→distribution. (3) cross-border FX for local stables — distribution. (4) merchant chargeback/dispute — technical gap with distribution consequence. (5) trustless micropayment channels — technical + unproven demand.

## 4. Collectibles

**Phygitals.** **Collector Crypt** $1.6B lifetime, 130K+ cards, ~22K users, CARDS token, redemption = Metaplex burn + off-chain backend, no published on-chain escrow state machine (https://solanacompass.com/news/collector-crypt-crosses-16-billion-in-volume-as-solanas-tokenized-card-market-builds-scale). **Phygitals** $250M+, self-funded, compressed NFTs, Fanatics Collect partner. **Courtyard.io** Polygon, $37M raised, >$1B volume. **Baxus** whisky. "Vaulted"/"Nukleus" unconfirmed. **ORO** gold, Brink's custody.

**Provenance.** **Arianee** ($30.6M, 2.2M products) and **Aura** (LVMH) Ethereum-permissioned; brand is sole attestor. No ZK physical-authenticity project shipping anywhere.

**Metaplex.** **MPL-Hybrid** (NFT↔fungible escrow swap, SPL-404 lineage) — most technically substantive primitive found; scoped to Core NFTs + SPL, not physical-redemption semantics (https://www.metaplex.com/docs/smart-contracts/mpl-hybrid/faq).

**Ticketing.** **XP** (Captain Labs) $6.2M seed, 34M tickets claimed, anti-scalping logic roadmap only.

**Collectibles gaps:** (1) trustless physical-custody ↔ token linkage — buildable, unbuilt. (2) **no standardized redemption/burn state machine** — every project rolls own; MPL-Hybrid closest — technical, solvable. (3) fractional ownership vs single physical redemption — needs buyout/governance mechanism — technical + regulatory. (4) provenance without trusted brand/vault — technical in theory. (5) ticketing resale — distribution.

## Cross-domain takeaway
Primary-issuance/compliance tech is where real engineering exists and is incomplete (transfer hooks, MPL-Hybrid, Chainlink PoR, SAS); everything downstream — redemption, custody proof, cross-border settlement, offline finality, secondary liquidity — is either unbuilt (technical gap) or blocked by licensing/custody/habit. Clearest pure-technical white spaces: (a) standardized on-chain redemption/escrow state machine for physical-backed tokens, (b) production-grade confidential-transfer tooling for payroll, (c) attested proof-of-custody for vaulted phygitals/RWA.
