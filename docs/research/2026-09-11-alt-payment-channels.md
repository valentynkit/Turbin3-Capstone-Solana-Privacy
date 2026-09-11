# R2 — P1 Trustless payment channels + disputes: feasibility + red team (2026-09-11)

## Verdict: Go-with-changes
The core primitive **already shipped, audited, mainnet 2026-09-03 by Solana Foundation**: https://github.com/solana-foundation/payment-channels , program `CHNLxYvVA28MJP9PrFuDXccuoGXAx7jBacfLEkahyGsX`, Cantina audit July 2026. Non-custodial PDA escrow, unidirectional, payer `requestClose` → `grace_period` → payee `settleAndSeal` overrides → permissionless `seal` → `distribute`; partial settlement; Token-2022 allow-listed extensions. docs/001-payment-channel-state-machine.md. "1M/s" = ~1.09M voucher-authorizations/s in a 100k-wallet proxy test, 203s settlement cycle.
Building the channel from scratch = redundant. What remains open: **proof-of-delivery-gated settlement**, **watchtower / liveness delegation**, griefing suite.

## Solved vs open
- Open: no watchtower ("a third party able to pay fees cannot replace a missing payee signature" — CryptoSlate); no delivery-proof hook (claim valid purely by highest nonce); unidirectional only; roadmap docs/004–006 (batch settlement, rearm, Groth16 rollup settlement) unimplemented; issues #83–85 open on Token-2022 coverage + inline Ed25519.
- Prior art: SF `fiber` (research prototype, unaudited); BOBER3r/solana-payment-channel-kit (trusted resolver role); **otomat-fun/otomat** (pushed 2026-09-01) has real `handle_dispute_channel`/`handle_resolve_dispute` with `dispute_window_end` and a `facilitator` co-signer; no delivery-proof gating, no watchtower.
- Design: unidirectional monotonic-nonce channel suffices for x402 metering; Lightning penalty machinery unnecessary; the gap is a **conditional claim** (Perun/Sprites generalized-condition model) not bidirectionality.

## Scope (proposed)
Extension layer on SF channel state (standalone program or module PR). Handlers: open_channel, top_up, submit_claim (Ed25519 precompile + instructions-sysvar introspection), register_delivery_proof, dispute_delivery (novel), request_close, counter_close, delegate_watchtower (data custody of signed voucher, not spend authority), seal, distribute, reclaim, close_cooperative.
Hard: delivery-conditioned claim + dispute path; watchtower incentive design; griefing suite (close/counter-close spam, sysvar spoofing à la Wormhole, voucher replay, rent exhaustion). Plumbing: escrow, CPIs, cooperative close.

## Red team
- Why channels at 400ms/$0.0001: tx count + latency floor; SF's own launch is the proof. But SF captured the core value.
- Alpenglow ~150ms: narrows, doesn't zero, the case. Counter-signal: raxdeveloper/votorflow uses Votor pre-finality receipts instead of channels.
- Foundation redundancy: yes for core; no for delivery/watchtower slice.
- Demand: x402 volume figure unverified; no public SF/Coinbase ask for a dispute layer found (inconclusive). Competing MPP standard (Tempo/Stripe) adds integration risk. SF also ships `solana-foundation/pay` facilitator CLI without channel logic.
- Watchtower reintroduces availability trust; capital lockup (rearm unimplemented); challenge-spam griefing scope of Cantina audit unverified.

## Kill criteria
SF already ships delivery-proof settlement → false. No demand → inconclusive. x402 wash → unverified. Alpenglow kills → unverified/not standalone. Existing project ships delivery+watchtower+suite → false.

## Who cares
SF payment-channels maintainers (#83–85), x402-foundation/Coinbase, Helius, MagicBlock, otomat-fun, Alibaba Cloud, CryptoSlate/Forkast.
