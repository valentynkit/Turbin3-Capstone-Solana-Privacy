---
status: draft
last_verified: 2026-09-11
---

# Privacy

TL;DR: hidden from the public: who received, how much. Visible: who sent, that a fresh account was paid, and, if the recipient consolidates, which accounts belong together. Readable by an auditor: amounts. No formal analysis yet.

## Adversaries

| Adversary | Capabilities | Goal |
|---|---|---|
| Chain observer | reads every account and transaction | link payments to a recipient; learn amounts |
| Sender | knows S for its own payments, the one-time account's decryption key | learn the recipient's other income; move funds |
| Recipient | knows all its secrets | none against itself; may leak by consolidating |
| Auditor | mint auditor key or disclosed per-account keys | read amounts (intended) |
| Relayer | pays fees for sweeps | learn which account belongs to whom |

## Artifacts per payment and what they link

| Artifact | Public | Links to recipient without b_scan | Amount |
|---|---|---|---|
| MetaAddress | yes, once | it is the recipient | none |
| StealthAccount (E, view tag, P) | yes | no; needs S | none |
| Token account ciphertexts | yes | no | encrypted; auditor readable |
| Funding transaction | sender identity, one-time account | no | encrypted |
| Sweep transaction | one-time account to destination | yes, if the destination is a reused main account | encrypted |
| Timing of open and fund | yes | correlates sender with the one-time account, not with the recipient | none |
| Relayer-submitted sweep | relayer identity | relayer learns the mapping it submits | none |

## Residual leaks and mitigations

- **Consolidation.** Inherent to stealth addresses. Spend directly from one-time accounts, or sweep into a fresh one via the same protocol. CLI default: sweep to fresh. [open: MPC-held balance for consolidation, phase three]
- **Sender's key.** Sender can read that one account's balance until close. One-time semantics enforced: fund once, sweep, close.
- **Timing.** Open, configure, fund land close together. Sender-to-account link only. Batching and delays help; not solved.
- **View tag.** One byte of the shared secret hash is public. Standard, accepted.
- **Relayer.** Learns the accounts it sweeps. Use your own fee payer if that matters.

## What we do not claim

- Sender privacy. Use a pool.
- Unlinkable consolidation.
- Privacy from the auditor.
- Resistance to a global adversary correlating timing across many payments.
- A formal proof. This is a documented model, to be reviewed.
