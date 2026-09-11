---
status: draft
last_verified: 2026-09-11
---

# Landscape

TL;DR: Every shipped Solana privacy system is a custodial pool with an anonymity set plus a relayer, enclave, MPC committee, or prover server. We are the pool-less option: weaker (sender visible) and simpler (money stays a normal token, nothing to trust beyond Solana). Compliance posture is defensible but untested; demand is strongest in payroll.

## Positioning

Not the first private payment on Solana. The first pool-less one.

| Property | Pools (Hinkal, Umbra, Helius Rings) | This |
|---|---|---|
| Funds stay ordinary Token-2022 tokens | no, custodied in the pool program | yes |
| Trust beyond Solana | enclave, MPC committee, or prover server | none; proofs made locally, verified by Solana's own program |
| Needs an anonymity set | yes, weak when the pool is small | no, each payment is private on its own |
| Sender | hidden | visible, by design |
| Audit | provider viewing keys, varies | native mint auditor plus per-payment key disclosure |
| Regulatory shape | mixer-adjacent | private statement, visible payer |

## Competitors, verified against primary docs

| System | Hides sender / recipient / amount | Funds stay standard tokens | Trust beyond Solana | Compliance | Status |
|---|---|---|---|---|---|
| Hinkal | yes / yes / yes | no | TEE enclave does proving and key custody, relayer | KYT screening, selective disclosure | mainnet, program closed source |
| Umbra (Solana, unrelated to Ethereum Umbra) | partial / yes / yes | no | Arcium MPC majority, relayer, indexer | hierarchical viewing keys | mainnet-beta, no audit found |
| Helius Rings | default: no / no / yes; anonymous ring: yes / yes / yes | no | prover server, delegated decryption (provider reads balances) | viewing keys, per-ring auditor, freeze lists | devnet, audits in progress |
| Privacy Cash | weak / public at withdrawal / no | no | relayer sees recipient and amount | deposit screening | mainnet |
| Light PSP (2022) | yes / yes / yes | no | relayer | none | dormant |
| Arcium CSPL | unverified / unverified / claimed | yes | permissioned MPC clusters | none documented | not shipped |
| Token-2022 confidential alone | no / no / yes | yes | none | mint auditor key | mainnet |
| sRFC-42 stealth alone | no / yes / no | yes | none | none | draft, dormant |

Details and sources: research/2026-09-11-competitors-privacy-verified.md.

## When to use a pool instead

If the sender must be hidden, use a pool. Ours is the wrong tool for that and we say so. If amounts must be hidden but the recipient is a known merchant, confidential balances alone are enough.

## Compliance posture

Findings [verified against EUR-Lex, govinfo, Cornell LII, Treasury, solana-program docs]:

- Travel-rule duties bind VASPs, not protocols. Self-hosted to self-hosted transfers are outside EU TFR scope by text (Art. 2(4)). Where a VASP is involved, originator and beneficiary data travel off-chain between VASPs regardless of the on-chain address.
- MiCA: a protocol with no issuer or service provider sits outside Titles II to IV (Recital 22). Article 76(3) bars trading platforms from listing assets with an "inbuilt anonymisation function" unless the platform can identify holders and history; listability therefore turns on decrypt access being available to the platform.
- Tornado Cash: the 2022 designation reasoned from pooling that breaks the sender-recipient link; the 2025 delisting was procedural. Treasury remains hostile to mixers.
- The auditor key is per mint and belongs to the mint authority. On USDC that is Circle. Per-account keys can be shared voluntarily; in our design every one-time account has its own key, so either party can disclose a single payment.

Exposure [open]: FinCEN's October 2023 proposed rule defines mixing as obfuscating "source, destination, or amount" and lists "single-use wallets, addresses, or accounts" as an indicator. A literal reading reaches stealth addresses without any pool. Counter: source is public, there is no commingling, amounts are recoverable by disclosure. No agency has addressed this shape; the rule's finalisation status is unverified.

Recommendation: state this as a considered position, not a cleared one; keep the visible sender as a hard design constraint; get outside counsel before any "compliant by design" claim.

## Target users and demand evidence

| Segment | Evidence | Strength |
|---|---|---|
| Stablecoin payroll | Toku CEO: CFOs balk when "their payroll would be public"; Toku/Aleo/Paxos private payroll launch Jan 2026; Zebec $500M/year on Solana with no privacy; Helius names payroll platforms first | strong pain, no Solana-native customer named yet |
| Merchant revenue hiding | PayPal shipped PYUSD confidential transfers for this; payments.org uses the same framing | strong, but amount-only; served by confidential balances alone |
| Recipient privacy for individuals | Fluidkey on EVM: 28k users, $900M volume; HRF on activists under authoritarian regimes | strong analogue, not Solana |
| Grants, DAO payouts, bounties | unverified, research budget ran out | open |
| Freelance and B2B payment products | ten named products, zero stated payee-privacy demand; several market transparency | negative |

Reading: the pain is real and named; nobody on Solana has stated it in their own words yet; the first buyers will be platforms, not end users. Details: research/2026-09-11-demand.md.

## Open questions

1. Does the FinCEN mixing definition, if finalised, reach single-use addresses with a visible sender? Owner: Valentyn; resolve by: before any public launch claim; recommendation: assume yes for planning, keep the sender visible, document disclosure paths.
2. Is there a named Solana payroll or DAO platform that will say "we want this"? Owner: team; resolve by: week 2; recommendation: ask Zebec, Streamflow, Superteam Earn directly.
