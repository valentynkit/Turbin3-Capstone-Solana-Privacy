---
status: draft
last_verified: 2026-09-11
---

# Brief

TL;DR: Private payments on Solana that hide the recipient and the amount from the public, keep the sender visible, and let an auditor read amounts. Built on Token-2022 confidential balances plus stealth addresses, with no pool, no relayer, no MPC, no enclave.

## One line

Pay anyone on Solana so the public sees neither who received it nor how much, while the money stays an ordinary token and an auditor can still read the numbers.

## Problem

A public ledger turns every payment into a permanent record. Pay a contractor once and anyone can follow what they earn from then on. Public-company CFOs say this is what stops them moving payroll to stablecoins; fewer than 1% of businesses pay salaries in crypto (research: demand).

Solana has two halves of a fix that were never joined:

| Tool | Hides | Leaves public | Status |
|---|---|---|---|
| Token-2022 confidential balances | the amount | the recipient's fixed address, so the payment pattern is linkable | Mainnet since mid-2026, usage near zero, no wallet support [verified] |
| Stealth addresses (sRFC-42) | the recipient, via a fresh address per payment | the amount | Draft, devnet reference, dormant since June [verified] |

They were never joined for a concrete reason [verified]: setting up a confidential account requires a proof that you hold its encryption key and a signature from the account owner. A sender paying a fresh stealth address has neither. The recipient would have to be online for every payment, which defeats a reusable address.

## Idea

Make the one-time account owned by our program, and derive its encryption key from the secret the sender and recipient already share through the stealth handshake. The sender can then produce the key proof, the program signs as owner, and the sender sets up and funds the account alone. The recipient, who also knows the shared secret, finds the account, decrypts the balance, and proves ownership of the one-time key, which only they hold.

Four steps: publish one reusable address; pay into a fresh program-owned account with an encrypted amount; find it and sweep it; disclose to an auditor if required.

## What is new, what is not

New, as far as we can find [likely, competitors verified]:
- Recipient privacy and amount privacy together, on unwrapped tokens, with no pool, relayer, MPC, or enclave. Every shipped Solana privacy system is a custodial pool that needs one of those.
- Sender-side setup of a confidential account for an offline recipient via an ECDH-derived key. The zk-sdk ships the derivation function [verified, published in solana-zk-sdk 7.0.1 and @solana/zk-sdk 0.5.2]; nobody has wired it to a stealth scheme.
- Verifying a stealth one-time-key signature inside a program to release funds from a program-owned account.

Not new, and cited:
- Program-owned confidential accounts (Occult does it) [verified].
- The stealth construction itself (BIP-352, ERC-5564, sRFC-42).
- Discovery by scan and trial decryption (Zcash, Monero view tags).

## Who it is for

Launch use, one-off payments: grants, bounties, invoices between pseudonymous entities, donations. Cheap to demo, no batching needed.

Primary buyers, phase two: stablecoin payroll and B2B settlement platforms. The demand evidence is strongest there (Toku/Aleo/Paxos private payroll launch, Zebec's $500M/year payroll with no privacy, Helius naming payroll platforms first). Payroll needs batch tooling and an automated close crank; the primitive is the same.

Merchant amount-hiding (PayPal's stated reason for PYUSD confidential transfers) is served by confidential balances alone; we don't compete there.

## Limitations, stated up front

- **Sender is visible.** By design; it is what keeps this outside mixer territory. If you need sender privacy, use a pool (Hinkal, Helius Rings).
- **Consolidation re-links.** Sweeping many one-time accounts into one lets an observer cluster them. Inherent to stealth addresses. Mitigation: spend from them directly or sweep into a fresh one. [open: MPC-based consolidation as phase three]
- **Sender keeps the decryption key of that one account.** Learns nothing new in a strict one-time model, but it is a departure from "only the owner can decrypt". The program enforces one-time use. The alternative (recipient pre-registers keys) needs the recipient online. [open, see crypto.md]
- **Cost.** About ten transactions per payment end to end and about 0.013 SOL locked until close [likely, from the Foundation's sample client]. Fine for one-off, needs batching for payroll.
- **Regulatory exposure.** FinCEN's 2023 proposed mixing definition lists single-use addresses as an indicator on its own. Counter: visible sender, no pool, auditor-readable amounts. No agency has drawn that line. [open, see landscape.md]
- **Audit access depends on who controls the mint.** The mint auditor key belongs to the mint authority (Circle for USDC). What we add is per-payment key disclosure by either party, which needs no issuer cooperation.
- **Cryptographic review pending.** Two derivations from one shared secret must be domain-separated; recipe exists, external review does not. [open]

## Scope

Build: two Anchor programs (address registry; one-time account lifecycle), a TypeScript CLI (register, pay, scan, sweep, close, bench), a privacy model, adversarial tests, a devnet demo, a write-up posted to the sRFC-42 discussion.

Do not build now: batch payroll tooling, wallet integration, a web app beyond a status page, our own proof system, unlinkable consolidation, mobile.

## Success criteria

- Pay, scan, sweep, close end to end on devnet on a Token-2022 mint with an auditor key.
- Published measurements: transactions, compute, SOL locked, wall-clock per payment.
- A privacy model listing every on-chain artifact per payment and what it links.
- One external applied-cryptography review, findings addressed.
- A write-up someone else could implement from.

## Team

Valentyn: programs, key derivation, ownership proof, privacy model. Teammate: TypeScript SDK and CLI, proof generation, benchmarks. Teammate: devnet deployment, fee-paying relayer, status page, demo.

## Claims we make

| Claim | Confidence | Source |
|---|---|---|
| No shipped Solana system hides recipient and amount on unwrapped tokens without a pool, relayer, MPC, or enclave | likely | research: competitors-privacy-verified |
| A program-owned account can be configured for confidential transfers via invoke_signed | verified | research: token2022-mechanics-verified |
| The zk-sdk key-from-raw-material function is published | verified | research: crypto-derivation-review |
| Only the recipient can sweep; the sender cannot | verified by construction | research: crypto-derivation-review |
| Confidential balances have near-zero usage and no wallet support | verified | research: feasibility, skeptic |
| Payroll privacy is the strongest stated demand | medium | research: demand |
| Self-hosted transfers are outside EU TFR scope by text | high | research: compliance |
| ~10 transactions and ~0.013 SOL per payment | likely | research: token2022-mechanics-verified |
