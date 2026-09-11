# Solana Hackathon & Turbin3 Landscape: RWA, Tokenization, Payments, Collectibles (2024–2026)

*Research compiled 2026-09-11 via web search. Where a claim could not be independently confirmed, it is marked "unverified."*

## 1. Colosseum Hackathons

### Renaissance (2024) — 9th Solana Foundation hackathon, 1st run by Colosseum
8,300+ participants, 1,071 projects, 95+ countries.

- **Grand Champion: Ore** ($50k) | Payments/Infra | Novel PoW mining algorithm on Solana, mines a fungible token via onchain hash-puzzle solved every 60s | tech: custom compute-budget-constrained hashing, no ASIC advantage by design | https://solana.com/news/solana-renaissance-winners
- **Urani** (DeFi/Payments track win, $30k) | intent-based swap aggregator with MEV protection
- **High TPS Solana Client** (Infra track win, $30k) | client-level scheduling/pipeline optimization for validator throughput — genuine systems engineering
- **Nomad / Ripe** (DeFi & Payments finalists) | Nigeria off-ramp and SE-Asia QR merchant payments — early instances of the "remittance app" pattern that later saturates
- Source: https://blog.colosseum.com/announcing-the-winners-of-the-solana-renaissance-hackathon/

### Radar (2024) — 10th hackathon, 2nd by Colosseum
10,000+ participants, 1,359 projects, 120+ countries.

- **Grand Champion: Reflect** ($50k) | Stablecoins/DeFi | hedge-backed delta-neutral synthetic dollar plus LST-yield pass-through; later "stablecoin-as-a-service" infra, raised $3.75M seed (a16z CSX, Sept 2025) | https://blockworks.co/news/solana-based-reflect-wins-radar-hackathon
- **FXSwap** (Payments track win, $30k) | forex↔stablecoin swap protocol
- **Hylo** (Payments finalist) | dual-token stablecoin backed by LSTs
- **Supersize** (Gaming track win, $30k) | fully onchain multiplayer game on **MagicBlock ephemeral rollups**
- **Genesis** (Consumer finalist) | fractional/tokenized IP investing
- Source: https://solana.com/news/solana-radar-winners

### Breakout (2025) — 11th hackathon, 3rd by Colosseum, April 14–May 16 2025
10,000+ participants, 1,412 projects, 140+ countries. First hackathon with dedicated **Stablecoins** and **AI** tracks.

- **Grand Champion: TapeDrive** ($50k) | Infra/Storage | decentralized object store anchored on Solana; erasure-coded data plane + onchain stake-weighted control plane; storage-proof challenges every minute; ~1,400x cheaper than onchain storage | https://blockworks.com/news/solana-data-startup-wins-hackathon , https://github.com/spool-labs/tape
- **Vanish** (DeFi track win, $25k) | Payments/Privacy | privacy via smart routing through protected liquidity, explicitly *not* Token-2022 Confidential Transfers; raised $1M pre-seed
- **CargoBill** (Stablecoins track win, $25k) | non-custodial multisig business wallets for B2B logistics payments; Colosseum flags "B2B stablecoin remittance for logistics SMEs" as *saturated*
- **FluxRPC** (Infra track win, $25k) | RPC fully separated from validator layer
- **Latinum** (AI track win, $25k) | payment middleware for MCP-server monetization, precursor to x402 wave
- **LootGo** (Mobile/Seeker Award, $25k)
- Source: https://blog.colosseum.com/announcing-the-winners-of-the-solana-breakout-hackathon/

### Cypherpunk (late 2025) — 12th hackathon, 4th by Colosseum
9,000+ participants, 1,576 projects, 150+ countries. First standalone **RWA** track.

- **Grand Champion: Unruggable** ($30k) | Solana-exclusive hardware wallet — moat is manufacturing/distribution
- **Autonom** (RWA track win, $25k) | oracle handling corporate actions (splits, dividends, mergers) for onchain equity pricing — real technical differentiator | https://blog.colosseum.com/announcing-the-winners-of-the-solana-cypherpunk-hackathon/
- **Seer** (Infra track win, $25k) | transaction debugging/dev-tooling
- **MCPay** (Stablecoins track win, $25k) | payment infra connecting MCP + x402
- **Corbits** (Infra runner-up) | open-source x402 endpoint dashboard
- **Bore.fi / Pencil Finance / Watchtower** (RWA finalists) | tokenized SME private equity, onchain student loans, space-infra financing
- **attn.markets** (Undefined track win) | revenue tokenization protocol
- Sponsors included Arcium (confidential computing); no winner confirmed built on it (unverified).

### Frontier (2026) — 13th hackathon, 5th by Colosseum, April 6–May 11 2026
10,000+ builders, 2,857 submissions, 150+ countries. 26 winners.

- **Grand Champion: CrowdBrain** ($30k, paid in Phantom's CASH) | DePIN | robotics teleoperation DePIN
- **Housd** | RWA | German real-estate-debt tokenization; admitted to Colosseum accelerator cohort 5
- **ODL** | RWA | liquidation marketplace for discounted PE/private credit secondaries
- **Crafts** | Tokenization | "Stakeholder Token Offerings"
- **JK Index / One Arena / Traded.gg** | Collectibles | TCG marketplace + grading intelligence; gacha-pack game with PSA-graded cards + onchain randomness; aggregation/execution layer for collectible markets (Collector Crypt: 130k+ cards, $1.6B volume; Phygitals: $180M+ volume; both vault-authenticate-tokenize-redeem)
- **DashX / KinnectFi / Stablecorp** | Payments | cross-border stablecoin infra, Philippine-diaspora neobank, India stablecoin business infra
- Source: https://blog.colosseum.com/announcing-the-winners-of-the-solana-frontier-hackathon/ , https://solanacompass.com/news/colosseum-announces-26-winners-of-the-solana-frontier-hackathon-the-largest-crypto-hackathon-ever

### Colosseum Eternal (rolling, 2025–ongoing)
Perpetual submission program with semi-annual "Eternal Award", feeds accelerator. No standalone track-winner list found (unverified).

## 2. Side bounties
- **Superteam Earn**: mostly content/research bounties in RWA (e.g. "Deep Dive of the State of RWAs on Solana", Perena-sponsored content). No engineering-heavy product bounty winners found for these domains. https://superteam.fun/earn/listing/deep-dive-of-the-state-of-rwas-on-solana
- **Solana Speedrun**: game jam, out of scope.
- **Solana x402 Hackathon** (Nov 2025, 400+ submissions): **Intelligence Cubed** (AI-model marketplace paying per inference via x402), **PlaiPin** (ESP32 with own wallet, M2M payments), **x402 Triton Gateway** (pay-per-query historical ledger via Old Faithful), **Sentinel Agent**, **Galaksio**. Most technically dense payments cluster — agent micropayment infra, not remittance UX. https://www.bitget.com/news/detail/12560605084981

## 3. Turbin3
GitHub org `solana-turbin3` (637+ repos, individual coursework). Capstones are pass/fail deliverables, not prize-ranked. Quarterly cohorts, "Bridge to Turbin3" bootcamp. Curriculum: architecture, Anchor, Token Programs/Extensions, RWAs, Payment Infra, Privacy.

Capstone deliverables: TS prereq, Rust prereq, NFT module, **Capstone LOI**, then capstone; deploy to devnet (some mainnet), defend live. Demo Day: 65+ builders over multiple days.

Example capstones: customizable-vault platform (`Q3T_Sol_kox`), freelancer trust marketplace (`0xkinley/turbin3`), **BloodLedger**, **Payclip** (USDC payments), escrow/AMM/NFT-staking programs, MagicBlock-based projects (Q3 2025). No confirmed funding follow-ons (unverified). Post-Builders: "Artisan Cohort".

## 4. ~10 projects where engineering was the differentiator
1. TapeDrive — erasure-coded storage + onchain proof-of-storage
2. Ore — compute-budget-constrained PoW primitive
3. FluxRPC — RPC decoupled from validator
4. Seer — tx debugging / CPI traces
5. Autonom — corporate-action-aware oracle for RWA
6. Supersize — MagicBlock ephemeral rollups realtime state
7. High TPS Solana Client — validator scheduling
8. CrowdBrain — robotics DePIN
9. Txtx — runbook devex platform
10. x402 Triton Gateway / Old Faithful — historical ledger plumbing

Contrast (distribution-bound): Unruggable (hardware/retail), CargoBill (enterprise sales, saturated niche), Vanish (compliance positioning, avoided confidential transfers), Reflect (BD/integrations), and every "stablecoin for [emerging market] diaspora" entry (Nomad, Ripe, LocalPay, Xelio, Paytos, Credible, KinnectFi, Stablecorp, DashX) — near-identical tech, regional distribution wins.

## Patterns
**Judges reward**: (1) infra fixing a Solana-specific pain (storage cost, RPC centralization, tx debuggability, RWA oracle blind spots); (2) new financial primitives with defensible mechanics; (3) consumer products with realtime state loops; (4) agentic payments (x402/MCP) since Breakout.

**Saturated**: stablecoin remittance/off-ramp for emerging markets (every cycle since Renaissance); generic AI trading agents; "tokenize X" that stops at a mint with no oracle/compliance/redemption (Autonom stood out precisely by solving the oracle problem).

**Under-used primitives among winners**: Token-2022 **Confidential Transfers** (Vanish bypassed it); **Transfer hooks** for RWA compliance (no winner built around them); **ZK compression / cNFTs** for high-volume phygital tokenization (none cite it); **Arcium MPC** (sponsor, no winner); **Solana Pay / Blinks** (near-absent from payments track).
