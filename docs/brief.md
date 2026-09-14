---
status: draft
last_verified: 2026-09-14
---

# Brief

TL;DR: a confidential payout rail on Solana. A visible payer pays many recipients with encrypted amounts; recipients who want it get a fresh, unlinkable account per payment that the payer sets up alone, in one transaction. No pool, no relayer, no MPC, no enclave. Money stays an ordinary Token-2022 token; an auditor can read amounts.

How to read: **[verified]** checked against source, chain, or primary document; **[likely]** strong evidence; **[open]** unknown.

## 1. One line

Pay people on Solana without the public seeing who got paid or how much, while the payer stays visible and an auditor can still read the numbers.

## 2. Problem

Every payout on a public ledger is a permanent record: who was paid and how much. Companies say this is what keeps payroll and supplier payments off-chain: "every public-company CFO gets excited about stablecoins until they realize their payroll would be public" [research: demand].

Solana has both halves of a fix and neither is usable for a real payout flow:

| Tool | Hides | Leaves public | Status |
|---|---|---|---|
| Token-2022 confidential balances | the amount | the recipient's fixed address, so the pattern "X pays W monthly" is visible | mainnet since mid-2026, usage near zero, no wallet support [verified] |
| Stealth addresses (sRFC-42) | the recipient (fresh address per payment) | the amount | draft, devnet reference, dormant since June [verified] |

They cannot be combined today for a concrete reason [verified]: setting up a confidential account requires the owner's signature and a proof that the owner holds the encryption key. A payer sending to a fresh address has neither. So the recipient would have to be online and set up every account by hand.

## 3. What we build

**The rail.** A payout run to N recipients with hidden amounts: one transaction per recipient, proofs generated on the payer's machine, fees handled, rent refunded, an auditor path. Most of the engineering lives here.

**The primitive.** For recipients who want to be unlinkable: a fresh, program-owned account per payment whose encryption key is derived from the secret both sides share through the stealth handshake. The payer produces the key proof and the program signs as owner, so the payer creates, configures and funds the account in one transaction, then hands the account to the recipient's one-time key. From then on the recipient moves the money with plain Token-2022 instructions, in one transaction of its own; our program is never in the funds path again.

Two recipient modes in the same run:
- **Plain.** The recipient already has a confidential account; we transfer to it. Amount hidden, address fixed.
- **Stealth.** The recipient published one reusable address; each payment lands in a fresh account only they can find. Amount and recipient hidden from the public.

## 4. What is new, what is not

New, as far as we can find [likely, research: competitors-privacy-verified]:
- The payer sets up the recipient's confidential account for them. Solana's zk-sdk ships the key derivation and a helper for program-owned accounts [verified, solana-zk-sdk 8.0.0]; nobody has connected them to a stealth scheme.
- Recipient privacy plus amount privacy on unwrapped tokens, with no pool, relayer, committee, or enclave. Every shipped Solana privacy system is a custodial pool that needs at least one of those.
- One transaction per side. With the 4096-byte transaction format every proof rides inline, so a stealth payment is one payer transaction and one recipient transaction [likely, research: arch-r3-verified].

Not new, and cited: program-owned confidential accounts (Occult) [verified]; the stealth construction (BIP-352, ERC-5564, sRFC-42); scan-and-trial-decrypt discovery (Zcash, Monero view tags); batch payout tooling in general.

What the primitive is not: "rotating addresses". Those exist. The new part is payer-side setup of a confidential account for a stranger.

## 5. Who it is for

Primary buyers: stablecoin payroll and B2B settlement platforms serving payers who are public anyway and want their counterparties and amounts private (Toku/Aleo/Paxos private payroll launch, Zebec's $500M/year on Solana with no privacy, Helius naming payroll platforms first) [research: demand]. Launch use: one-off payouts (grants, bounties, invoices, donations), which need no batching. Merchant revenue hiding (PayPal's stated reason for PYUSD confidential transfers) is served by confidential balances alone; we don't compete there.

## 6. Limitations stated up front

- **Payer is visible.** By design; it keeps this outside mixer territory. Need payer privacy? Use a pool.
- **Private from the public, not from the payer.** The payer sees where its payment is swept and can follow that account's later counterparties, amounts hidden, as any payer can on a public ledger. What the rail protects is the recipient's other income, as long as the recipient never merges accounts that different payers can see.
- **Payer shields once.** The payer must hold a confidential balance; converting treasury into it shows an amount that one time. The recipient's fresh account never goes through a public balance.
- **Merging re-links.** Sweeping several payments into one account clusters them. Rule: sweep each into a fresh account. Truly unlinkable consolidation is out of scope. [open: MPC-held balance, phase three]
- **Payer holds that one account's decryption keys.** It learns only the amount it paid: the account accepts exactly one credit, is emptied in one transaction, and only the recipient holds the owning key. If the recipient spends part of the balance straight from that account instead of sweeping it, the payer sees the split.
- **Mint policy decides where it works.** The rail works unmodified on mints whose authority auto-approves new confidential accounts. PYUSD and USDG require the issuer to approve every account, plain or stealth; USDC is not a Token-2022 mint at all [verified 2026-09-14]. On issuer-approved mints the flow needs an approver run by or for the issuer; documented, not built.
- **Cost.** Per stealth payment the payer fronts about 0.006 SOL, refunded at sweep, and sends the recipient about 0.005 SOL for its account and fees; the payer's net cost is fees, of which the priority fee on a 200,000-CU range proof is the only part that grows with congestion [likely, unmeasured]. Batching amortises nothing on chain; it amortises operations.
- **Audit access.** Either party can disclose one payment's receipt, verifiable against the chain without trusting the discloser. A mint auditor key, where the issuer sets one, is the secondary path. [verified, research: compliance]
- **Regulatory exposure.** FinCEN's 2023 proposed mixing definition lists single-use addresses on its own. Counter: visible payer, no pool, recoverable amounts. No agency has drawn the line. [open]
- **Adoption we don't control.** No wallet shows confidential balances. The rail is useful only as far as confidential balances are.
- **Cryptographic review pending.** Several derivations from one shared secret must be domain-separated; recipe written, reviewer not yet. [open]

## 7. Scope

Build: one Pinocchio program (register, close registration, open, reclaim), one Rust CLI (register, pay, scan, sweep, disclose, verify, bench), a privacy model, adversarial tests, a chaos harness for the batch engine, a devnet demo, a write-up posted to the sRFC-42 discussion.

Do not build: wallet integration, a web app beyond a status page, our own proof system, unlinkable consolidation, an indexer beyond demo scale, an issuer approver (a one-page spec ships in the write-up), mobile.

## 8. Success criteria

- A payout run to 50 recipients on devnet, some plain, some stealth, one transaction per recipient; then a third party verifying a disclosed receipt against the chain.
- Published measurements: bytes and compute per transaction, SOL fronted, SOL kept by the recipient, SOL burned, wall-clock per payment and per run.
- A privacy model listing every artifact per payment and what it links.
- The batch engine converging after being killed at every step of a run.
- One external applied-cryptography review, findings addressed.
- A write-up someone else could implement from.

## 9. Claims we make

| Claim | Confidence | Source |
|---|---|---|
| No shipped Solana system hides recipient and amount on unwrapped tokens without a pool, relayer, MPC, or enclave | likely | competitors-privacy-verified |
| A program-owned account can be configured for confidential transfers via invoke_signed; no confidential instruction requires the account itself to sign | verified | arch-r1-mechanics-verified |
| Inline proof offsets resolve correctly when Token-2022 runs under our CPI | verified | arch-r1-mechanics-verified |
| A stealth payment fits one 4096-byte transaction per side, under the 64-account and 64-instruction caps | likely; measured for the pre-handover shape, re-measured in S5 | arch-r3-verified, arch-r5-consistency |
| The zk-sdk key-from-raw-material function is published | verified | arch-r2-verified |
| Only the recipient can move funds out, and it needs only its key and Token-2022, not our program | verified | arch-r5-handover-verified |
| No third party without mint authority can block a funded payment: one credit allowed, public credits disabled, the recipient's transaction atomic; a freeze authority can | verified | arch-r3-crypto-privacy-skeptic |
| Confidential balances have near-zero usage and no wallet support | verified | feasibility, skeptic |
| Payroll privacy is the strongest stated demand | medium | demand |
| Self-hosted transfers are outside EU TFR scope by text | high | compliance |
| PYUSD and USDG require issuer approval per account; USDC has no confidential extension | verified | arch-r2-verified |
| Proof generation is not a bottleneck: a 50-recipient proof set in 0.2 s | verified | arch-r3-verified |
