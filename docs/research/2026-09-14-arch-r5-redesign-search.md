---
status: draft
last_verified: 2026-09-14
---

# Arch R5 — redesign search over the decided shape

TL;DR: the decided design searched "what instructions does our program need" and found a good answer; it never searched "what must our program own", and that is where the remaining simplification is.
One change dominates: hand the one-time token account to P at the end of open, delete sweep, and the program leaves the funds path forever. Everything else here is small or worse.

Marks: **[verified]** read in source this round, cited by line; **[likely]** inference over verified facts; **[open]** unknown.

Source read this round: `solana-program/token-2022` `main`: `program/src/processor.rs` (`process_set_authority`, `process_close_account`), `program/src/extension/confidential_transfer/processor.rs` (`process_configure_account`, `process_approve_account`), `interface/src/extension/mod.rs` (`init_extension`, `alloc`).

## The constraint that generates the whole shape

ConfigureAccount takes the ElGamal pubkey either from a proof of knowledge of the secret or from an ElGamal registry account whose owner equals the token account's owner (`confidential_transfer/processor.rs`, `process_configure_account`, the `ElGamalPubkeySource` match) [verified]. Offline setup means no recipient signature and no recipient-created registry, so the payer must know the ElGamal secret. And the key cannot be rotated afterwards: ConfigureAccount calls `init_extension::<ConfidentialTransferAccount>(false)`, and `alloc` returns `ExtensionAlreadyInitialized` when the extension exists and overwrite is false [verified]. There is no other path: Withdraw publishes the amount, Deposit reuses the same key, CloseAccount needs a zero balance.

So the money must move exactly once before the recipient can hold it under a key the payer does not have. That is Token-2022's choice, not ours. What is ours is who signs that move, where it goes, and who owns the account until then.

## A. Hand the account to P at the end of open

**Change.** `open` ends with two CPIs: SetAuthority(CloseAccount -> StealthAccount PDA), then SetAuthority(AccountOwner -> P). ImmutableOwner is not set. The recipient drains with plain Token-2022 instructions signed by P (ApplyPendingBalance, Transfer, EmptyAccount, top level, one transaction). Our program has no sweep.

`process_set_authority`'s `AccountOwner` arm blocks only on ImmutableOwner and CpiGuard, with no confidential-transfer condition at all (`processor.rs:730-767`) [verified]; `process_close_account` authorises against `close_authority.unwrap_or(owner)` (`processor.rs:1319-1343`), so pinning close_authority to the PDA survives the owner change [verified].

**Disappears.** The sweep instruction and every guard in it; four CPIs plus a PDA close plus a System drain, all hand-encoded; `state`, `P` and `mint` in the announcement; the `drain_to` argument; the sweep golden-byte tests; and our program's standing authority over recipient funds. The recipient's transaction stops touching our program, so it is indistinguishable from any confidential transfer.

**Appears.** One SetAuthority encoder, a `reclaim` instruction (CPI CloseAccount signed by the PDA, lamports to the stored payer, then close the announcement, gated on the token account's balance being zero), and a payer-side reclaim loop as backstop. The recipient's client includes `reclaim` as a top-level instruction in its own transaction on the happy path, so the common case is still one transaction per side and no crank.

**Guarantees.** Unchanged where it matters. The one-shot property never depended on sweep being one instruction, only on one transaction: the credit counter capped at 1 blocks a second confidential credit, DisableNonConfidentialCredits blocks public dust, and apply/transfer/empty in one transaction leaves no boundary. What we gain is the brief's strongest claim becoming literally true. Today the PDA owns every unswept account, so a bug in sweep bricks funds permanently, and an upgrade authority colluding with the payer (who holds the ElGamal secret and can build a valid transfer proof) can take them; ImmutableOwner blocks the upgrade authority alone, not the pair. After A, the recipient depends on Token-2022 and nothing else, and any wallet that speaks confidential transfers recovers the money without our software.

**Costs.** The payer acts again: reclaim is a second, batchable transaction for stragglers, roughly 15 accounts each, cheap in lamports and about 150 lines in the engine. decisions.md liked "the payer never acts after its one transaction"; that goes. Without the close_authority pin the refund becomes a convention a hostile client can ignore, about 6.05M lamports per payment, so pin it. Payer reclaim of never-swept accounts (Q10) becomes impossible forever: a loss of flexibility, a gain in honesty. The recipient's transaction grows by about 20 bytes and loses an account key [likely].

**Verify.** One LiteSVM test, an hour: configure a PDA-owned account, pin close_authority to the PDA, set owner to a fresh keypair, fund it, apply/transfer/empty signed by that keypair with no call into our program, then reclaim. Confidence: high on mechanics, high on value.

## E. No sweep transfer at all

**Change.** Drop the dedicated destination D. The recipient's one outbound move is its first real spend, to whoever it is actually paying, full balance, followed by EmptyAccount.

Safety follows from the section above: the payer can decrypt the balance and every transfer ciphertext under the source key, so a partial spend tells it the split, while a full-balance spend tells it only the number it already knows. The sweep to D is not a privacy mechanism, it is a full-balance spend with a self-owned destination done early. Deferring it to the recipient's first real payment is the same operation with a useful destination.

**Disappears.** D, D_owner, the `d` branch of the derivation tree, the VerifyPubkeyValidity for D, five CPIs in the recipient's transaction, about 600 bytes, and the payer's prefunding of D's rent. Per-payment SOL handed to the recipient drops from about 0.0047 to fees only, roughly 0.0002.

**Appears.** A client rule stated as loudly as "never merge accounts": spend the whole balance, or lose amount privacy to that payer. Recipients who want to hold hold in place, one account per payment, which is what the merging limitation already prescribes.

**Costs.** A program-enforced invariant becomes a client rule, on a scheme whose failure mode is self-harm. Fees on P go stale if the recipient defers for months, so overprovision P. Sweep-to-a-fresh-D stays available as a client mode at zero program cost. Reachable only after A, because it is the freedom A buys.

**Verify.** No new on-chain behaviour; it is the same instruction sequence with a different destination. The check is a privacy re-read, not a spike. Confidence: high on mechanics, medium on whether the owner wants to trade a program invariant for a client rule.

## B. No announcement account

Three variants, and the account survives all three. In `decryptable_zero_balance` (r1 idea 4) the scan becomes getProgramAccounts against Token-2022 itself, which paid RPCs restrict precisely because that program is the largest on the network, and the memcmp offset depends on a TLV layout we do not control. As a memo or a log, discovery needs transaction history, which public RPCs prune, so a recipient offline for six months loses the money permanently; discovery is the one artifact that must outlive arbitrary downtime, and account state is the only Solana medium that does. One announcement per run (r1 idea 5, Q11) either serialises N opens against one account or moves discovery after the run, so a payer crash between funding and announcing leaves money nobody can find. Worth taking: under A the announcement needs only E, payer, tag, k, bump and a version, about 69 bytes instead of 144, and its close is permissionless, gated on the token account being gone. Confidence: high. Verdict: shrink, do not delete.

## C. Registry without the torsion check, or no registry

The torsion check is input validation, not a security boundary: a registrant who publishes a mixed-order B_spend makes P unspendable and burns one payer's funds, and nothing about it lets them forge, repudiate or link anything. That reclassifies S7 from a design risk to a nice-to-have: if the curve25519 syscalls are awkward from `no_std` Pinocchio, ship without the check and say so. Keep it if the spike is easy, because the cost lands on a payer rather than on the registrant.

Dropping the registry is worse than decisions.md argues, for a reason decisions.md does not give: the registry is what lets a recipient rotate B_scan and B_spend without re-contacting every payer, which is the difference between a payroll integration and an address in a spreadsheet. Seeding the PDA on a label instead of a wallet, to blunt the r3 finding that the registering wallet is the deanonymiser, buys nothing: a throwaway registering wallet already is a label, and it keeps reverse lookup. Confidence: high. Verdict: keep both, demote S7.

## D. Payer creates the destination inside open

Dead, and cleanly. To configure D the payer needs a proof of knowledge of D's ElGamal secret, so D's key would have to come from S, so the payer holds read access to the recipient's destination permanently, not just to the one-time account. That turns the central limitation ("the payer reads one amount, once") into "the payer reads the recipient's account forever". The associated-token-account route is closed for a related reason: the ATA program creates with owner set to the wallet and ImmutableOwner on, foreclosing payer-side configuration. Confidence: high. Verdict: reject. Its one good instinct, that the payer already funds D's rent, is what alternative E removes outright.

## F. Issuer-approved mints without an approver service

`process_approve_account` checks only `authority_info.is_signer && key == confidential_transfer_mint.authority`, with no owner or token-account signature anywhere (`confidential_transfer/processor.rs`, ApproveAccount arm) [verified]. Two consequences. Approvals batch: one transaction approves tens of accounts, so an approver service is a stateless loop of maybe 100 lines, not a system. Better, that authority can be a program PDA. A sixty-line approver program with one policy ("approve any Token-2022 account on mint M whose owner is a PDA of program X") turns approval from a service the issuer operates into a CPI the payer makes inside its own transaction: no key custody, no latency, no second probe in the resume logic. The issuer acts once by pointing `confidential_transfer_mint.authority` at the PDA, and a hand-back instruction gated on a designated issuer key keeps it reversible. That is the one-page spec Q9 asks for, and an easier thing to put in front of Paxos than "run our daemon". Wrapping PYUSD into an auto-approving mint of our own is a pool; reject it. Under E the recipient's D disappears, which removes the r3 CRITICAL on these mints outright: only the one-time account ever needs approving. Confidence: high on mechanics, [open] on whether an issuer would flip the authority. Verdict: spec it, do not build it in v1.

## G. Discovery by derivation, no announcement, no scan (mine)

**Change.** One ephemeral scalar per payer, not per payment, published once in the payer's own registry entry. S = ECDH(e_payer, B_scan) is fixed per payer-recipient pair; payment i uses k = i, so P_i = B_spend + t_i·G. The recipient derives candidate addresses for k = 0, 1, 2, ... and stops after a gap, BIP-44 style, using getMultipleAccounts.

**Disappears.** The announcement account and its rent, the view tag, the getProgramAccounts scan, spike S6, the RPC-sees-your-tags leak, the PDA-squatting race, and the announcement close. The program drops to register_meta, close_meta, open. **Appears.** A gap-limit scanner and a per-payer E field, and a requirement that the recipient knows who pays it.

**Costs.** Donations from strangers break: the recipient cannot discover a payment from a payer it has never heard of. Payroll, invoices, grants and bounties survive, because in each the recipient knows the counterparty. It also deviates from every stealth-address standard, all of which are per-payment-ephemeral, raising review cost and weakening the sRFC-42 interop story. Public unlinkability is unaffected: without S the P_i look independent.

**Verify.** No chain behaviour to test. Re-derive the tree with fixed E and index k and put it in front of the Q1 reviewer, since it is the same question. Confidence: high that it works, medium that the trade is worth it.

## H. Ship the rail with no program at all, first (mine)

Plain mode needs zero on-chain code: a payer paying N recipients with confidential transfers is a CLI over Token-2022, and decisions.md already concedes recipient privacy is second-order to amount privacy. So the first demoable milestone is a batch engine with nothing deployed, and the program's reason to exist narrows to one sentence: offline setup of a one-time confidential account. Sequencing, not redesign, but it de-risks week 2 and sharpens the pitch. Confidence: high. Cost: none; the plan already builds the plain path first.

## Ranked recommendation

| Rank | Alternative | Call | Reasoning |
|---|---|---|---|
| 1 | A, hand ownership to P | adopt now | Deletes the largest instruction and takes the program out of the funds path, so "only the recipient can spend" stops being a claim about our code and becomes a property of Token-2022; costs a batchable reclaim loop. |
| 2 | F, approver as a policy program | adopt now as a spec | The authority can be a PDA and approvals need no owner signature [verified], which turns the issuer ask from "run a daemon" into "flip one field", at the cost of one page of writing. |
| 3 | B-lite, shrink the announcement | adopt now | 69 bytes instead of 144, permissionless close gated on the token account being gone; falls out of A for free. |
| 4 | C-lite, demote S7 | adopt now | A mixed-order B_spend is self-harm, so the torsion check is validation, not a boundary, and should not be able to block week 1. |
| 5 | E, drop the sweep destination | adopt after a spike | Removes D, five CPIs, 600 bytes and most of the SOL the payer hands over, but converts a program invariant into a client rule; needs A first. |
| 6 | H, plain mode with no program first | adopt now | Free; the plan already implies it, and saying it aloud gives week 2 a demo no spike can block. |
| 7 | G, derivation-based discovery | adopt after review | Deletes the announcement, the tag and the whole scan path, but breaks payments from unknown payers and leaves the standards track; same reviewer as Q1 and Q11. |
| 8 | C-full, drop the registry | reject | Rotation without re-contacting payers is the registry's real job and nothing else provides it. |
| 9 | B-full, delete the announcement | reject | Discovery must survive arbitrary recipient downtime; logs and memos prune, and account state is the only medium that does not. |
| 10 | D, payer creates the destination | reject | Configuring D requires knowing its ElGamal secret, so the payer would read the recipient's destination forever. |

## Is the decided design at a local optimum

Yes, and local is doing the work. Three rounds asked "what instructions does our program need" and answered well: the instruction set is minimal for the ownership model it assumed, every griefing window was closed by removing a boundary rather than guarding it, and each remaining guard traces to a line of Token-2022. No round asked "what must our program own", and the answer is: nothing, after open returns. Once the account belongs to P, sweep is not a simpler instruction, it is one that need not exist, and D is not a cheaper account, it is an account whose only job was to escape a key the recipient escapes by spending once. The design is one move from a smaller system with stronger custody properties, and that move costs a reclaim loop in the batch engine. Take it. G is a second, larger move in another direction, and it belongs behind the same cryptographic review Q1 and Q11 already wait on.
