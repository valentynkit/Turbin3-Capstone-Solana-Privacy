---
status: draft
last_verified: 2026-09-14
---

# Architecture R1, the obvious build

TL;DR: one Pinocchio program with two account families, a Rust batch client as the primary tooling, and a generated TypeScript SDK for the demo. All proofs go through context-state accounts, funding is a plain Token-2022 transfer with no program call, and the only thing the program guarantees is the state machine and the signature of the one-time key.

Confidence tags: [verified] seen in a cited source or a repo research file that cites one; [likely] strong inference; [open] unknown. This file diverges from `design.md` in four places, each marked **divergence**.

## 1. Program topology

One program, `payd`, with two account families (meta-address registry, stealth account lifecycle). **Divergence** from `design.md`, which specifies two Anchor programs.

The registry is never read on-chain. The stealth path cannot check `P = B_spend + t*G` without the shared secret, so it never loads a `MetaAddress` [verified: the program has no access to S, crypto.md]. Its only consumer is the off-chain payer client, over RPC. A second program buys a CPI boundary nobody crosses and costs a second deploy, upgrade authority, keypair, and hand-written IDL. Nothing cross-references, so a later split stays cheap.

Program ID: one vanity keypair from `solana-keygen grind`, pubkey embedded as `pub const ID: Address` [verified: pinocchio.md], keypair stored outside the repo. Upgrade authority is the owner's deploy keypair; deploys go through a buffer so a failed upload does not brick the live program. Never `--final` during six weeks. Handover if this reaches mainnet: 2-of-3 Squads multisig [likely: design-patterns.md].

## 2. Account model

Zero-copy rule: every field is a byte array, integers little-endian as `[u8; N]` behind accessors. Pinocchio's `AccountDeserialize` casts `&data[2..]`, past the 1-byte discriminator and 1-byte version [verified: pinocchio.md], so alignment is 1 at best and any `u64` field would be an unaligned reference. All-byte-arrays makes `align_of == 1` and the cast sound. `assert_no_padding!` on both structs.

Rent below is `(128 + len) * 6960` lamports, which reproduces the research figure of 0.00416 SOL for a 469-byte confidential token account [verified: token2022-mechanics-verified].

### MetaAddress, discriminator 0x01, version 0x01

PDA seeds `["meta", wallet]`. Created and closed by the recipient wallet, which pays rent.

| Offset | Field | Size |
|---|---|---|
| 0 | discriminator | 1 |
| 1 | version | 1 |
| 2 | spend_pub, compressed Edwards | 32 |
| 34 | scan_pub, Montgomery u | 32 |
| 66 | scheme_id, LE | 2 |
| 68 | flags, bit0 = active | 1 |
| 69 | bump | 1 |
| 70 | reserved | 16 |

Total 86 bytes, rent 1,489,440 lamports (0.00149 SOL). The wallet is not stored: `rotate` and `close` prove ownership by deriving the PDA from the signer.

### StealthAccount, discriminator 0x02, version 0x01

PDA seeds `["stealth", E]`, where E is the 32-byte ephemeral public key. Created by the payer, who pays rent; closed permissionlessly once swept, rent to `rent_refund`.

| Offset | Field | Size |
|---|---|---|
| 0 | discriminator | 1 |
| 1 | version | 1 |
| 2 | ephemeral_pub E | 32 |
| 34 | one_time_pub P | 32 |
| 66 | mint | 32 |
| 98 | token_account | 32 |
| 130 | rent_refund | 32 |
| 162 | created_slot, LE | 8 |
| 170 | view_tag | 1 |
| 171 | state: 0 Opened, 1 Configured, 2 Swept | 1 |
| 172 | bump | 1 |
| 173 | flags | 1 |
| 174 | reserved | 16 |

Total 190 bytes, rent 2,213,280 lamports (0.00221 SOL). `view_tag` at offset 170 and `dataSize = 190` are the scan filter. Fixed-size fields only, so those offsets stay stable across upgrades [verified: design-patterns.md].

### Accounts the program does not own

- **Token account**: PDA `["ta", E]`, assigned to Token-2022, pre-allocated at 469 bytes for base + ImmutableOwner + ConfidentialTransferAccount [verified: token2022-mechanics-verified]. `owner` field is the StealthAccount PDA. Rent 0.00416 SOL, paid by the payer, reclaimed on close. Created with allocate plus transfer-of-deficit plus assign, not `create_account`, to dodge the 1-lamport pre-funding grief [verified: design-patterns.md].
- **P's system account**: no data, funded by the payer at open. It must hold the rent-exempt floor for a fee payer (890,880 lamports) plus the transient proof rent the sweep creates (0.00854 SOL) plus fees. Call it 0.0096 SOL, and 0.014 if P also has to create the sweep destination (section 6). **Divergence**: `design.md` calls this "fee dust". It is not dust; it is the largest single item in per-payment cost. [likely, arithmetic over verified rent figures]
- **Proof context accounts**: keypair accounts owned by the ZK ElGamal proof program, 65 / 161 / 385 / 297 bytes for pubkey-validity / equality / validity / range [verified: token2022-mechanics-verified]. Transient, closed in the same transaction that consumes them.

Per stealth payment the payer locks roughly 0.0159 SOL and recovers roughly 0.0150 [likely].

## 3. Instruction set

Single-byte discriminator. Naming follows subject_verb_object [verified: design-patterns.md].

| # | Name | Accounts | Data | Guards | CPI |
|---|---|---|---|---|---|
| 0 | `recipient_register_meta` | wallet (s, w), meta (w), system | spend_pub 32, scan_pub 32, scheme_id 2 | PDA from signer; spend_pub decompresses and is torsion-free; scan_pub nonzero; not already initialized | System allocate/assign |
| 1 | `recipient_rotate_meta` | wallet (s), meta (w) | same | PDA from signer; active | none |
| 2 | `recipient_close_meta` | wallet (s, w), meta (w) | none | PDA from signer | none |
| 10 | `payer_open_stealth` | payer (s, w), stealth (w), token_account (w), mint, P sysacct (w), token_2022, system | E 32, P 32, view_tag 1, rent_refund 32, dust_lamports 8 | stealth PDA from E, canonical bump; ta PDA from E; mint is a Token-2022 mint with ConfidentialTransferMint; uninitialized | System create for both PDAs, Token-2022 `InitializeImmutableOwner`, `InitializeAccount3` (owner = stealth PDA), System transfer to P |
| 11 | `payer_configure_stealth` | payer (s), stealth (w), token_account (w), mint, pubkey_validity_ctx, token_2022 | decryptable_zero_balance 36, max_pending_counter 8 | state == Opened; token_account matches record; ctx owned by proof program | Token-2022 `ConfigureAccount`, stealth PDA signs via `invoke_signed` [verified: no on-curve check in `validate_owner`, token2022-mechanics-verified] |
| 12 | `recipient_sweep_stealth` | P (s, fee payer), stealth (w), token_account (w), mint, destination (w), equality_ctx, validity_ctx, range_ctx, token_2022 | apply_new_decryptable 36, transfer_new_source_decryptable 36, expected_pending_counter 8 | state == Configured; signer key == stored P; token_account matches; destination != token_account | Token-2022 `ApplyPendingBalance` then `Transfer`, both signed by the stealth PDA |
| 13 | `anyone_close_stealth` | closer (s, w), stealth (w), token_account (w), mint, rent_refund (w), zero_balance_ctx, token_2022 | none | state == Swept; rent_refund matches record | Token-2022 `EmptyAccount` then `CloseAccount`, stealth PDA signs; then `account.close()` on the stealth PDA |

**Divergence**: there is no `fund` instruction. Funding a stealth account is an ordinary Token-2022 confidential transfer from the payer, identical to the plain-recipient path. The program cannot stop a third party from transferring in anyway, so an on-chain `fund` records a fact it cannot enforce while adding a CPI, a transaction, and a second copy of the transfer code. One transfer module now serves both modes. One-time semantics survive as a client convention plus the Opened to Configured to Swept ratchet.

**Proof threading.** Every proof goes through a context-state account, because instruction-offset proofs resolve against top-level instructions and our calls sit behind a CPI [verified: design.md, traced to token-2022 source]. Per proof: a prior transaction creates the account (System create, owner = ZK ElGamal proof program) and runs `VerifyX` with a context-state account. The context authority is the payer for configure and funding and P for the sweep, so our program never signs a context close. Our instruction takes the context read-only and forwards it into the Token-2022 CPI at the expected index; Token-2022 reads the context data directly, with no CPI to the proof program. `CloseContextState` runs top-level in the same transaction as the consuming call.

**Error codes** (`ProgramError::Custom`): 1 BadDiscriminator, 2 WrongState, 3 NotOneTimeKey, 4 TokenAccountMismatch, 5 MintMismatch, 6 BadSeeds, 7 AlreadyInitialized, 8 NotTorsionFree, 9 InvalidPoint, 10 RentRefundMismatch, 11 BadProofContext, 12 DestinationIsSource, 13 MetaInactive, 14 UnexpectedProgramId. One test per code.

## 4. Key derivation placement

Everything cryptographic runs client side: ECDH, the HKDF tree, the tweak, P, the ElGamal and AES keys, all proofs [verified: crypto.md].

The program verifies exactly four things: the StealthAccount PDA is derived from E with the canonical bump; the token account PDA is derived from E; the sweep signer's key equals the stored P (the runtime already verified the Ed25519 signature algebraically, including raw-scalar ones [verified: token2022-mechanics-verified]); and the state ratchet.

The program cannot verify that P is the tweak of a registered `B_spend`, that the ElGamal key installed by configure was derived from S, that the view tag is correct, or that the account was ever funded. None of these need on-chain verification: getting any of them wrong produces an account the recipient never finds or never sweeps, which is indistinguishable from the payer not paying. The only on-chain curve work is the torsion-free check on `B_spend` at registration, via the curve25519 syscalls, which turns a permanent fund-loss bug into a rejected registration [likely, CU cost unmeasured].

## 5. Payer batch client

Primary language: **Rust**. `plan.md` already carries "proof generation too slow in TypeScript, client rewrite in Rust" as a live risk, and starting in Rust removes the branch. The owner writes Rust and no frontend; `solana-zk-sdk` 7.0.1 is the reference implementation of the derivation [verified: crypto-derivation-review]; Mollusk and LiteSVM are Rust, so layouts and seeds come from one shared crate; and a crash-safe long-running batch process with a local journal is not a comfortable TypeScript program. The teammates' TypeScript SDK, generated from a hand-written Codama IDL, covers register, scan, and the demo, not the batch engine.

Modules: `config`, `keyring`, `registry`, `derive` (the HKDF tree, shared with the recipient client), `proofgen` (bounded worker pool over `solana-zk-sdk`), `planner`, `packer` (transaction assembly, ALTs, compute budget), `submitter`, `journal`, `reclaim`.

Run state machine, one append-only fsynced journal entry per (run_id, recipient_index, step): `Plan`, `Prepare`, `Stage`, `Execute`, `Reclaim`, `Done`. Resume re-reads the journal and re-derives every address, because the ephemeral secret is deterministic: `e_i = HKDF(run_master_secret, run_id || i)`. Same seed, same E, same PDAs, so a crash mid-run costs nothing and a retried `open` hits AlreadyInitialized, which the client treats as success. Never reuse a run_id.

Proof staging: every proof is generated before any transaction is signed, so the 150-block blockhash window never expires during CPU work; transactions are signed immediately before sending and re-signed on expiry. The 1.4 KB range proof does not fit a transaction and is staged through an `spl-record` account in two [verified: token2022-mechanics-verified].

Lookup tables: one ALT per run holding the static program IDs, the mint, the payer's token account, and both PDAs per recipient, up to 256 entries. Warmed one slot ahead, deactivated at `Reclaim`. ALTs buy transaction size, not account locks [verified: design-patterns.md].

Packing: recipients are independent, so the client pipelines with bounded concurrency rather than packing across them. Batching amortises proof CPU and the ALT, not transaction count [verified: design.md]. Per stealth recipient, 6 to 7 transactions: one setup transaction (create pubkey-validity context, verify, open, configure, close context), falling back to two if it exceeds 1232 bytes, then four transfer-proof transactions, then the transfer. Plain recipients are 5 [verified: token2022-mechanics-verified].

Accounting: a pre-flight ledger per recipient (rent, dust, transient rent, fees) and a post-run reconciliation against the journal. `SetComputeUnitLimit` per transaction from simulation, configurable priority fee.

## 6. Recipient client

Same Rust crate, `payd scan` and `payd sweep`, sharing `derive`.

Scan: `getProgramAccounts` with `dataSize = 190`, `memcmp` on the discriminator at offset 0 and the view tag at offset 170, which cuts candidates to one in 256 server side [verified: standard RPC filter; view tag per crypto.md]. Per candidate: one X25519 for S, re-derive the tag to drop false positives, derive t, recompute `B_spend + t*G` and compare to the stored P, then derive the ElGamal and AES keys and decrypt. Adequate to roughly 10^5 program accounts; a geyser indexer keyed on view tag is the scaling answer and is out of scope.

Sweep: apply pending and transfer out in one instruction, P signing and paying, then close.

Destination policy is the weak point. `privacy.md` says "sweep to a fresh self-owned account", but a fresh account still has to be created and funded by somebody, and if the recipient's own wallet pays, the chain shows wallet W creating D and then the one-time account sweeping into D. That links the payment to W and undoes the scheme. The only funding source that does not link is P. So the recipient generates a fresh keypair D_owner off-chain, and P, already holding payer-provided lamports, creates and funds D's token account while D_owner signs the configure. Nothing publicly tied to the recipient appears. Cost: another 0.00416 SOL left on P. The client refuses a destination it has seen before, with an explicit override flag [Q4 recommendation, plan.md].

## 7. Relayer

Not in v1, and probably not ever. P pays its own fees and transient proof rent from lamports the payer left, so the default flow has no fee-payer gap [verified by construction, decisions.md 2026-09-14]. A relayer is needed only if the payer declines to fund P, and it costs the exact privacy property the design is built on, since the relayer learns the mapping from one-time account to destination [verified: privacy.md]. It also adds an always-on service to a six-week project that has none. Revisit only if measured dust makes payer funding commercially unacceptable, and then as an optional plug-in, not a dependency.

## 8. Test pyramid

| Harness | Sole role |
|---|---|
| Mollusk | Compute-unit benchmarks with the markdown differ, one bench per instruction, run in CI to catch CU regressions [verified: testing.md] |
| LiteSVM | The default gate. Every functional test, including full flows through Token-2022 and the ZK ElGamal proof program. Rust, in-process, fast [verified: testing.md] |
| Surfpool | Integration and demo network: RPC-dependent client paths (`getProgramAccounts` scanning, ALTs), transaction profiling, and the recorded demo. Embedded SDK, not the CLI daemon [verified: testing.md] |
| Devnet | Release smoke only, one happy path per instruction, never a PR gate [verified: testing.md] |

If the schedule slips, Mollusk goes first, since LiteSVM also reports CUs.

Adversarial suite, all on LiteSVM: sweep signed by any key other than P; sweep before configure, twice, and after close; close before sweep; close with a non-zero balance using a forged zero-balance context; a proof context of the right size but the wrong proof type, wrong authority, or borrowed from another payment; a foreign token account with the right mint substituted at sweep; a counterfeit StealthAccount owned by another program with a matching discriminator; a non-canonical bump; duplicate mutable accounts; one-lamport pre-funding of a PDA before open; close then re-open the same E; a fake system or Token-2022 program ID; transfer spam pushing the pending-balance counter to its maximum; and a second valid Ed25519 signature for P, to confirm the program never verifies a signature itself.

## 9. Build, deploy, CI

```
programs/payd/       pinocchio program
crates/payd-core/    no_std: layouts, seeds, discriminators, errors, shared with the program
crates/payd-crypto/  derivation tree, no chain deps, test vectors from slnt
crates/payd-client/  run engine, proofgen, journal, scanner
crates/payd-cli/     binary `payd`
clients/ts/          Codama-generated client plus a thin demo SDK
codama.ts            hand-written IDL, since Pinocchio emits none
tests/               litesvm suites, mollusk benches
```

Pinned: `pinocchio` 0.11.2, `pinocchio-system` 0.6.1, `pinocchio-token-2022` 0.3.1, `pinocchio-log` 0.5.1 [verified: pinocchio.md, 2026-07]; `solana-zk-sdk` 7.0.1 [verified: crypto-derivation-review]; `litesvm` 0.14.x, `mollusk-svm` 0.14.x, `surfpool-sdk` 1.5.0, `@solana/kit` 7.x [verified: testing.md]; `@solana/zk-sdk` 0.5.2, `@solana-program/token-2022` 0.17.0, `@solana-program/zk-elgamal-proof` 0.4.0 [verified: token2022-mechanics-verified]. The Rust confidential-transfer stack (`spl-token-2022`, `spl-token-confidential-transfer-proof-generation`) is listed in the skill reference against `solana-zk-sdk` 5.0.0, which contradicts 7.0.1, so the compatible set is resolved by building on day one and pinned then [open]. `rust-toolchain.toml` and the Solana platform-tools version are both pinned; every crate carries `license = "MIT OR Apache-2.0"`.

CI, GitHub Actions, three jobs: fmt plus `clippy -D warnings`; `cargo test-sbf` running the LiteSVM suite and Mollusk benches; and a serial vitest job with embedded Surfpool for the TypeScript client. Devnet smoke is a manual workflow.

## 10. Where I am not confident

1. **Pre-sized token account plus ConfigureAccount with no reallocate.** The reference flow reallocates first [verified: confidential-transfers.md]. Settle with a LiteSVM spike, or by reading `process_configure_account`'s `init_extension` path.
2. **ZK ElGamal proof program availability in LiteSVM and on devnet.** The skill reference says confidential transfers run only on a TXTX cluster; `brief.md` says mainnet since mid-2026. Both cannot be current. Settle by running one `VerifyPubkeyValidity` in each, day one. This gates the demo plan.
3. **Context-state authority rules** when the token account owner is a PDA. Settle by reading `verify_and_extract_context` and by a substitution test.
4. **P draining itself to zero while acting as fee payer.** The fee-payer rent check runs before execution, so it should work. LiteSVM spike. If it fails, 890,880 lamports per payment are stranded.
5. **Dust size per P.** My arithmetic gives 0.0096 SOL, 0.014 with the destination, against `design.md`'s "fee dust". Measure a full sweep in week 2. This decides whether the rail is commercially sane.
6. **Destination funding leak** (section 6). A design decision, not a measurement; needs the same external review as the derivation tree.
7. **Torsion-free check cost** via curve25519 syscalls, which Pinocchio does not wrap. Mollusk bench.
8. **`getProgramAccounts` with a one-byte memcmp at scale.** Measure response size and latency on devnet against 1,000 seeded accounts.

## 11. Brief refinements

The draft above stays aligned with `brief.md` as written. Four places where changing the brief would make the architecture better. None touch positioning or the claims table.

1. **Scope, section 7: "two Anchor programs ... a TypeScript client".** Both are now wrong: the owner picked Pinocchio, the registry is never read on-chain (section 1), and the batch client is Rust (section 5). Proposed: "one Pinocchio program, a Rust batch client, a generated TypeScript SDK". Why: a reader of the brief alone would plan the wrong work.

2. **Limitations, section 6: "About 0.013 SOL locked per payment until close".** Omits the lamports the payer must leave on P for the sweep's transient proof rent (0.00854 SOL alone) and for the destination account. Proposed: "roughly 0.02 SOL locked per stealth payment, most of it recoverable, pending week-2 measurement". Why: this is the number a payroll operator multiplies by headcount.

3. **Limitations, section 6: add the destination-funding leak.** "Consolidation re-links" covers sweeping several accounts into one place, not the simpler break: a recipient wallet that creates and funds the sweep destination links the payment to a public identity on its own. Proposed addition: "The sweep destination must be funded from the one-time account, never from a wallet the recipient can be identified by." Why: it is a hard constraint that costs the payer real lamports, so it belongs with the other stated-up-front limits.

4. **Success criteria, section 8: "A payout run to N recipients on devnet".** Hinges on the ZK ElGamal proof program being enabled on devnet, which is uncertainty 2 and unverified. Proposed: "on a public cluster where the ZK ElGamal proof program is enabled, with Surfpool as the recorded fallback". Why: a success criterion should not rest on an unverified fact about someone else's cluster, and `plan.md` already keeps a local-SVM fallback.
