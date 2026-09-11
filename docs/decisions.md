---
status: draft
last_verified: 2026-09-11
---

# Decisions

TL;DR: append-only, newest first. Each entry: what, why, what was rejected. Docs describe the present; this file holds the history.

## 2026-09-11 Payroll is phase two, not out of scope

Decision: reclassify payroll from "not for" to "phase two once batch tooling exists". Launch use is one-off payments.
Why: demand evidence is strongest for payroll (research: demand). The earlier objection, "the employer already knows the employee", confused privacy from the payer with privacy from the public; stealth hides that a wallet gets paid at all. The remaining objection is engineering cost of batching, not chain cost.
Rejected: keeping payroll excluded.

## 2026-09-11 ECDH-derived encryption key, one-time accounts enforced

Decision: derive the one-time account's confidential-balance key from the shared secret so the sender can configure and fund alone. Enforce one-time use (fund once, sweep, close) in the program.
Why: the alternative, Token-2022's registry path, skips the proof and owner signature but needs the recipient to pre-register each one-time key, which requires the recipient online per payment and defeats the reusable address. In a strict one-time model the sender learns nothing it doesn't already know.
Rejected: registry path as default (kept as documented option); pre-published proof batches (refill bursts link accounts).
Open: external review of domain separation (see crypto.md).

## 2026-09-11 Pool-less positioning

Decision: position as the pool-less private payment: recipient and amount hidden, sender visible, money stays a normal token, nothing to trust beyond Solana.
Why: Hinkal, Umbra, and Helius Rings already hide all three fields via custodial pools with enclave, MPC, or prover-server trust. "First to hide recipient and amount" is false; "first without a pool" is true and is a trust-model and composability claim reviewers respect.
Rejected: claiming novelty on the privacy properties alone.

## 2026-09-11 Choose private payments as the capstone

Decision: build stealth-address plus confidential-balance payments (Payments domain).
Why: of four finalists it is the only one whose hardest part is cryptography rather than plumbing, is useful with zero partners, and demonstrable from a terminal. Others: permissioned-token liquidity layer (its headline bug turned out fixed upstream in August; LP-token compliance question unresolved), payment-channel dispute layer (Foundation shipped the core on Sep 3), security-token corporate actions (mostly CRUD; issuers already do splits on-chain).
Source: research/2026-09-11-idea-funnel-round2.md.
