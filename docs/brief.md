---
status: draft
last_verified: 2026-09-14
---

# Brief

TL;DR: a confidential payout rail on Solana. A visible payer pays many recipients with encrypted amounts; recipients who want it get a fresh, unlinkable account per payment that the payer can set up alone. No pool, no relayer, no MPC, no enclave. Money stays an ordinary Token-2022 token; an auditor can read amounts.

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

**The rail.** A payout run to N recipients with hidden amounts: proofs generated in batch on the payer's machine, fees handled, rent reclaimed, an auditor path. Most of the engineering lives here.

**The primitive.** For recipients who want to be unlinkable: a fresh, program-owned account per payment whose encryption key is derived from the secret both sides share through the stealth handshake. The payer can produce the key proof and the program signs as owner, so the payer creates, configures, and funds the account alone. Only the recipient can spend from it.

Two recipient modes in the same run:
- **Plain.** The recipient already has a confidential account; we transfer to it. Amount hidden, address fixed.
- **Stealth.** The recipient published one reusable address; each payment lands in a fresh account only they can find. Amount and recipient hidden.

## 4. What is new, what is not

New, as far as we can find [likely, research: competitors-privacy-verified]:
- The payer sets up the recipient's confidential account for them. Solana's zk-sdk ships the key derivation and a helper for program-owned accounts [verified, published in solana-zk-sdk 7.0.1 and @solana/zk-sdk 0.5.2]; nobody has connected them to a stealth scheme.
- Recipient privacy plus amount privacy on unwrapped tokens, with no pool, relayer, committee, or enclave. Every shipped Solana privacy system is a custodial pool that needs at least one of those.

Not new, and cited: program-owned confidential accounts (Occult) [verified]; the stealth construction (BIP-352, ERC-5564, sRFC-42); scan-and-trial-decrypt discovery (Zcash, Monero view tags); batch payout tooling in general.

What the primitive is not: "rotating addresses". Those exist. The new part is payer-side setup of a confidential account for a stranger.

## 5. Who it is for

Primary buyers: stablecoin payroll and B2B settlement platforms serving payers who are public anyway and want their counterparties and amounts private (Toku/Aleo/Paxos private payroll launch, Zebec's $500M/year on Solana with no privacy, Helius naming payroll platforms first) [research: demand]. Launch use: one-off payouts (grants, bounties, invoices, donations), which need no batching. Merchant revenue hiding (PayPal's stated reason for PYUSD confidential transfers) is served by confidential balances alone; we don't compete there.

## 6. Limitations stated up front

- **Payer is visible.** By design; it keeps this outside mixer territory. Need payer privacy? Use a pool.
- **Payer shields once.** The payer must hold a confidential balance; converting treasury into it shows an amount that one time. The recipient's fresh account never goes through a public balance.
- **Consolidation re-links.** Sweeping many fresh accounts into one clusters them. Rule: sweep into a fresh self-owned account, spend from there. Truly unlinkable consolidation is out of scope. [open: MPC-held balance, phase three]
- **Payer holds that one account's key.** Learns nothing new in a strict one-time model; accounts are one-time and closed after sweep, and no one spends directly from a payer-funded account.
- **Cost.** A confidential transfer is three proofs, about five transactions; stealth mode roughly doubles it. About 0.013 SOL locked per payment until close [likely]. Batching amortises proof work, not transaction count.
- **Audit access depends on the mint.** The mint auditor key belongs to the mint authority (Circle for USDC). What we add: either party can disclose one payment's key. [verified, research: compliance]
- **Regulatory exposure.** FinCEN's 2023 proposed mixing definition lists single-use addresses on its own. Counter: visible payer, no pool, recoverable amounts. No agency has drawn the line. [open]
- **Adoption we don't control.** No wallet shows confidential balances. The rail is useful only as far as confidential balances are.
- **Cryptographic review pending.** Two derivations from one shared secret must be domain-separated; recipe written, reviewer not yet. [open]

## 7. Scope

Build: two Anchor programs (address registry; one-time account lifecycle), a TypeScript client (register, pay, batch pay, scan, sweep, close, bench), a privacy model, adversarial tests, a devnet demo, a write-up posted to the sRFC-42 discussion.

Do not build: wallet integration, a web app beyond a status page, our own proof system, unlinkable consolidation, mobile.

## 8. Success criteria

- A payout run to N recipients on devnet, some plain, some stealth, then an auditor decrypting one payment.
- Published measurements: transactions, compute, SOL locked, wall-clock per payment and per run.
- A privacy model listing every artifact per payment and what it links.
- One external applied-cryptography review, findings addressed.
- A write-up someone else could implement from.

## 9. Team

Valentyn: programs, key derivation, sweep authorisation, privacy model. Teammates: TypeScript SDK and batch client; devnet deployment, status page, demo.

## 10. Claims we make

| Claim | Confidence | Source |
|---|---|---|
| No shipped Solana system hides recipient and amount on unwrapped tokens without a pool, relayer, MPC, or enclave | likely | competitors-privacy-verified |
| A program-owned account can be configured for confidential transfers via invoke_signed | verified | token2022-mechanics-verified |
| The zk-sdk key-from-raw-material function is published | verified | crypto-derivation-review |
| Only the recipient can sweep; the payer cannot | verified by construction | crypto-derivation-review |
| Confidential balances have near-zero usage and no wallet support | verified | feasibility, skeptic |
| Payroll privacy is the strongest stated demand | medium | demand |
| Self-hosted transfers are outside EU TFR scope by text | high | compliance |
| ~10 transactions and ~0.013 SOL per stealth payment | likely | token2022-mechanics-verified |
