# Solana privacy systems, verified against primary docs (2026-09-11)

| System | Hides sender | Hides recipient | Hides amount | Funds stay standard tokens | Trust beyond Solana | Auditor / compliance | Status |
|---|---|---|---|---|---|---|---|
| Hinkal | yes | yes | yes | no (vault PDA custodies) | TEE (GCP SEV-SNP enclave does proving + key custody), relayer, Groth16 setup (unverified) | Chainalysis KYT, selective disclosure, zkKYC > $10k | mainnet; on-chain program closed source |
| Umbra (Solana, umbra-defi; unrelated to Ethereum Umbra) | partial (mixer tree) | yes | yes (Arcium MPC balances) | no (custodied) | Arcium MPC majority, relayer, indexer, Groth16 | hierarchical viewing keys (mint→year→month→day) | mainnet-beta, no audit found, program closed source |
| Helius Rings | default: no; anonymous custom ring: yes | same | yes | no (escrow PDA) | prover server; delegated decryption (provider reads balances); relayer for anonymous | viewing keys, per-ring auditor, freeze, allow/block lists | devnet only, audits in progress, Apache-2.0 (helius-labs/zolana) |
| Privacy Cash | partial (amount/timing traceable) | public at withdrawal | no | no | relayer sees recipient + amount; 4-party Groth16 ceremony | deposit screening only | mainnet, audited, BSL |
| Light Protocol PSP (2022) | yes | yes | yes | no | relayer, Groth16 | none | dormant since 2022-11 |
| Arcium CSPL | unverified (likely no) | unverified (likely no) | claimed | yes (account-based) | Cerberus MPC, permissioned clusters | none documented | not shipped (stub crate, single-commit examples) |
| Token-2022 confidential balances alone | no | no | yes | yes | none; local proofs | global mint auditor key | mainnet, audited |
| sRFC-42 stealth alone | no | yes | no | yes | none | none | draft, devnet, dormant |

## What is distinct about stealth × Token-2022 on program-owned one-time accounts
- No anonymity set: each payment is private on its own. All five pool systems need a shared Merkle tree.
- Funds never leave standard-token form; composable immediately, no unshield step.
- No relayer, MPC, TEE, or prover server in the core; local proof generation.
- Protocol-level audit path (mint auditor) plus per-account key disclosure.
- Sender visible by design (the trade-off).
Only the *combination* is new; each half exists.

Sources: hinkal.io, hinkal-team.gitbook.io, github.com/Hinkal-Protocol; umbraprivacy.com, sdk.umbraprivacy.com, github.com/umbra-defi; helius.dev/privacy, docs.helius.dev/docs/privacy, github.com/helius-labs/zolana; privacycash.org, github.com/Privacy-Cash, github.com/0xVector0/privacycash-analyzer; github.com/Lightprotocol/light-protocol-v1; arcium.com, docs.arcium.com, github.com/arcium-hq/c-spl-example-programs.
