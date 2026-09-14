---
status: draft
last_verified: 2026-09-14
---

# FAQ and messages

TL;DR: questions people asked in the cohort channel and the answers we gave, plus short message variants.

## Is this like Tornado Cash?

No, closer to the opposite. Tornado is a mixing pool: everyone dumps funds in, withdraws to a fresh address, and nobody, including regulators, can trace anything. Here there is no pool and nothing is mixed. The payer is visible, the money moves as normal tokens, and an auditor can read every amount. Think private bank statement, not mixer.

## Why not use Arcium?

Arcium hides the amount with MPC; Token-2022 does the same natively with on-chain proofs and no node committee to trust, so we start there. Neither hides who gets paid, which is the part being built. Where Arcium could help: merging many one-time accounts into one is visible with ZK balances; an MPC-held balance could hide that. Phase three candidate.

## Why not full privacy like Midnight or Zcash?

Full shielded pools are silos: funds stop being SPL tokens and nothing else can touch them; they take a year with custom circuits; and hiding the payer is what regulators object to. This keeps the payer visible on purpose and the money composable.

## What about MagicBlock and Helius?

MagicBlock's private payments run in a hardware enclave. Helius Rings is a custodial pool with a prover server; its anonymous mode currently also requires delegated decryption, which is their product choice rather than something pools need. Both are valid designs with different trust. Neither keeps the money as an ordinary token, and neither lets you pay someone who hasn't been onboarded.

## Doesn't the payer know the recipient's key?

The payer knows the decryption key for the one account it funded, not for anything else. It learns the amount it paid, which it knows, and sees the sweep, which is public. Accounts are one-time, swept into a fresh self-owned account, then closed. The alternative, where the recipient pre-registers keys, needs the recipient online per payment.

## Is this for payroll?

Payroll and B2B payouts are the primary buyers, and the batch tooling for them is core, not phase two. The demo is a payout run to many recipients, some with fixed confidential accounts, some with fresh stealth accounts. What we don't do in six weeks is wallet integration, which is what payroll platforms would need to adopt it.

## Is the novelty just rotating addresses?

No, those exist (sRFC-42, BIP-352). The new part is that the payer can set up the recipient's confidential account for them, which Token-2022 normally blocks: configuring needs the owner's signature and encryption key. A program-owned account plus a key derived from the shared secret gets around it. The payer does the setup; only the recipient can spend.

## Isn't "setup-free" false, since you have to shield tokens first?

That is the payer's side, and it is true: the payer shields once and that one conversion shows an amount. The recipient's fresh account is funded confidential-to-confidential and the recipient does nothing. For payroll that is the right split: the company shields once, employees do nothing.

## Message variants

Discord, short:

> I'm building private payouts on Solana. Here's the thing about a public chain: if I pay you once, anyone can watch everything you earn from then on. That's why most real money still isn't on-chain. Solana already has half-solutions: one hides the amount, another hides who's getting paid. Nobody has put them together without a mixing pool. I want to build the version that keeps both, with an auditor key so a business can still show its numbers. Crypto and protocol work, no frontend. If that sounds like fun, come say hi.

One-liner:

> Pay anyone on Solana so the public sees neither who got it nor how much, with an auditor key so it stays compliant.

Reply to "use Arcium":

> Considered it. Arcium hides the amount, but so does Token-2022 natively with no committee to trust, so I'm starting there. Neither hides who gets paid, which is the part I'm actually building. One place Arcium could shine: merging many one-time accounts into one leaks on-chain. MPC could hide that.
