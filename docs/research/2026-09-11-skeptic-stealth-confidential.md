# R3 — design-skeptic on P2 (2026-09-11)

## FATAL
- **F1 Sweep is the unlinkability killer.** Sweep destination = recipient's persistent main confidential account, cleartext on-chain → two sweeps cluster two "unlinkable" payments. Classic stealth consolidation leak (BIP-352 has it too). Not listed in 09's leak list. Mitigations: spend directly from stealth PDAs (never consolidate), sweep to a fresh self-stealth account via the same protocol, delayed/batched sweeps. Must be disclosed as residual.
- **F2 Sender keeps a permanent decryption key** (ElGamal sk derived from ECDH secret sender also knows). Tolerable only if accounts are strictly one-time and swept promptly; contradicts any account reuse (batch payroll reuse). Enforce/document "one-time, sweep-then-close".

## SERIOUS
- **S1 Is the PDA trick necessary?** Check `ConfigureAccountWithRegistry`: registry is per *owner*; a fresh stealth owner would need its own registry signed by that owner → sender can't. A single persistent registered ElGamal pubkey reused across stealth accounts links them immediately (worse). Real research question: recipient's key must not be reused across payments AND sender must not hold it. (Possible direction: recipient pre-publishes one-time ElGamal pubkeys with pre-verified PubkeyValidityProof context-state accounts; sender consumes one. Linkability of those pre-created accounts to the recipient is the catch.)
- **S2 HKDF domain separation** between sRFC-42 tweak derivation and ElGamal-key derivation from the same ECDH secret is unverified; "zk-sdk accepts arbitrary IKM" is an API note, not a security argument. Needs applied-crypto review before scope is final.
- **S3 Payroll is the wrong headline.** Employer already knows the employee; stealth adds nothing there. Value is where the sender↔recipient relation is worth hiding from observers: pseudonymous B2B invoicing, grants/bounties, donations, creator payouts.
- **S4 Cost:** ~10 txs per recipient (announce, configure w/ PubkeyValidityProof, confidential transfer with equality + validity + range proof context accounts, apply pending, sweep = another confidential transfer, close). 100-person payroll ≈ 1,000 txs per run. Directionally disqualifying for payroll.
- **S5 sRFC-42 dependency buys nothing** (1 comment, devnet, dormant). Implement the standard stealth construction natively; cite sRFC-42 + BIP-352 as prior art.

## MINOR
- M1 11 handlers + 4 off-chain tools too much when 3 primitives are unverified; cut batch_payout/payroll SDK.
- M2 No wallet support → CLI demo, plan for it.

## Reshaped scope
Unlinkable, amount-hidden one-off payments between pseudonymous counterparties. Own minimal stealth implementation, Pinboard announcement program, PDA-owned stealth account funded by one confidential→confidential transfer, sweep with discrete-log proof, sweep-linkage documented as known residual with "sweep to fresh throwaway account" mitigation.
Before more scope: (1) confirm ConfigureAccountWithRegistry can't replace the ECDH key; (2) get HKDF domain-separation reviewed.

**Biggest unresolved question:** can registry-based configuration replace the ECDH-derived ElGamal key? Determines whether the "novel" piece is real crypto needing sign-off or unnecessary.
