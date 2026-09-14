---
status: decided
last_verified: 2026-09-14
---

# Client

TL;DR: one Rust CLI for both sides. The payer's batch engine derives everything from a run secret and sends sequentially per source account; the recipient scans by view tag and sweeps in one transaction of plain Token-2022 instructions signed by the one-time key. Split from design.md on 2026-09-14.

## Batch engine

Everything the run creates derives from a 32-byte `run_secret`: `e_i = HKDF(run_secret, run_id || i)`. Losing it before every refund has landed or before an audit is answered is data loss; it is stored in an encrypted file and the CLI says so at creation. A journal holds one fsynced line per (run_id, i, step) with signature, last valid block height and outcome, never a secret. A run id is never reused: the same (run_secret, run_id, i) gives the same E, and open then fails as already initialized, which is the safety property, not a bug.

Plain recipients: the payer reads the recipient's confidential account, takes its ElGamal pubkey from the extension, and builds the transfer; no instruction of ours. Stealth recipients: the payer derives everything from the meta-address and the run secret and adds open.

Each Transfer proof binds the payer's source balance after the previous transfer, so sends are sequential per payer source account; parallelism means several source accounts. Pubkey-validity proofs are generated ahead in parallel; the three transfer proofs are generated just before each send. Measured on the dev machine with solana-zk-sdk 8.0.0: range proof 16 ms, the other three under 0.5 ms, a 50-recipient proof set in 0.2 s of wall clock [verified, research: arch-r3-verified]. Confirmation latency, not proving, bounds throughput.

Priority fee per run from a live `getRecentPrioritizationFees` sample at p75 with a hard cap and pause-and-alert, never silent retries. Before every send the client simulates and checks the payer can cover both the fee and every rent transfer inside that transaction; only an included transaction burns a fee. `lamports_for_P` is sized from the same sample; the recipient may sweep much later, so the margin is generous rather than tight.

Refunds arrive by themselves when recipients sweep. `pay --status` lists announcements still open per run; nothing else for the payer to do.

## Recipient flow

Nothing on the recipient side is persisted: D_owner is derived from `b_spend` and (E, k), so a crash at any point is a retry, the same property the run secret gives the payer.

`scan`: `getProgramAccounts` on our program with `dataSize = 69`, `memcmp(0, [2])`, `memcmp(66, [tag])`; per candidate one X25519, re-derive the tag, recompute `B_spend + t·G` and compare it with the owner field of the token account at `["ta", E, k]`, then derive the confidential keys and decrypt the pending balance from the ElGamal ciphertexts. Fine to about 100k accounts on a paid RPC; an indexer beyond that is out of scope.

`sweep`: builds the one transaction in design.md, signed by P (raw scalar, fee payer) and D_owner. D is created with CreateAccountWithSeed from D_owner so it adds no signer. Before sending, the client simulates and checks P can cover the fee and D's rent; if P is short it stops and says by how much, because the only top-up path links the recipient (privacy.md). It refuses a destination it has seen before unless overridden, and offers `--spend-to` for a whole-balance payment straight to a counterparty instead of D.

Any wallet that can sign Token-2022 confidential instructions with an imported key can do the same sweep without this CLI; only the rent reclaim at the end needs our program, and funds do not depend on it.

## Mints

Works unmodified on mints whose `auto_approve_new_accounts` is true; open refuses others, and `pay --issuer-path` is not implemented in v1. PYUSD and USDG are Token-2022 with the extension but require the Paxos authority's ApproveAccount for every new confidential account; USDC is legacy SPL Token with no extension [verified 2026-09-14 on mainnet]. The issuer path, documented and not built: open keeps the announcement PDA as owner when the mint requires approval; after the issuer approves, a second payer instruction applies any pending dust, hands the account to P, and runs in the same transaction as the transfer. The issuer's side can be one field: set the mint's confidential-transfer authority to the PDA of a small policy program that approves every account, since ApproveAccount needs only that authority's signature [verified, research: arch-r5-redesign-search].

## Audit

Disclosure receipt `{E, k, ct_ikm, payer transaction signature}`. `verify` re-derives the ElGamal and AES keys from `ct_ikm`, checks the ElGamal pubkey against the proof instructions in the referenced transaction, and decrypts the amount. Works after the accounts are closed; needs an archival RPC. A mint auditor key, where the issuer sets one, is the secondary path.

## Testing

LiteSVM is the gate; it loads the ZK ElGamal builtin by default [verified]. Mollusk with the `all-builtins` feature for CU assertions in CI. Surfpool embedded for scanning and the recorded demo. Devnet for release smoke, funded early. Golden-byte tests for every hand-written Token-2022 encoder against the interface crate's builders. Chaos harness: kill the client at every step of a 20-recipient run and assert convergence. Adversarial suite: transfer, apply or empty signed by any key but P after handover; public credit after open; second confidential credit; reclaim before the balance is zero; counterfeit announcement; wrong mint; non-canonical bump; a mint that does not auto-approve; two P signatures share no R.

## Stack

pinocchio 0.11.2; solana-zk-sdk 8.0.0; spl-token-2022-interface 3.1.1 for discriminants and golden bytes; ed25519-dalek hazmat for raw-scalar signing; litesvm 0.16, mollusk-svm, surfpool sdk; v1 transactions via solana-message and solana-transaction [verified present]. Every crate `license = "MIT OR Apache-2.0"`.
