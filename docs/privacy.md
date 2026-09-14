---
status: draft
last_verified: 2026-09-14
---

# Privacy

TL;DR: hidden from the public: who received, how much. Visible: who paid, that fresh accounts were paid, and, if the recipient merges accounts, which accounts belong together. The payer can follow the funds it paid, as on any public ledger, but reads no amount after the sweep. An auditor reads amounts by disclosure. No formal analysis yet.

## Adversaries

| Adversary | Capabilities | Goal |
|---|---|---|
| Chain observer | reads every account and transaction | link payments to a recipient; learn amounts |
| Payer | knows S for its own payments, holds that account's decryption keys, sees P, D and D_owner in the sweep | learn the recipient's other income; move funds; block the sweep |
| Recipient | all its secrets | may leak by merging accounts |
| Auditor | mint key or disclosed receipts | read amounts (intended) |
| Mint authority | ApproveAccount policy, freeze authority | gate or freeze accounts (a property of the mint, not of the rail) |
| RPC provider | sees every query | learn which view tags a scanner asks for, from which IP |

## Artifacts per payment

| Artifact | Public | Links to recipient without b_scan | Amount |
|---|---|---|---|
| MetaAddress | yes, once | it is the recipient's registered wallet | none |
| Payer's transfer to a plain recipient | yes | yes, fixed address | encrypted |
| Payer's transaction: StealthAccount (E, tag, P, k), token account, P funded, transfer in | payer identity, one-time account | no, needs S | encrypted; auditor readable |
| Recipient's transaction: D created by P, apply, transfer, empty, reclaim to the payer, P drained to D_owner | yes | no; links P, D and D_owner to each other | encrypted |
| A later spend from D | yes | no for the public; the payer can follow it | encrypted if confidential |
| A sweep or spend into an account another payment also reached | yes | **yes**, merges everything that touched it | encrypted |
| Timing of the payer's transactions in a run | yes | correlates payer with one-time accounts and counts them, not with recipients | none |

## Residual leaks and mitigations

- **Merging.** Inherent to stealth addresses (BIP-352 has it). Sweep into a fresh D per payment and never merge accounts that different payers can see. The client refuses a destination it has seen before unless overridden; this is client policy, not a program invariant. [open: MPC consolidation, phase three]
- **The payer follows the funds.** The payer sees D in the sweep and can watch D's counterparties forever, with amounts hidden. This is what any payer on a public ledger can do with a wallet it paid, and the payer already knows who it paid. Honest limit: this rail hides the recipient from the public, not from the payer.
- **Payer's keys.** The payer can read that one account until it is emptied. It cannot move funds (P owns the account), cannot decrypt the destination side, and cannot block the sweep: the credit counter rejects any second credit, public credits are disabled, and P's key is the recipient's alone. A partial spend straight from the one-time account shows the payer the split; sweep the whole balance.
- **Balance cache.** The AE-encrypted balance hint is writable by anyone holding the AES key, which includes the payer. The client never trusts it.
- **Registration.** Registering a meta-address makes the registering wallet public and permanent, and that wallet is what a payer looks up. For payroll it costs nothing; for a grants or donations flow, register from a wallet with no other history.
- **Underfunded P.** If the payer leaves too little on P, the only top-up path is a wallet the recipient controls, which links it. The client refuses to send below a computed floor and says so; the payer overprovisions.
- **Headcount and timing.** A run's transactions land close together from one payer: an observer counts recipients per run with certainty. Batching and delays help; not solved.
- **Scanning.** A view-tag filter reveals the tags a scanner asks for to the RPC provider, together with the IP. Run your own RPC or accept it, as BIP-352 and Zcash light clients do.
- **Program fingerprint.** Calling this program, or confidential-transfer instructions at all, marks a wallet as privacy-seeking while usage is low. Applies to every privacy tool on Solana.
- **Mint authority.** On approval-required mints the issuer sees and gates every account; a freeze authority can freeze a one-time account and block its sweep. A property of the mint.
- **View tag.** One byte of the shared-secret hash is public. Standard.

## What we do not claim

- Payer privacy. Use a pool.
- Privacy from the payer.
- Unlinkable consolidation.
- Privacy from the auditor or from a party a receipt was disclosed to.
- Resistance to a global adversary correlating timing across many payments.
- Anonymity of any kind. This is not a mixer and must never be described as one.
