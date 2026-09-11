# Hard Open Problems: Solana RWA / Tokenization / Payments / Collectibles
*Research 2026-09-11. Ranked by impressiveness-to-engineers × usefulness-to-ecosystem. No X API access; "someone should build" tweets unverified.*

## 1. Hook-aware AMM/CLOB standard, with jurisdiction whitelisting via SAS — RANK 1
No production-grade, audited, composable AMM standard for Token-2022 transfer hooks exists.
- Four independent zero-star hackathon repos attempt it (preeeetham/solana2022-AMM, mitgajera/Token2022-Hook-AMM, Vector-Protocol/vector-contracts, solanadevkit/TOKEN2022); no shared standard.
- https://github.com/solana-program/token-2022/issues/66 (joncinque): `invoke_transfer_checked` runs out of heap with >2 hooked transfers per instruction — hard resource-limit bug.
- Security writeup: CPI-depth exhaustion / reentrancy-style attacks specific to hook-integrated pools — malicious hook exhausts 4-level CPI stack so the AMM's post-transfer invariant check silently fails while hook side effects persist. https://dev.to/ohmygod/solanas-token-2022-transfer-hooks-how-a-safe-feature-imported-ethereums-deadliest-bug-class-16p6
- Liquidity fragmented to Raydium CLMM, Orca Whirlpools, Manifest (https://github.com/Bonasa-Tech/manifest); Manifest docs admit hooks make orderbooks less efficient.
- Competing standard: sRFC 37 "Token ACL" (delegated-freeze + gating program, lower overhead), mid-audit Jan 2026, Phantom integration unresolved. https://github.com/solana-foundation/SRFCs/discussions/2

Status: unsolved as a standard. Capstone (2-3 devs, 10-12 wks): reference hook-aware CLOB (fork Manifest's formally-verified core) with (a) hardened extra-account-meta resolution that doesn't heap-overflow, (b) post-CPI invariant-check pattern proven safe vs CPI-depth exhaustion, (c) compliance hook on SAS enforcing jurisdiction whitelists at pool level, benchmarked vs Token ACL. Who cares: Anza/token-2022 maintainers, Jupiter/Raydium, SF RWA vertical.

## 2. Confidential-transfer payroll / invoice settlement (ZK ElGamal)
ZK ElGamal Proof program re-enabled epoch 982 (~mid 2026) after year-long disable; Token-2022 redeployed with confidential ixs two weeks later. Usage "close to zero".
- SDK: Rust + JS/WASM only; Go open request https://github.com/solana-foundation/solana-go/issues/412.
- Auditor-key UX (sender/recipient/auditor ciphertext consistency proofs) has no polished client reference; only toy `ajanaku1/zalary`.
- No mobile-latency benchmark for proof gen.
- Anza exploration repo: https://github.com/solana-foundation/confidential-balances-exploration

Status: unblocked, unsolved as product. Capstone: payroll/invoicing reference with auditor-key rotation, batch-payout proof gen benchmarked on mobile, Go/Python proof-gen bridge.

## 3. Dispute/refund layer for Solana Payment Channels (x402/MPP)
SF launched "Payment Channels" 2026-09-03 with Alibaba Cloud; 1M payments/sec is a controlled 100k-wallet proxy test (https://cryptoslate.com/solanas-million-payments-a-second-ai-system-can-leave-sellers-unpaid-even-after-they-deliver/ , https://forkast.news/solana-payment-channels-hit-1-million-payments-per-second-but-who-is-actually-paying/). Unresolved: buyer halts mid-channel / operator goes dark → deposit recovery vs merchant's last bill; merchant unpaid after delivery. Classic dispute-window engineering on Solana's account model, not solved in shipped OSS reference.

Capstone: hardened channel-close/dispute module (challenge periods, partial settlement, proof-of-delivery hooks) on published x402/MPP SDK, with griefing-attack test suite.

## 4. Offline / mesh payments with delayed settlement
Only native primitive: durable nonces (offline signing, not settlement) https://solana.com/docs/core/transactions/durable-nonces. "Zypp Protocol" unverified. Stellar MeshPay (BLE/AWDL via MultipeerConnectivity) portable architecture but no Solana double-spend analysis: two conflicting durable-nonce txs signed offline against same nonce can propagate different mesh paths; nonce-advance ordering is the only tiebreaker. Durable nonces caused 2022 outage; possible future deprecation in SIMD discussion.

Capstone: formalize double-spend/ordering guarantees for durable-nonce txs over offline mesh, reference BLE relay with pre-signed nonce-advance chains + conflict rules, adversarial partition tests. Who cares: Solana Mobile (Seeker BT 5.4 + Seed Vault), EM payments.

## 5. Tokenized-equity corporate actions engine
Primitives in production: Permanent Delegate (forced transfer), Freeze Authority, Pausable — used by xStocks, Superstate, Galaxy Digital (32,374 tokenized GLXY shares). Missing: open engine composing splits, dividends/coupons, ticker changes, cap-table sync into one auditable state machine; each issuer hand-rolls vs transfer-agent's off-chain master file (SEC May 2025 staff FAQ permits blockchain-as-master-file).

Capstone: open corporate-actions state machine (event log → split/dividend/freeze ixs, canonical mapping to transfer-agent obligations).

## 6. Scaled UI Amount / coupon scheduling for tokenized bonds
Scaled UI Amount multiplier is "purely cosmetic"; docs: "no standardized mechanism for wallets and explorers to discover and apply" it. No reference implements a date-driven coupon schedule.

Capstone: on-chain scheduler driving ScaledUiAmount from coupon calendar + wallet/explorer discovery standard (sRFC candidate).

## 7. Crankless at-scale coupon/royalty distribution via ZK compression
Light `merkle-distributor` example proves compressed-PDA claims with vesting/clawback (https://github.com/Lightprotocol/examples-zk-compression); 1M-wallet airdrop ~$260k→~$50. Not built: recurring scheduled distribution with periodic re-proof against a changing holder tree without a centralized crank.

Capstone: periodic coupon runs with permissionless "anyone triggers next epoch", integrated with #6.

## 8. RFQ-based secondary market for illiquid RWAs
Dragonfly partner: CLOB liquidity only works for a few macro assets; long tail needs RFQ (https://www.bitget.com/news/detail/12560605421231). Manifest is mature CLOB (formally verified, https://manifest.trade/) but no open RFQ settlement layer for permissioned RWA instruments.

Capstone: on-chain RFQ primitive (quote-request, signed maker quote, atomic settlement + compliance hook).

## 9. Physical-asset proof-of-reserve via zkTLS
No Solana impl combining zkTLS/TLSNotary/Reclaim with vault attestation. TLSNotary June 2026: proofs are designated-verifier, not publicly verifiable (https://tlsnotary.org/blog/2026/06/17/public-verifiability/). Must design multi-notary quorum + SAS layer, honest about trust model.

## 10. Decentralized provenance graph for phygitals
Metaplex Core soulbound/attributes solved. Market (Phygitals $250M+, Collector Crypt $1.6B) runs on custodial vault databases (PSA/Alt/Fanatics). NFC/ISO 29167 tap-to-verify exists; no open reference wires it into on-chain provenance graph.

Capstone: SAS-based provenance chain (mint → custodian attestation → NFC tap-verify → transfer history).

## 11. Post-quantum vault beyond one-time use
Dean Little's Winternitz Vault (https://github.com/deanmlittle/solana-winternitz-vault): WOTS + truncated Keccak256, single-use (signing reveals ~50% key). No XMSS-style tree extension tuned to CU limits.

Capstone: XMSS-style hash-tree over Winternitz vault, CU-benchmarked per spend at tree depths.

### Ranking rationale
1–3 combine sharpest cited obstacles (documented OOM bug, reentrancy class, dispute gaps in just-shipped SF product) with widest reuse surface. 4–8 well-scoped on partially-built primitives. 9–11 most intellectually interesting, narrower pull.
