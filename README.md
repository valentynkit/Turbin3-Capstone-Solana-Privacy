# solana_privacy

A confidential payout rail on Solana: a visible payer pays many recipients with encrypted amounts, and recipients who want it get a fresh, unlinkable account per payment that the payer can set up alone. The money stays an ordinary Token-2022 token and an auditor can still read amounts. Built from two existing halves that were never joined, stealth addresses and Token-2022 confidential balances, with no pool, no relayer, no MPC, no enclave. Turbin3 Q3 2026 Builders Cohort capstone.

Status: design and research phase, no code yet. Two Anchor programs and a TypeScript CLI are planned; see `docs/plan.md`.

Docs live in `docs/`. Start at `docs/README.md` for the map and the reading order per role.

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or <https://www.apache.org/licenses/LICENSE-2.0>)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or <https://opensource.org/licenses/MIT>)

at your option.

### Contribution

Unless you state otherwise, any contribution you intentionally submit for inclusion in this
work, as defined in the Apache-2.0 license, is dual licensed as above, with no additional
terms or conditions.
