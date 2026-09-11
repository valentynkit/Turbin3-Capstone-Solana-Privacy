# sRFC notes (verified via gh api, 2026-09-11)

Source: https://github.com/solana-foundation/SRFCs/discussions

| # | Title | Updated | Notes |
|---|---|---|---|
| 15 | DRAFT: RWA Asset Classification and Data Discovery (pzupan/ioniqx) | 2026-08-24 | SAS-based `rwa.class`/`rwa.claim` vocabulary; nonce = mint for indexer-free discovery; findings: SAS has no pubkey type, no revoke (closeAttestation deletes; expiry is the control). Seeks co-sponsors. Extends stalled sRFC 00020 (RWA security token standard, stalled on NAV/valuation). Repo https://github.com/pzupan/rwa-classification |
| 14 | sRFC 43: Receive Terms and Held Delivery (EfeDurmaz16) | 2026-08-20 | Receiver publishes terms; non-conforming value routes to PDA-owned guard vault + receipt instead of reverting. Fixed 6-account `can_credit` interface (explicitly rejects extra-account-metas). Open: under DefaultAccountState=Frozen, guard PDA can't be thawed until sRFC 37 decides how gate thaws PDA owners. Ref impl https://github.com/EfeDurmaz16/solana-receive |
| 13 | sRFC-0042: Silent Payments for Solana (susruth) | 2026-06-11 | Stealth addresses: meta-address → per-payment one-time address; Pinboard (announcement program, `post`/`post_batch`, view tags) + Registry (wallet→meta-address, PDA ["meta", registrant, scheme_id]). Anchor reference programs, candidate IDs SLNTPD…/SLNTRC…. Not audited. |
| 2 | sRFC 37: Token ACL Standard (tiago18c) | 2026-07-13 | DefaultAccountState=Frozen + Token ACL program as delegated freeze authority + Gate Program (allow/block). Audited; targeted Q1 2026 ship. Exo-tech: "sRFC 37 much better than TransferHooks"; hooks = "account dependency hell", most protocols blacklist hooked mints. Open issue (Jul 2026): Phantom does NOT auto-append ThawPermissionless → transfers to allow-listed wallets with no ATA fail 0x11. Open Q: multisig freeze authority, PDA-owned accounts (protocol vaults). Guide: https://solana.com/developers/guides/advanced/acl |
| 10 | sRFC 40: Vault Standard Program | 2026-05-20 | ERC-4626-like vault standard, 9 comments. |
| 9 | Solana Agent Protocol (SAP) | 2026-07-27 | Hardware identity + DePIN + STARK proofs for agents. |
| 4 | sRFC 39: Clear Sign | 2026-06-11 | |
| 3 | sRFC 38: Offchain Message Spec v1 | 2026-06-04 | |

Also verified: https://github.com/solana-program/token-2022/issues/66 — `invoke_transfer_checked` OOM with 2 hooked transfers — OPEN, 0 comments since Jan 2025.

## Implications
- Compliance direction is shifting hooks → ACL (frozen-by-default + permissionless thaw). The unsolved composability problem moves to: **protocol-owned PDAs (AMM vaults, escrows, guard vaults) as holders of permissioned tokens** — who KYCs a PDA, how does a gate thaw it, how does a router create+thaw ATAs mid-transaction. Both sRFC 43 and the ACL thread flag this as open.
- sRFC 42 + Token-2022 confidential balances have never been combined: unlinkable *and* amount-hidden payments. Stealth ATAs + confidential transfer + auditor key = privacy payroll with compliance escape hatch.
- Classification/attestation layer (SAS) is being built by lone issuers asking for co-sponsors; Foundation RWA group involvement signals a real appetite.
