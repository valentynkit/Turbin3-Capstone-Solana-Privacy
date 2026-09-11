# Round 2 verdict (2026-09-11)

All four came back "Go-with-changes". Re-scored after primary-source reading and on-chain checks (research/07–11).

| | Novelty (engineer-wow) | Verified ecosystem need | Independence from others' roadmaps | Scope after narrowing | Main risk |
|---|---|---|---|---|---|
| **T1** liquidity layer for permissioned tokens | Medium — hardening + adversarial suite, not new crypto | **High, verified on-chain**: xStocks + Ondo ship hooks armed-but-null, DAS=initialized; Raydium rejects the extensions; no DEX auto-thaws vaults; token-acl has zero vault example; #66 open 20 months | High — nobody (SF, DEXs) is building it; RFP dead but the standard's own repo has the hole | AMM vault + compliance adapter + router shim + attack suite. Fits 2–3 devs | "Just an AMM with a thaw ix" unless the suite + upstream bug reports carry it; securities-law framing |
| **P2** stealth × confidential payments | **High** — PDA-owned stealth confidential account with ECDH-derived ElGamal key + discrete-log sweep proof; nobody has combined stealth + hidden amounts on any chain | Medium — SF pushes confidential balances + auditor keys; usage ~0; sRFC 42 dormant, devnet only | High — no one else is on it | Registry + pinboard programs + scanner/sweep SDK + batch payout. Fits 2–3 devs | Crypto soundness unreviewed; batch proof cost + rent unbenchmarked; wallet support nil |
| P1 payment channels + disputes | Medium — conditional claims + watchtower | Medium — SF shipped the core (audited, 2026-09-03); the gap is real but narrow | **Low** — it's an add-on to SF's program; SF may ship it next | Delivery-proof claims + watchtower + griefing suite | Redundancy; demand unverified; identity as "someone else's extension" |
| T2 corporate-actions engine | Low-Medium — snapshot trust + crank-less claims; splits/halts already done by issuers (on-chain verified) | Medium — issuers do it off-chain on purpose | High | Snapshot + distribution only | Mostly CRUD; legally aspirational; needs T1 to matter |

## Cut
- **T2**: after on-chain check, the remaining novelty is a merkle-claim distributor + issuer-asserted snapshot. Fails the "engineer amazed" bar.
- **P1**: SF's audited channel program ate the core. What's left is a valuable OSS contribution (PRs against `solana-foundation/payment-channels`) but a weak capstone identity. Keep as a side-contribution idea.

## Finalists
- **T1** — wins on *verified need + clear adopters* (tiago18c, joncinque, Orca). Pitch line: "xStocks and Ondo compiled transfer hooks into their mints and left them switched off, because no DEX can hold a permissioned token. We build the vault layer that lets them switch it on, and the attack suite that proves it safe."
- **P2** — wins on *novel cryptography*. Pitch line: "Confidential balances hide the amount but not who you pay. Silent payments hide who you pay but not the amount. We're the first to do both, with an auditor key for compliance."

## Round 3 (proposed)
For each finalist: (1) design-skeptic pass on the narrowed scope, (2) demo storyboard (what a judge sees in 3 minutes with no frontend), (3) the one technical spike that de-risks the main risk — T1: build a Token ACL mint + PDA vault + permissionless thaw on Surfpool and reproduce #66; P2: derive the ElGamal key from an ECDH secret and run ConfigureAccount for a PDA-owned account on devnet, plus a proof-cost benchmark. Then pick one.
