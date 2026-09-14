---
status: draft
last_verified: 2026-09-14
---

# Decisions

TL;DR: append-only, newest first. Each entry: what, why, what was rejected. Docs describe the present; this file holds the history.

## 2026-09-14 Hand the one-time account to P; no sweep instruction

Decision: open ends by pinning the token account's close authority to the announcement PDA and setting its owner to P. The recipient sweeps with plain Token-2022 instructions signed by P; our program's only later role is reclaim, which closes both accounts and returns rent to the payer. The sweep instruction is gone; the announcement shrinks to 69 bytes (P and mint no longer stored); the state byte is gone.
Why: an end-to-end review (research: arch-r5-e2e-simulation, arch-r5-handover-verified, arch-r5-redesign-search) asked what the program must own after open returns; the answer is nothing. SetAuthority is blocked only by ImmutableOwner or CpiGuard, every later Token-2022 authorisation reads the current owner, and the credit cap has no setter after configure. The one-shot and anti-griefing properties come from the transaction being atomic, not from routing through the program. With the account owned by P, funds survive any bug or upgrade of our program, and an upgrade authority colluding with a payer (who holds the ElGamal key) can no longer move them. The largest instruction disappears.
Rejected: keeping program-owned custody for the rent guarantee (the close-authority pin gives the same guarantee); dropping the sweep destination D (a partial spend from the one-time account shows the payer the split; D stays the client default, whole-balance spend is an option); a payer-created D (the payer would hold its key); deleting the announcement (logs prune, accounts do not); dropping the registry.
Consequence: the payer can never reclaim a never-swept account, since only P can empty it. Stated as a limitation.

## 2026-09-14 Full scope stays; on-chain registry stays

Decision: the six-week table keeps every deliverable despite a 45 to 60 engineer-day cold estimate; the cut order in plan.md is a safety valve, not a plan. The registry stays on-chain as two instructions in the one program.
Why: the work is AI-assisted throughout, so the estimate for an unassisted engineer overstates it. The on-chain registry is the cleaner and more future-proof shape: sRFC-42 interop, the torsion check on B_spend enforced at registration instead of trusted client-side, and any wallet can resolve a meta-address by wallet address without our software.
Rejected: trimming scope up front; an out-of-band address book (simpler, but every integrator would rebuild lookup and validation).

## 2026-09-14 Architecture rounds: one transaction per side, Pinocchio, Rust CLI

Decision: three rounds of parallel drafting, verification and critique (research: arch-r1-*, arch-r2-*, arch-r3-*) replace the two-program Anchor design. Outcome:
- One Pinocchio program with four instructions (register_meta, close_meta, open, sweep). The registry is a PDA family inside it; nothing reads it on-chain.
- No fund instruction. The payer funds with an ordinary Token-2022 confidential Transfer, a top-level instruction in the same transaction as open. The program cannot stop inbound transfers anyway.
- Every proof inline. Instruction-offset proofs resolve under CPI (verified against runtime source), and the 4096-byte v1 transaction format (active on devnet, mainnet expected 2026-09-15) carries all of them. No proof context accounts, no lookup tables, no transient rent.
- Griefing closed by construction: `maximum_pending_balance_credit_counter = 1`, DisableNonConfidentialCredits inside open, and apply, transfer, empty, close in one sweep instruction. Each of the three was a source-verified attack on the previous shape.
- Sweep destination D and its owner are created by P and funded from P; D_owner is derived from the recipient's secret and (E, k); P drains to D_owner. Nothing on the recipient side is persisted.
- Rent refunds go to the payer. The payer never acts after its one transaction.
- One Rust CLI. Proof generation measured at 16 ms for the range proof and under 0.5 ms for the rest, so a TypeScript engine has no reason to exist and the batch engine runs sequentially per source account with just-in-time proofs.
- Relayer removed from the design and the brief.
- `k` in every PDA seed from day one.
Why: two independent drafters converged on one program, no fund instruction and a deterministic run seed; the verifier overturned the premise that context-state accounts were required; the skeptics found griefing windows at every transaction boundary, and the single-transaction shape removes the boundaries rather than guarding them.
Rejected: Anchor; two programs; a fund instruction; context-state accounts as the default; a separate close instruction (one dust credit would force a second full sweep); a TypeScript client; stuffing E into `decryptable_zero_balance` (scan set would be every token account of the mint); one ephemeral key per run (deferred behind the crypto review); a two-byte view tag (a scale we are not at).
Fallback: research/2026-09-14-arch-r2-candidate-v1 if v1 transactions fail spike S1.

## 2026-09-14 Auto-approve mints first; issuer path documented, not built

Decision: v1 targets mints whose authority auto-approves new confidential accounts. Issuer-approved mints (PYUSD, USDG) are documented as a two-step flow on both sides needing an approver run by or for the issuer.
Why: on-chain reads show PYUSD and USDG set `auto_approve_new_accounts = false` under one Paxos authority, and USDC is legacy SPL Token with no confidential extension. Offline receiving on those mints depends on a third party. Building an approver speculatively adds a service we cannot run.
Rejected: headlining issuer partnerships; splitting the story into plain-only on stablecoins.

## 2026-09-14 Brief refinements from the architecture rounds

Decision: the brief now states that the rail is private from the public, not from the payer; that mint policy decides where it works; that the cost is fees plus about 0.005 SOL passed to the recipient; success criteria name 50 recipients, a verified receipt and a converging chaos harness; the relayer is gone.
Why: each was a finding a critic could defend with a source, and a brief a reader could not implement from is a liability.
Rejected: keeping "~10 transactions and ~0.013 SOL" (undercounted its own research and is now wrong in the other direction).

## 2026-09-14 Payments privacy is the capstone; batch exchange dropped

Decision: return to this project. The confidential batch exchange explored with a teammate on 2026-09-12 to 14 is not pursued.
Why: the exchange's core claim (a program clearing hidden orders) needs a trusted decryptor, a committee, or a reveal step; it fits none of the cohort's four domains; nothing like it is live on Solana and demand is unproven. This project fits Payments, is verified step by step against Token-2022 source, and demos from a terminal.
Rejected: continuing the exchange as a joint capstone.

## 2026-09-14 Re-centre on the payout rail; two sweep corrections

Decision: headline is a confidential payout rail with two recipient modes (plain confidential account; stealth one-time account). Batch payout tooling moves from phase two into the core. Two design changes: the recipient signs the sweep transaction with the one-time key P and pays the fee from dust the payer left on P (no precompile introspection, no relayer); sweeps always go to a fresh self-owned account, never spent directly from a payer-funded account (the payer holds that account's key).
Why: demand evidence points at payout flows, not "pay a stranger"; the honest critique is that recipient privacy is second-order to amount privacy and the rail attacks the first-order pain while keeping the primitive as the differentiator. The sweep corrections came from tracing the fee-payer problem and the payer-key issue to their conclusions.
Rejected: "pay a stranger privately" as the headline; precompile-based in-program signature verification; spending directly from one-time accounts.

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
