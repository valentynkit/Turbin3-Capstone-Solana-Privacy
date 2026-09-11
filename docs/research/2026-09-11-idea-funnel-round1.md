# Capstone ideas — Round 1 (wide net), 2026-09-11

Scoring 1–5 each: **H** hardness (engineer-wow) · **E** ecosystem pull (RFP/sRFC/named consumer) · **D** fit with the 4 brief domains · **B** backend-only (no frontend taste work) · **R** delivery risk is low · **T** wins on tech, not distribution. Max 30.

Constraints from Valentyn: complexity is not a constraint (AI-parallelised build), avoid frontend-heavy, team of 2–3, any primitive, any domain, open source, must impress SF/hardcore devs and be useful to the ecosystem.

## Tokenization / RWA

| ID | Idea | H | E | D | B | R | T | Σ | Evidence |
|---|---|---|---|---|---|---|---|---|---|
| T1 | **Permissioned-token liquidity layer**: exchange (CLOB fork of Manifest, or RFQ) native to Token ACL (sRFC 37) *and* transfer hooks; solves "protocol PDA as compliant holder" (vault/escrow thaw), router adapter for Jupiter-class integrators, adversarial suite for hook CPI-depth + heap OOM (token-2022 #66) | 5 | 5 | 5 | 5 | 3 | 5 | **28** | SF RFP "Unified Security Token/RWA Program" open since 2023; ACL thread + sRFC 43 both flag PDA-holder as open; protocols blacklist hooked mints |
| T2 | **RWA security-token engine** (sRFC 00020 reference impl): corporate actions state machine — splits via ScaledUiAmount, dividends/coupons via crankless ZK-compressed epochs, clawback, halts, cap-table event log mapped to transfer-agent obligations | 4 | 5 | 5 | 5 | 4 | 5 | **28** | 00020 stalled; each issuer (Superstate, Galaxy) hand-rolls; Cypherpunk RWA winner Autonom was the oracle half of this |
| T7 | Bond coupon scheduler + ScaledUiAmount wallet-discovery sRFC (subset of T2) | 3 | 4 | 5 | 5 | 4 | 5 | 26 | Docs: "no standardized mechanism for wallets to discover multiplier" |
| T4 | RFQ settlement primitive for illiquid permissioned RWAs (signed maker quotes, atomic settle, compliance check) | 4 | 3 | 5 | 5 | 4 | 3 | 24 | Dragonfly: CLOBs fail for long-tail RWA |
| T3 | Multi-notary zkTLS proof-of-reserve oracle → SAS attestation for vaulted commodities | 5 | 3 | 5 | 4 | 2 | 4 | 23 | TLSNotary: designated-verifier only; Chainlink PoR periodic |
| T5 | Confidential SPV cap table (confidential balances + permanent delegate + SAS eligibility) | 4 | 3 | 5 | 4 | 3 | 3 | 22 | Libre/Securitize gate but expose balances |
| T6 | Invoice-factoring Dutch auction + compressed invoice registry | 3 | 2 | 5 | 4 | 4 | 2 | 20 | Credix/Huma own this; distribution-bound |

## Payments

| ID | Idea | H | E | D | B | R | T | Σ | Evidence |
|---|---|---|---|---|---|---|---|---|---|
| P1 | **Trustless bilateral payment channels** with challenge windows, partial settlement, proof-of-delivery hooks, watchtower; griefing test suite | 5 | 5 | 5 | 5 | 3 | 5 | **28** | SF "Payment Channels" shipped 2026-09-03; CryptoSlate: dispute/refund gap unresolved; x402 channels are custodial batching |
| P2 | **Silent payments × confidential balances**: sRFC 42 stealth addresses + Token-2022 confidential transfers → unlinkable *and* amount-hidden payroll/invoicing with auditor key; scanning + sweep tooling | 5 | 4 | 5 | 4 | 3 | 5 | **26** | Neither combined anywhere; brief lists "Privacy" payments; confidential balances re-enabled ~mid-2026, usage ~0 |
| P6 | sRFC 43 Held Delivery + solving the ACL guard-vault thaw open question, for PSP settlement accounts | 3 | 4 | 5 | 5 | 4 | 4 | 25 | Ref impl exists (less novel) |
| P4 | Confidential payroll engine: batch ElGamal proofs, auditor rotation, mobile benchmarks, Go/Python proof bridge | 4 | 4 | 5 | 4 | 3 | 4 | 24 | solana-go #412 open; only toy `zalary` |
| P8 | Post-quantum multi-spend vault (XMSS tree over Winternitz), CU-golfed | 5 | 3 | 2 | 5 | 4 | 5 | 24 | Dean Little's vault is single-use; weak domain fit |
| P3 | Offline BLE mesh payments: pre-signed durable-nonce chains, double-spend formalisation, Seeker Seed Vault | 5 | 4 | 5 | 2 | 2 | 4 | 22 | Brief names it; nothing credible exists; needs mobile app + hardware |
| P7 | Merchant escrow / dispute-chargeback state machine + carrier attestation | 3 | 3 | 5 | 4 | 4 | 3 | 22 | Colosseum RFS "P2P marketplace escrow" |
| P5 | NFC tap-to-pay: secure-element ed25519 applet + Solana Pay #57 | 4 | 4 | 5 | 2 | 2 | 4 | 21 | Hardware-bound |

## Collectibles

| ID | Idea | H | E | D | B | R | T | Σ | Evidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | **Vaulted-asset redemption standard**: MPL-Hybrid-style escrow + SAS custodian attestation + ship/dispute state machine; secondary trade without leaving vault | 3 | 4 | 5 | 4 | 4 | 4 | 24 | Collector Crypt ($1.6B), Phygitals ($250M), BAXUS each roll their own burn+backend |
| C2 | Fractional phygital + buyout auction reconciling 404-style fractions with single physical redemption | 4 | 3 | 5 | 4 | 4 | 3 | 23 | Unsolved conflict; securities-law edge |
| C4 | Crankless compressed-token royalty/reward distribution at scale | 3 | 3 | 4 | 5 | 4 | 4 | 23 | Light merkle-distributor is one-shot only |
| C3 | NFC-chip provenance: on-chain chip ECDSA verification (secp256k1/r1 precompile) + attestation graph | 4 | 3 | 5 | 3 | 3 | 4 | 22 | Arianee/Aura trust the brand; nothing on Solana |

## Round-1 verdict
Top tier (28): **T1, T2, P1**. Second (26): **P2, T7**. Natural pairings for a 3-dev team: T1+T2 (issue → trade one compliant asset end-to-end) or P1+P2 (private micropayment rail).

Recommendation for today's pitch: lead with **T1** (explicit multi-year SF RFP, two live sRFC threads flag the exact open problem, Jupiter/Raydium are named consumers, reuses AMM coursework, zero frontend needed). Bring **P2** as the crypto-heavy alternative and **P1** pending a read of SF's shipped Payment Channels design.

Round 2 = pick 3–4 → deep technical feasibility + adversarial red-team per idea → Round 3 = one winner → LOI draft (Valentyn writes Part A himself per AI policy; research files feed competitor section; red-team log becomes Part 2).
