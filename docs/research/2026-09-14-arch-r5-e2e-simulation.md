---
status: draft
last_verified: 2026-09-14
---

# Arch R5: end-to-end simulation

TL;DR: the flow closes for the happy path and for every failure run below; every actor has what it needs when it needs it. The gaps are in docs, not in the construction: meta-address handoff, cross-run E collisions, P's floor under fee spikes, and the plain recipient's pubkey source are all unstated.

Confidential-transfer behavior below is checked against `solana-program/token-2022` `main` on 2026-09-14: `processor.rs` (base) and `extension/confidential_transfer/processor.rs`. Marked [verified] where read directly.

Run: `run_secret` generated once, source balance 1,000 USD-equivalent (encrypted, no plaintext ever on chain). Split A=500, B=300, C=200. Order: tx1 A (plain), tx2 B (stealth), tx3 C (stealth). `i` indexes stealth recipients only: B=0, C=1.

## 1. Recipient B registers, hands out a meta-address

| Actor | Action | Inputs (source) | Signer | Accounts | Lamports | Risk |
|---|---|---|---|---|---|---|
| B | generate `b_spend` (Ed25519), `b_scan` (X25519) locally | CSPRNG on B's machine | - | - | - | key file loss = re-register, no funds at risk |
| B | `register_meta(spend_pub, scan_pub)` | `spend_pub = b_spend·G`, `scan_pub = b_scan·G` | B's wallet | MetaAddress PDA `["meta", wallet]` (created), System | wallet pays rent-exempt minimum for 86 bytes, not stated in docs, plus one fee | `L·spend_pub != identity` rejected (cofactor-8 torsion) [verified: design.md cites the curve25519 syscall check]; `scan_pub = 0` rejected; re-registering from the same wallet fails, PDA already in use |
| B | hands out `(wallet, B_spend, B_scan)` as its meta-address | - | - | - | - | the handoff channel itself (QR, link, email) is undocumented, see (b) |

## 2. Payer pays A (plain), B and C (stealth) in one run

Payer holds 1,000 in one confidential source account. ElGamal/AES keys for that account were derived at the payer's own shield time (out of scope here).

**Per-recipient key material.**

| Recipient | Ephemeral | Shared secret | One-time key | View tag | Confidential keys | Dest owner |
|---|---|---|---|---|---|---|
| A (plain) | none | none | none, existing address | none | A's own, already on chain | none |
| B | `e_0 = HKDF(run_secret, run_id\|\|0)`, `E_0 = e_0·G` | `S_0 = ECDH(e_0, B_scan)`, reject if 0 | `t_0` from `PRK_0`; `P_0 = B_spend + t_0·G` (payer computes the point, not the scalar) | 1 byte from `PRK_0` | `ct_ikm_0 -> derive_confidential_keys_from_ikm` (payer can compute, has `S_0`) | payer cannot compute; needs `b_spend` |
| C | `e_1 = HKDF(run_secret, run_id\|\|1)`, `E_1` | `S_1 = ECDH(e_1, C_scan)` | `P_1 = C_spend + t_1·G` | 1 byte | same | payer cannot compute |

Payer reads A's ElGamal pubkey from A's on-chain confidential token account (public field in the extension state) to build the transfer proof. Not stated in either doc which RPC call does this; assumed `getAccountInfo` on the address A supplied [likely, not verified in docs].

**Transaction 1, A (plain).** Signer: payer only.
```
SetComputeUnitLimit
VerifyCiphertextCommitmentEquality   over payer's source available-balance ciphertext
VerifyBatchedGroupedCiphertext3HandlesValidity   over the 500-unit transfer amount, handles for payer/A/auditor
VerifyBatchedRangeProofU128          amount in range, no overflow
Transfer  payer source -> A's existing account, offsets to the three proofs above
```
Source balance before: Enc(1000). After: Enc(500). What can go wrong: A's account has `allow_confidential_credits = false` (A disabled it) — Transfer rejected [verified: the flag exists and gates confidential credits, per processor.rs]; stale balance ciphertext if the payer used two source accounts concurrently — proof commits to the wrong on-chain state, transfer reverts, atomic, no funds move.

**Transaction 2, B (stealth).** Signer: payer only. `P_0` and the PDAs are addresses, not signers here.
```
SetComputeUnitLimit
VerifyPubkeyValidity        proves payer knows the ElGamal secret behind the pubkey derived from S_0, for ConfigureAccount
VerifyCiphertextCommitmentEquality
VerifyBatchedGroupedCiphertext3HandlesValidity
VerifyBatchedRangeProofU128
open   StealthAccount PDA["stealth",E_0,0] + token account PDA["ta",E_0,0]:
       System allocate+assign both PDAs
       InitializeImmutableOwner, InitializeAccount3(owner=StealthAccount PDA), ConfigureAccount(elgamal pubkey from S_0, offset to pubkey-validity proof, max_pending_credit_counter=1)
       DisableNonConfidentialCredits
       System transfer payer -> P_0 (D-rent + fees + priority allowance + retry margin, ~0.0045-0.006 SOL [open, unmeasured])
Transfer  payer source -> new token account, offsets to the three proofs above, amount 300
```
Source balance before: Enc(500). After: Enc(200). What can go wrong: `E_0` collides with a prior run's ephemeral key (both PDAs already exist) — `open` fails, no guard against cross-run reuse is documented, see (b); P_0 fails to decompress — cheap reject per crypto.md; ConfigureAccount is called before `Transfer`, so the 1-credit cap is armed before any funds move, matching design.md's griefing argument.

**Transaction 3, C (stealth).** Same shape, `i=1`, amount 200. Source balance before: Enc(200). After: Enc(0).

## 3. B scans, finds the payment, sweeps

`scan`: `getProgramAccounts(program, dataSize=144, memcmp(0,[2]), memcmp(138,[tag_0]))`. B recomputes `S_0 = ECDH(b_scan, E_0)` per candidate, re-derives the tag, checks `B_spend + t_0·G == P_0`, derives confidential keys, decrypts pending balance = 300.

**Sweep transaction.** Signers: `P_0` (fee payer, B holds scalar `p_0`) and `D_owner` (derived by B from `b_spend, E_0, 0`, never on chain before this).
```
SetComputeUnitLimit
VerifyPubkeyValidity for D            D's ElGamal pubkey, derived by B same as the account's key
System create D (P_0 pays rent, 469 B, 4,155,120 lamports)
InitializeImmutableOwner, InitializeAccount3(owner=D_owner), ConfigureAccount(D_owner signs), DisableNonConfidentialCredits
VerifyCiphertextCommitmentEquality, VerifyBatchedGroupedCiphertext3HandlesValidity, VerifyBatchedRangeProofU128   for the sweep transfer
VerifyZeroCiphertext                  over the source token account's balance computed as it will be after the transfer
sweep   drain_to = D_owner:
        ApplyPendingBalance (source token account, owner = StealthAccount PDA via invoke_signed, no real signature needed [verified: validate_owner accepts a PDA authorized by the calling program's invoke_signed, no confidential instruction requires the token account itself to sign])
        Transfer full available balance, source -> D, 3 proofs
        EmptyAccount, zero-ciphertext proof checked against the account's live post-transfer ciphertext [verified: EmptyAccount compares the proof's ciphertext to confidential_transfer_account.available_balance and requires closable()]
        CloseAccount source token account -> lamports to StealthAccount.payer (the original payer wallet), 4,155,120 lamports
        close StealthAccount PDA -> same payer wallet, 1,893,120 lamports
        System transfer P_0's remainder -> D_owner
```
State of D after each stage: created empty (0 tokens, confidential extension configured, public credits disabled) -> after `Transfer` inside `sweep`, pending balance = Enc(300), available balance still 0 (Token-2022 lands confidential credits as pending, never available, until an apply [verified: `pending_balance_lo/hi` update on Transfer, `available_balance` untouched]). D is not yet spendable.

What remains on chain after this transaction: D (owned by D_owner, pending balance 300, no SOL fee reserve beyond what P_0 forwarded), StealthAccount and old token account gone, P_0 at 0 lamports (allowed for a fee payer [verified per design.md], effectively garbage-collected). Payer wallet balance is up 6,048,240 lamports (both rents).

What can go wrong: signer is not `P_0` or not `D_owner` — rejected; StealthAccount `state != Live` (already swept) — rejected; `destination == source` guard trips if D derivation collided with the source PDA, practically impossible; `expected_pending_balance_credit_counter` on ApplyPendingBalance is stored but never checked [verified], so a wrong value here is inert, not a risk.

## 4. Payer checks refunds; B spends from D

Payer watches its own wallet (journal + `getSignaturesForAddress`, or a balance delta) and sees +6,048,240 lamports per swept stealth payment, tagged to the CloseAccount/close instructions in B's sweep transaction, which is public. Payer also sees D and D_owner addresses in that transaction and can follow D's future activity (amounts encrypted), matching privacy.md's stated payer capability.

To spend from D, B needs: SOL for fees (from the leftover P_0 forwarded to D_owner at sweep time — the only funding D_owner has), and an `ApplyPendingBalance` on D signed by D_owner before any further confidential Transfer, since the swept funds landed as pending. If B skips apply, a spend attempt fails: Token-2022's Transfer draws from `available_balance`, which is still 0.

What the payer sees when B later spends: a public transaction moving funds out of D to some address, amount encrypted unless B makes it a non-confidential transfer (D's `allow_non_confidential_credits` was disabled at creation for inbound dust, but that flag does not block outbound public transfers, which is a separate Token-2022 code path — not tested here, flagged in (b)).

## 5. B discloses payment 2 to an auditor

Payment 2 is B's stealth payment (tx2). B sends the receipt `{E_0, 0, ct_ikm_0, sig}` (B's own transaction signature or the payer's tx2 signature — docs do not say which signature the receipt should reference, see (b); the payer's open+Transfer tx is the one carrying the proofs, so that is the only signature that lets `verify` check the pubkey and decrypt).

`verify`: re-derives ElGamal/AES keys from `ct_ikm_0`, checks the derived ElGamal pubkey against the pubkey-validity proof in the referenced transaction's instructions, decrypts the transfer amount ciphertext to 300. Works after B's one-time account is closed, since it reads historical instruction data, not account state; needs an archival RPC.

What can go wrong: forged `ct_ikm` fails the pubkey check, not a silent bad decryption [verified in crypto.md's own text, source-grounded claim about the derivation, not independently re-verified against Token-2022 source here]; wrong tx signature (e.g. B's own sweep tx instead of the payer's) gives a pubkey mismatch since the proofs live in the payer's transaction, not B's.

## 6. Failure runs

| Scenario | What happens | Recovery |
|---|---|---|
| Payer process dies after tx2, before tx3 | tx1 (A) and tx2 (B) are already final on chain, funds are safe. Journal has fsynced entries for i=0,1. On restart the engine reads the journal, sees i=2 (C) missing, regenerates C's proofs against the current on-chain source balance (200), sends tx3. If `run_secret`'s encrypted file was also lost, the payer can never again disclose payments under this run (needs `e_i` to recompute `S_i`), but B and C can still self-disclose since `E_i` is public on chain and each recipient derives `S_i` from its own `b_scan` | client.md's journal + run_secret model; consistent |
| B's sweep fails, P_0 underfunded | `sweep` simulation fails locally before send (client checks P or payer can cover fee + rent, per client.md, but that check runs at *pay* time against the budget the payer chose, not at sweep time against the live priority-fee market) | if congestion raised the fee since `open`, B's only top-up path is a wallet B controls sending SOL to P_0 or D_owner, which links that wallet to the payment — documented residual leak in privacy.md, not a program failure |
| Stranger sends dust to the token account between payer tx and sweep | A confidential credit is blocked: `maximum_pending_balance_credit_counter = 1` was already consumed by the payer's own Transfer, so Token-2022 rejects a second confidential credit [verified: ConfigureAccount sets the counter cap; design.md ties the cap directly to this exact griefing scenario]. A non-confidential (public) credit is blocked by `DisableNonConfidentialCredits` [verified: `allow_non_confidential_credits` is set false; design.md attributes exactly this purpose to the instruction, not independently re-checked against the ordinary Transfer code path that reads it]. Raw SOL sent via System Transfer to the token account's address is not gated by Token-2022 at all — harmless, it just rides along into the CloseAccount rent refund to the payer | no action needed; the counter and the disabled flag are the actual defenses, not something the client polls for |
| Payer's tx2 (B) fails on a stale blockhash, tx3's (C) proof set was already generated | tx2 never lands; source balance stays Enc(500). tx3's proofs were built assuming the post-tx2 balance Enc(200) — client.md says proofs are generated "just before each send," sequential per source account, precisely to prevent this. If pipelined anyway, tx3's equality proof commits to a ciphertext that does not match the account's actual `available_balance` at execution, Token-2022 rejects it, transaction reverts, atomic, no funds lost, only wasted client CPU | payer must regenerate C's proof set against confirmed post-tx2 state before resending; this is why the sequential rule exists, not a defect found here |

## (a) Does it close end to end

Yes for every step simulated, including all four failure runs: no actor is ever asked to produce an input it cannot derive from its own secrets plus public chain state. The one soft spot is disclosure signature choice (5) and cross-run key-material RPC lookups (2), both process gaps rather than cryptographic ones.

## (b) Where the docs are silent

- Meta-address handoff channel (step 1): no specified transport (QR, link, DNS record). Client.md and brief.md assume it exists but never name it.
- Rent-exempt cost of a MetaAddress account (86 bytes) is never stated, unlike the StealthAccount and token account figures in design.md's Money section.
- How the payer obtains a plain recipient's ElGamal pubkey for a direct confidential Transfer (step 2, tx1): design.md and client.md never describe the plain-recipient code path at all, only "the same without the pubkey proof and open."
- No documented guard against `E_i` colliding across separate runs (different `run_secret`, same recipient) with a low but nonzero chance both derive the same point; `open`'s PDA-exists check turns this into a hard failure for the second run rather than a silent overwrite, but nothing detects it before submission.
- Which transaction signature belongs in a disclosure receipt (step 5) when payer and recipient each sign a different transaction — the recipient's own sweep carries no proofs, only the payer's transfer does, but this is never spelled out.
- P's underfunding is checked once, at pay time, against the fee market then; nothing re-checks it at sweep time against a possibly higher live priority fee (step 6, scenario 2).
- Whether `DisableNonConfidentialCredits` on D also blocks D's *outbound* public transfers or only inbound credits (step 4) is unaddressed; Token-2022's flag semantics were not checked for the outbound path in this simulation.
- Payer reclaim of a never-swept account after a timeout is explicitly deferred to plan.md (design.md, Money section) but no interim behavior (does the payer's capital just sit locked indefinitely with no way to check progress besides its own journal?) is described.

## (c) Where Token-2022 source contradicts the docs

None found. Every specific behavioral claim checked against `processor.rs` and the confidential-transfer `processor.rs` on 2026-09-14 matched design.md and crypto.md: ConfigureAccount requires owner signature and enables both confidential and non-confidential credits by default [verified]; ApplyPendingBalance requires owner signature, resets the pending counter and pending balances, and stores but never validates `expected_pending_balance_credit_counter` [verified]; Transfer requires source owner signature and updates the available balance by homomorphic subtraction [verified]; EmptyAccount requires a zero-ciphertext proof matched against the live balance and `closable()` [verified]; CloseAccount requires a zero base amount, checks `closable()` for the confidential extension, and returns lamports to the given destination [verified]; InitializeAccount3 and InitializeImmutableOwner require no signature from the owner field, only that the account is program-owned [verified] — this is exactly what lets the program set `owner = StealthAccount PDA` without the PDA ever holding a private key.

## (d) Three weakest points

1. **P's funding is fixed at open time, spending happens minutes to days later.** The only underfunding recovery is a recipient-controlled top-up that links a wallet, which is the exact leak the whole design exists to prevent. I would size P's floor with a wider safety margin than a point-in-time p75 sample, or let the payer refresh P's balance in a later transaction the recipient requests without revealing which wallet asked.

2. **The plain-recipient code path is undocumented.** It is the one branch missing from both design.md and client.md, described only by omission ("the same without the pubkey proof and open"). It is also the only branch that talks to a recipient's ElGamal pubkey without ever deriving it locally, so it is worth writing down explicitly rather than inferring.

3. **Payer capital recovery after an abandoned stealth payment has no answer beyond "phase two."** Every never-swept account locks 0.006 SOL of the payer's money with no documented timeout, no documented monitoring path, and no documented owner. For a 50-recipient run this is small; for a payroll-scale operator running this rail continuously it accumulates as an unbounded, unrecoverable float. I would treat this as a blocker for the "success criteria" batch-convergence claim in brief.md, not a phase-two nicety, since convergence of a batch that leaves money permanently stuck is not full convergence.
