# Turbin3 Capstone Research Brief (2026-09-11)

*Turbin3 publishes no detailed public syllabus; reconstructed from GitHub metadata, student posts, secondhand mentions.*

## 1. Program structure
Turbin3 (turbin3.org, ex Web3 Builders Alliance, since 2022), Solana Foundation-backed. Tracks:
- **Builders Cohort**: 6–7 week intensive, 3 live sessions/week, ~5:1 ratio. Curriculum: Architecture → Anchor → Token Programs/Extensions → RWAs → Payment Infra → Privacy → capstone. https://turbin3.org/institute
- **Advanced SVM Cohort**: runtime internals (syscalls, sysvars, ZK/rollups). https://github.com/Turbin3/ADV-Runtime
- **Accelerated Program** (`accel-*` repos): post-Builders, more ambitious capstones.
- **Bridge to Turbin3**, **Enterprise Training**, **Developer Placement**.

Capstone milestones: **LOI → user stories → architecture diagram → Anchor program → frontend → Demo Day** (live demo, devnet deploy, sometimes mainnet). No public grading rubric found. Demo Day March 2026: 65+ builders over 3 days. https://x.com/solanaturbine/status/2029153436403679515

## 2. Past capstones
Builders Cohort (CRUD-to-moderate): CrowdFi (crowdfunding), StakeMyScore, OmniTrack, Solana Minesweeper, Sagent, Multisig Vault (https://github.com/kunalsinghdadhwal/turbin3-capstone), Gamified SRS, AirPay, SolanaPulse, DeResHub, Solana Watcher.

Accelerated-track (`Turbin3` org, 2025-2026) — closest to "grades well":
- **Cardon** — on-chain tx firewall for AI agents (policy in Anchor, Helius simulation pre-sign, multisig policy authority). https://cardon-protocol.vercel.app/
- **StealthX** — private gasless MPP for AI agents. https://github.com/Turbin3/accel-StealthX
- **Giuseppe** — zero-copy order book on MagicBlock ERs. https://github.com/Turbin3/accel-Giuseppe
- **FluxDEX** — hybrid CLOB + PMM. https://github.com/Turbin3/accel-FluxDEX
- **Latch** — PvP wagering with trustless escrow. **Intentra** — intent protocol. **Lending Orderbook**. **Enclave** — private AMM. **Mojo** — account-factory SDK with Pinocchio/bytemuck (no Anchor). https://github.com/Turbin3/accel-Mojo
- **P-ATA, pinocchio-multisig, pinocchio-stake-program, pinocchio-tapedrive** — Pinocchio infra.
- **Poseidon** (Turbin3-maintained TS→Anchor transpiler, 154 stars) — org's own bar for useful infra.

Outcomes: Priyansh Patel — SF grantee for SolixDB (indexing). Turbin3 claims "400+ builders, 10 winners/honorable mentions in Colosseum". No named capstone → Colosseum accelerator confirmed (unverified).

**Pattern**: CRUD capstones dominate Builders; technically heavy (order books, private AMMs, ER matching, policy engines, Pinocchio) cluster in Accelerated/Advanced SVM.

## 3. Leadership signals
Jeff Paul (founder), Colosseum panels. Ethos from student accounts: "doesn't hold your hand — read the source, break things". Capstone framed as "useful product that brings Solana to the masses". "Build, deploy, defend an MVP." No verbatim "no more todo apps" quote found. Andreia Canadas: GitHub bio only. "Berg": Valid8 team (https://github.com/Turbin3/valid8).

## 4. The four 2026 categories
No public Turbin3 post naming RWA/Tokenization/Payments/Collectibles as 2026 focus; no Collaterize/Collector Crypt/Phygitals partnership found — treat the brief as authoritative internal info. Collector Crypt + Phygitals partnered with Jupiter for card-collateralized lending (https://x.com/JupiterExchange/status/2061482091561288051). "Collaterize" exists as an RWA launchpad on Solana (see competitors file).

## 5. Public LOI examples
Only as PDFs in student repos, e.g. `Capstone_Letter_of_Intent(LOI).pdf` in https://github.com/solana-turbin3/Q1_25_Builder_brianobot — download via raw link to read.

## Bottom line
Deliverable spine: LOI → user stories → architecture diagram → deployed Anchor program → frontend → Demo Day. "Praised" bar: real problem with real user; devnet+; in advanced tracks "heavy" = SVM-runtime, Pinocchio, ERs, order books, private AMMs, policy engines.
