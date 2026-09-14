---
status: draft
last_verified: 2026-09-14
---

# Arch round 1: creative alternatives

TL;DR: nineteen ways to realise the same vision with less machinery. The strongest cluster deletes the announcement account, the state machine and the fund instruction, leaving a three-instruction Pinocchio program and one client.

Vision fixed: visible payer, no pool, no relayer by default, plain Token-2022 money, auditor reads amounts. Stack fixed: Pinocchio programs, one Rust client. Marks: [verified] read in source or research, [likely] strong inference, [open] unknown.

---

## 1. Payer-direct funding, no fund instruction

**What.** A Token-2022 confidential Transfer requires only the source owner to sign. The destination account is not a signer [verified, processor.rs owner validation path, research: token2022-mechanics-verified]. So the payer funds the one-time account with a plain confidential transfer, exactly as it pays a plain recipient. Our program never sees the funding.

**Why better.** Deletes one instruction, one CPI, one state transition. The client gets a single transfer implementation for both recipient modes instead of two. Maybe 300 lines and a class of state bugs gone.

**Costs.** Double-fund prevention leaves the program (see idea 2). The program can no longer assert "funded exactly once" from its own state.

**Verify.** LiteSVM: configure a PDA-owned account, then transfer into it from a normal wallet with no CPI. Exit: transfer lands, pending balance non-zero.

**Confidence.** High. Destination-not-signer is how every confidential transfer already works.

## 2. One-time-ness from the credit counter

**What.** Set `maximum_pending_balance_credit_counter = 1` at configure. Token-2022 then rejects a second incoming transfer before ApplyPendingBalance [likely, default is 65536, the counter is checked on credit]. Fund-once is enforced by the token program, not by us.

**Why better.** Removes the state enum (opened / configured / funded / swept) and every guard that reads it. With idea 1 it removes the reason the StealthAccount record exists at all.

**Costs.** A griefer can burn the single credit by sending 0, or the payer can send twice by accident and the second fails loudly. Recovery is the k index (open k=1).

**Verify.** Mollusk: configure with max counter 1, transfer twice. Exit: second transfer fails with the counter error.

**Confidence.** Medium-high. Semantics read from docs, not yet from the processor.

## 3. P-seeded PDA replaces StealthAccount

**What.** Seed the owning PDA on the one-time key itself: `["s", P]`. At sweep the program checks the signer equals P and that the PDA re-derives from it. The PDA address is the commitment to P, so no account needs to store P, E, mint or state.

**Why better.** Deletes an account per payment: 0.00134 SOL of rent, one create and one close transaction, and about 150 lines of serialisation. Combined with ideas 1 and 2 the program shrinks to open, sweep, close.

**Costs.** E must be published somewhere else (ideas 4, 5). Losing the record loses the audit-time convenience of reading one account to see a payment's shape.

**Verify.** Read `pinocchio.md` PdaSeeds; write the sweep check in 20 lines and prove in Mollusk that a wrong P fails.

**Confidence.** High. Pure PDA arithmetic.

## 4. Announcement inside decryptable_zero_balance

**What.** ConfigureAccount takes a 36-byte `decryptable_zero_balance` that Token-2022 stores unchecked [verified, research: token2022-mechanics-verified]. The payer writes E (32 bytes) plus the view tag there. Discovery becomes a `getProgramAccounts` on Token-2022 with a memcmp on mint plus a memcmp on the tag byte, at fixed offsets.

**Why better.** Zero announcement accounts, zero announcement instructions. The account that must exist anyway carries the announcement. Saves another 0.0013 SOL and a transaction per payment.

**Costs.** Abuses a field with a declared purpose. The recipient overwrites it at ApplyPendingBalance, harmless once discovery happened. A future Token-2022 version that validates the ciphertext breaks it; the fallback is idea 5 plus E in the open instruction data.

**Verify.** LiteSVM: configure with arbitrary 36 bytes, read them back. Exit: bytes survive, offsets stable across the ImmutableOwner layout.

**Confidence.** Medium. The unchecked store is verified, the layout stability is not.

## 5. One ephemeral key per run

**What.** BIP-352 style: one scalar e for the whole payout run, one E posted once, and S_i = ECDH(e, B_scan_i) differs per recipient anyway. Per-payment announcement data drops to one tag byte.

**Why better.** A recipient scans by doing one ECDH per run, not one per candidate account. At 1M announcements from 1000 runs that is 1000 ECDH instead of thousands of trial decryptions, and the tag then names the account exactly. Announcement bytes per run of 500 drop from 16KB to 32 bytes plus tags.

**Costs.** Everyone in a run shares E, so identifying one payment groups the rest as same-run. The payer already groups them by time and signature, so the marginal leak is small. Reusing e across runs is a real break, so the run id must enter the derivation.

**Verify.** Read BIP-352 output derivation; re-derive the tree in crypto.md with k as the recipient index under a fixed E. Exit: an external reviewer signs off on E-per-run plus k-per-recipient.

**Confidence.** High that it works, medium that the privacy trade is acceptable without review.

## 6. View tag width chosen by scan math

**What.** Use two bytes, not one. The filter runs server-side in the RPC, so a wider tag costs nothing in bandwidth and cuts false candidates by 256.

**Why better.** At 1M announcements a 1-byte tag returns about 3900 candidates per scan; 2 bytes returns about 15. That is the difference between a scan that needs pagination and one that fits a single RPC response.

**Costs.** Two bytes of the shared secret hash are public instead of one. Standard schemes chose one byte for bandwidth reasons that do not apply to a server-side memcmp.

**Verify.** Generate 100k synthetic accounts on a local validator, time gPA with each filter width.

**Confidence.** High.

## 7. Registry as a PDA, or no registry

**What.** Drop the registry program. Either the stealth program owns a `["meta", wallet]` PDA, or the meta-address is a 64-byte bech32m string exchanged out of band like an IBAN, with no chain state at all.

**Why better.** One fewer program to write, deploy, upgrade and key-manage. With Pinocchio the whole registry is about 80 lines inside the existing program. The no-registry variant removes a component entirely and is honest: a payer needs the recipient's address by some channel regardless.

**Costs.** No on-chain lookup by wallet. If an ecosystem registry ever matters, slnt already has one on devnet to reuse.

**Verify.** Write the payer client against a local address book first. Exit: the demo runs without a registry; add one only if the demo actually needs it.

**Confidence.** High.

## 8. Inline proofs instead of context accounts

**What.** For the small proofs (pubkey validity at configure, zero-ciphertext at close), put the ZK verify instruction at top level in the same transaction and pass `ProofLocation::InstructionOffset`. Token-2022 reads the instructions sysvar, which reflects top-level indices, so a CPI'd ConfigureAccount should still resolve a top-level offset.

**Why better.** Each context account avoided is a create transaction, a close transaction and about 0.0013 SOL of transient rent. Two proofs per payment means four fewer transactions in a run of one.

**Costs.** design.md decided context-state everywhere on the assumption that CPI breaks offsets. If that assumption is right this idea dies for configure but may still hold for close.

**Verify.** One LiteSVM test, an hour of work. Exit: ConfigureAccount via invoke_signed with InstructionOffset(1) succeeds.

**Confidence.** Medium. The mechanics research lists inline configure in its own transaction sequence, which suggests it works, but nobody has run it behind a CPI.

## 9. Open plus configure in one transaction

**What.** A v0 transaction with a lookup table holding the static addresses (our program, Token-2022, system, ZK program, mint), our open instruction (create the PDA token account, initialize, configure by CPI, fund P's dust), and the top-level pubkey-validity verify.

**Why better.** Budget check: the proof payload is 64 bytes, five static addresses cost 1 byte each through the table, three or four fresh writable addresses cost 32 bytes each, one signature. Well under 1232 bytes [likely]. Stealth setup becomes one transaction instead of three.

**Costs.** Needs the lookup table created and warmed one slot ahead of the run. Depends on idea 8.

**Verify.** Build the transaction and call `getTransaction` size locally before sending. Exit: serialised size under 1232 with margin.

**Confidence.** Medium-high, conditional on idea 8.

## 10. Pipelined proof context accounts

**What.** For the funding transfer's three proofs, overlap the lifecycle across recipients: close recipient i's context accounts in the same transaction that creates recipient i+1's, and reuse a single spl-record account for every range proof chunk in the run.

**Why better.** A 100-recipient run creates and closes 300 context accounts plus 100 record accounts. Pipelining and record reuse removes roughly 100 transactions and keeps transient rent at one recipient's worth (about 0.0085 SOL) instead of the batch's.

**Costs.** A failed transaction now spans two recipients, so the resume logic gets harder. Worth doing only after idea 12 exists.

**Verify.** Measure a 20-recipient run both ways. Exit: transaction count drops at least 25 percent with no extra failure modes in the chaos harness.

**Confidence.** Medium. Close-then-create ordering inside one transaction is untested here.

## 11. The payer reaps the rent

**What.** The payer holds the account's ElGamal key, so the payer can generate the zero-balance proof and close the account after the sweep. Rent goes back to the payer, who paid it. No crank, no incentive design, no recipient action.

**Why better.** Removes the whole permissionless-close discussion. Recovers 0.0042 SOL per payment automatically as a background mode in the payer client. On a 500-person monthly run that is about 2 SOL a month returning instead of bleeding.

**Costs.** The payer has to keep polling until the account is swept, a small ongoing job. design.md currently sends rent to the sweep destination, which gifts the recipient the payer's money and hides nothing: the sweep transaction already names both accounts.

**Verify.** Read `EmptyAccount` in Token-2022 processor for who must sign. Exit: confirm only the owner (our PDA) signs and the proof generator is unconstrained.

**Confidence.** High for the mechanics, medium for the polling ergonomics.

## 12. Deterministic run seed, chain is the state

**What.** Derive every ephemeral key from a run secret: e = HKDF(run_secret, run_id, i). Persist nothing but the run secret. Resume after a crash by re-deriving and re-attempting every step; a second open of the same PDA fails with "already initialized", which the client treats as success.

**Why better.** Idempotence for free. No run manifest account, no local database, no bitmap, no contention on a shared run account. Perhaps 30 lines of client code instead of a persistence layer.

**Costs.** Losing the run secret loses the ability to reap rent or answer an audit for that run. It becomes a real secret to back up. Reusing (run_id, i) across runs collides PDAs, so the run id must be unique by construction.

**Verify.** Chaos harness (idea 13). Exit: a killed run resumes to the same final state with no duplicate payments.

**Confidence.** High.

## 13. Chaos harness over the batch client

**What.** A LiteSVM test that runs a 20-recipient batch and kills the client at every step index, then resumes and asserts convergence: every recipient funded exactly once, no orphan accounts, no double sweeps.

**Why better.** Partial failure is the most likely source of real bugs in this system and the least likely to be found by hand. About 100 lines of harness buys the entire robustness section of the write-up.

**Costs.** A day of work that produces no demo footage.

**Verify.** It is the verification. Exit: 7 kill points, 7 clean resumes.

**Confidence.** High.

## 14. One Rust client, no TypeScript

**What.** Drop the TypeScript SDK and Codama. One Rust binary does derivation, proof generation, batch payout, scan, sweep, close, reap and benchmarking.

**Why better.** The owner writes no frontend and the demo is a terminal. This removes an npm toolchain, a wasm proof-speed risk flagged in plan.md as a rewrite trigger, and a second implementation of the derivation tree that would need its own test vectors. Probably two weeks. Proof generation gets rayon and real parallelism for free: 8 cores turn a 100-recipient run's proof work into roughly a second of wall clock [open, needs measurement].

**Costs.** No browser path without doing the TS work later. A TypeScript-only teammate has less to do.

**Verify.** Time one range proof in Rust and in `@solana/zk-sdk` wasm. Exit: if wasm is within 2x, the argument is convenience only, not performance.

**Confidence.** High.

## 15. Probe the mint policy first

**What.** Before anything else, read `ConfidentialTransferMint` on the mints that matter (USDC, PYUSD) and check whether new accounts are auto-approved or need the mint authority's approval.

**Why better.** If approval is required, every one-time account needs the issuer to approve it and offline receiving is dead on that mint. This is a go/no-go for the headline use case and costs twenty lines of script.

**Costs.** None. The answer may be unpleasant.

**Verify.** `getAccountInfo` on the mint, parse the extension. Exit: a recorded policy value per mint, dated.

**Confidence.** High that the check is cheap, [open] on the answer.

## 16. Self-authenticating disclosure receipt

**What.** A per-payment disclosure is `{E, k, ct_ikm}` as JSON. The verifier tool re-derives the ElGamal and AES keys, checks the derived public key matches the one in the token account on chain, then decrypts the amount from the transfer ciphertext and prints a receipt.

**Why better.** The auditor does not have to trust the discloser: a forged ikm fails the pubkey match. Either party can produce it, which is what an audit or a subpoena actually needs. About 150 lines, and it is a demo artifact you can hand someone.

**Costs.** Disclosing ct_ikm hands over that account's full read access, which is the intended granularity but must be stated.

**Verify.** Write the verifier against a devnet payment. Exit: a wrong ikm is rejected by the pubkey check, not by a decryption failure.

**Confidence.** High.

## 17. Exact dust, drained at sweep

**What.** Fund P with a computed amount (base fee times the number of sweep transactions, plus a priority allowance, call it 0.001 SOL) and drain P to zero in the last sweep transaction. A fee-payer PDA is impossible, because the fee payer must sign and PDAs cannot sign the transaction envelope [verified].

**Why better.** Dust is small next to rent: 0.001 SOL against 0.0042 SOL of account rent, so the fee question is a liveness question, not a cost question. Draining P at sweep returns most of it.

**Costs.** If fees rise above the allowance the sweep stalls. Anyone can top P up, but a top-up from the recipient's wallet links them. That is the real failure mode and it argues for a generous allowance.

**Verify.** LiteSVM: fund P below the fee, attempt sweep, then top up and retry. Exit: failure is clean and recoverable, and a zero-lamport system account with no data is allowed.

**Confidence.** Medium-high. The rent-exemption floor on a drained 0-data system account is [open].

## 18. Privacy diff tool

**What.** A command that takes a run id and prints two columns: what happened, and what a chain observer can reconstruct from the chain alone. Reads the chain only, uses no secrets beyond what an observer has.

**Why better.** Turns privacy.md from a claim into an executable artifact. It also catches leaks the doc missed, because the tool enumerates artifacts mechanically.

**Costs.** About 200 lines and a day. No cryptographic risk.

**Verify.** Run it on the demo, diff against privacy.md's artifact table. Exit: the tool finds at least one artifact the table does not list, or the table is confirmed complete.

**Confidence.** High.

## 19. Pinocchio deploy economics and a CU gate

**What.** Budget the program at 10 to 15KB instead of the 200 to 400KB an Anchor build produces, and put a Mollusk CU assertion on every instruction in CI so a regression fails the build.

**Why better.** Deploy and upgrade cost scales with binary size: roughly 2 SOL for a 300KB Anchor program against under 0.1 SOL for a Pinocchio one, doubled transiently by the buffer. On devnet that is the difference between airdrop-limited and not. The CU gate makes the published measurements a byproduct of CI rather than a week-5 task.

**Costs.** Manual account validation, one-byte discriminators, no IDL. Every hand-written check is one that can be forgotten, so the Pinocchio security checklist becomes mandatory review material.

**Verify.** Build a hello-world Pinocchio program, check the .so size and the deploy cost on devnet.

**Confidence.** High.

---

## Ranking by expected value over verification cost

| Rank | Idea | Value | Verify cost | Notes |
|---|---|---|---|---|
| 1 | 1. Payer-direct funding | high | 1 hour | deletes an instruction and a CPI |
| 2 | 15. Probe the mint policy | high | 20 minutes | possible go/no-go, nearly free |
| 3 | 2. Credit counter as one-time enforcement | high | 1 hour | deletes the state machine |
| 4 | 3. P-seeded PDA | high | 2 hours | deletes an account per payment |
| 5 | 12. Deterministic run seed | high | half a day | resume for free |
| 6 | 7. Registry as PDA or none | medium-high | 0 | a program you never write |
| 7 | 14. Rust-only client | high | half a day | removes a toolchain and a risk |
| 8 | 8. Inline proofs | high | 1 hour | four fewer transactions if it holds |
| 9 | 6. Two-byte view tag | medium | 2 hours | scan scaling |
| 10 | 13. Chaos harness | medium-high | 1 day | pays for itself in bugs found |
| 11 | 11. Payer reaps rent | medium-high | 1 hour | kills the crank discussion |
| 12 | 5. One ephemeral key per run | high | needs review | blocked on crypto review |
| 13 | 16. Disclosure receipt | medium-high | 1 day | demo artifact |
| 14 | 18. Privacy diff tool | medium | 1 day | demo and doc validation |
| 15 | 4. Announcement in decryptable_zero_balance | medium-high | 2 hours | clever, fragile |
| 16 | 9. Open plus configure in one tx | medium | 2 hours | depends on 8 |
| 17 | 19. Pinocchio deploy and CU gate | medium | 2 hours | mostly already decided |
| 18 | 17. Exact dust | medium | 2 hours | liveness, not cost |
| 19 | 10. Pipelined context accounts | medium | 1 day | do after 12 |

Ideas 1, 2, 3 and 4 compose into one shape: a three-instruction program (open, sweep, close) with no announcement account, no state enum and no registry. Ideas 5 through 10 are the transaction-count work. Ideas 11 through 19 are robustness, tooling and evidence.

---

## Brief refinements

None of these touch the core identity: confidential payout rail, visible payer, no custodial pool. They are places where the architecture argues the brief is describing a bigger system than the one worth building.

**B1. Scope: one program, one Rust client.** §7 says "two Anchor programs (address registry; one-time account lifecycle), a TypeScript client". Ideas 1, 2, 3 and 7 collapse that to one Pinocchio program with three instructions, and idea 14 argues the client should be Rust. Change §7 to "one Pinocchio program (open, sweep, close), one Rust CLI". Why: the registry is 80 lines inside the existing program or nothing at all, and a second language buys a browser path nobody is building. Knock-on: §9 gives a teammate "TypeScript SDK and batch client", which becomes "batch client and demo tooling in Rust" or the teammate takes the scan and disclosure tools instead.

**B2. Drop the relayer entirely.** §6 and design.md both carry the relayer as an optional component. Nothing in this architecture needs it: P signs its own sweep and pays from dust. Keeping it optional still costs a threat-model section and a privacy caveat. Change "no relayer by default" to "no relayer", and delete the relayer row from design.md's component table and privacy.md's adversary table. Why: an unbuilt optional component is a claim you have to defend without evidence.

**B3. Add the mint approval policy as a stated limitation.** §6 lists cost, consolidation and audit access but not this: the rail only works on mints that auto-approve new confidential accounts. If a mint requires the authority to approve each account, offline receiving is impossible on that mint and the stealth mode dies there. Add a bullet, marked [open] until idea 15 runs. Why: it is the most likely external blocker and it is currently invisible.

**B4. Make the disclosure receipt the primary auditor path.** §6 frames audit access as dependent on the mint auditor key, with per-payment disclosure as an addition. Idea 16 inverts it: the receipt is self-authenticating, works on any mint, needs no issuer cooperation, and either party can produce it. Rewrite the bullet with disclosure first and the mint key as a note. Change the §8 criterion from "an auditor decrypting one payment" to "a third party verifying a disclosed payment against the chain without trusting the discloser". Why: the stronger claim is also the one we control.

**B5. Restate the cost claim as a target with a range.** The claims table says "~10 transactions and ~0.013 SOL per stealth payment [likely]". Ideas 1, 2, 3, 8 and 10 plausibly reach 6 to 7 transactions; ideas 3 and 4 remove about 0.0027 SOL of rent. Change the row to a measured-by-week-2 target with a floor and a ceiling rather than a point estimate. Why: a point estimate that improves by 40 percent reads as a wrong number either way.

**B6. Fix N in the success criteria and name the robustness artifacts.** §8 says "a payout run to N recipients". Pick a number (50 is enough to show batching and small enough to fund on devnet) and add two deliverables that the architecture says matter more than they look: the chaos harness (idea 13) and the privacy diff tool (idea 18). Why: "adversarial tests" as a scope line has no exit criterion; these two do.

**B7. Promote one-time-ness from apology to invariant.** §6 bullet "Payer holds that one account's key" and the crypto.md trade-off both depend on accounts being strictly one-time. Idea 2 makes Token-2022 enforce that with its own counter rather than our state machine. Add a claims-table row: "one-time use is enforced by the token program, not by our program logic" [likely, pending the Mollusk check]. Why: the whole payer-key argument rests on it, and enforcement by a program we did not write is a much better answer to a reviewer.
