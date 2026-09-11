---
status: draft
last_verified: 2026-09-11
---

# FAQ and messages

TL;DR: questions people asked in the cohort channel and the answers we gave, plus short message variants.

## Is this like Tornado Cash?

No, closer to the opposite. Tornado is a mixing pool: everyone dumps funds in, withdraws to a fresh address, and nobody, including regulators, can trace anything. Here there is no pool and nothing is mixed. The sender is visible, the money moves as normal tokens, and an auditor can read every amount. Think private bank statement, not mixer.

## Why not use Arcium?

Arcium hides the amount with MPC; Token-2022 does the same natively with on-chain proofs and no node committee to trust, so we start there. Neither hides who gets paid, which is the part being built. Where Arcium could help: merging many one-time accounts into one is visible with ZK balances; an MPC-held balance could hide that. Phase three candidate.

## Why not full privacy like Midnight or Zcash?

Full shielded pools are silos: funds stop being SPL tokens and nothing else can touch them; they take a year with custom circuits; and hiding the sender is what regulators object to. This keeps the sender visible on purpose and the money composable.

## What about MagicBlock and Helius?

MagicBlock's private payments run in a hardware enclave; Helius Rings is a pool with a prover server and provider-readable balances. Both are valid designs with different trust. Neither is pool-less on unwrapped tokens.

## Doesn't the sender know the recipient's key?

The sender knows the decryption key for the one account it funded, not for anything else. It learns the amount it sent, which it knows, and sees the sweep, which is public. Accounts are one-time and closed after sweep. The alternative, where the recipient pre-registers keys, needs the recipient online per payment.

## Why not payroll first?

Payroll is where the demand is loudest and it is phase two: same primitive, needs batch proof generation and an automated close. One-off payments demo the primitive without that tooling.

## Message variants

Discord, short:

> I'm building private payments on Solana. Here's the thing about a public chain: if I pay you once, anyone can watch everything you earn from then on. That's why most real money still isn't on-chain. Solana already has half-solutions: one hides the amount, another hides who's getting paid. Nobody has put them together without a mixing pool. I want to build the version that keeps both, with an auditor key so a business can still show its numbers. Crypto and protocol work, no frontend. If that sounds like fun, come say hi.

One-liner:

> Pay anyone on Solana so the public sees neither who got it nor how much, with an auditor key so it stays compliant.

Reply to "use Arcium":

> Considered it. Arcium hides the amount, but so does Token-2022 natively with no committee to trust, so I'm starting there. Neither hides who gets paid, which is the part I'm actually building. One place Arcium could shine: merging many one-time accounts into one leaks on-chain. MPC could hide that.
