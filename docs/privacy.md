---
status: draft
last_verified: 2026-09-14
---

# Privacy

TL;DR: hidden from the public: who received, how much. Visible: who paid, that fresh accounts were paid, and, if the recipient reuses a destination, which accounts belong together. Readable by an auditor: amounts. No formal analysis yet.

## Adversaries

| Adversary | Capabilities | Goal |
|---|---|---|
| Chain observer | reads every account and transaction | link payments to a recipient; learn amounts |
| Payer | knows S for its own payments and that account's decryption key | learn the recipient's other income; move funds |
| Recipient | all its secrets | may leak by consolidating |
| Auditor | mint key or disclosed per-payment keys | read amounts (intended) |
| Relayer, if used | pays fees | learn which accounts it swept |

## Artifacts per payment

| Artifact | Public | Links to recipient without b_scan | Amount |
|---|---|---|---|
| MetaAddress | yes, once | it is the recipient | none |
| Payer's transfer to a plain recipient | yes | yes, fixed address | encrypted |
| StealthAccount (E, view tag, P) | yes | no, needs S | none |
| Payer's transfer into a fresh account | payer identity, one-time account | no | encrypted; auditor readable |
| P's system account funded by the payer | yes | no | dust |
| Sweep to a fresh self-owned account | yes | no | encrypted |
| Sweep to a reused main account | yes | **yes**, links every account swept there | encrypted |
| Timing of open, configure, fund | yes | correlates payer with the one-time account, not with the recipient | none |

## Residual leaks and mitigations

- **Consolidation.** Inherent to stealth addresses (BIP-352 has it). Sweep into a fresh self-owned account, spend from there. Client refuses reused destinations by default. [open: MPC consolidation, phase three]
- **Payer's key.** The payer can read that one account until close, and can decrypt the source side of any transfer out of it. Hence: one-time accounts, sweep-then-close enforced by the program, and never spend directly from a payer-funded account.
- **Timing.** Open, configure, fund land together. Payer-to-account link only. Batching and delays help; not solved.
- **Program fingerprint.** Calling the stealth program, and calling confidential-transfer instructions at all, marks a wallet as privacy-seeking while usage is low. Accepted; it applies to every privacy tool on Solana.
- **View tag.** One byte of the shared-secret hash is public. Standard.
- **Fees.** P pays its own sweep fee from dust the payer left, so the recipient's wallet never appears. If a relayer is used instead, it learns the mapping.

## What we do not claim

- Payer privacy. Use a pool.
- Unlinkable consolidation.
- Privacy from the auditor or from a party a key was disclosed to.
- Resistance to a global adversary correlating timing across many payments.
- Anonymity of any kind. This is not a mixer and must never be described as one.
