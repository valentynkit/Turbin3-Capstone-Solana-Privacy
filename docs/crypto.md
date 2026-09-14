---
status: draft
last_verified: 2026-09-14
---

# Cryptography

TL;DR: standard stealth-address construction on Curve25519, plus derivations from the same shared secret for the Token-2022 confidential keys and, on the recipient's side only, for the sweep destination. The shared-secret derivations are the novel part and the source of every open concern. Recipe is source-verified; external review is still pending.

## Derivation tree

```
S      = ECDH(e, B_scan) = ECDH(b_scan, E)              reject all-zero S
PRK    = HKDF-SHA512-Extract(salt = "stealth-conf/v1", S)
t      = mod_wide(Expand(PRK, "tweak"      || E || k, 64))     P = B_spend + t·G
tag    = Expand(PRK, "view-tag"   || E || k, 1)
ct_ikm = Expand(PRK, "ct-ikm"     || E || k, 32)  ->  zk-sdk derive_confidential_keys_from_ikm(ct_ikm)
nonce  = Expand(PRK, "nonce-seed" || E || k, 32)  ->  hash prefix for raw-scalar Ed25519 signing

Recipient only:
d      = HKDF-SHA512(salt = "stealth-dest/v1", ikm = b_spend, info = "d-owner" || E || k)  ->  D_owner keypair
```

Why: RFC 5869 says distinct `info` under a PRF gives computationally independent outputs; that is the whole argument for the four shared-secret leaves, the same pattern as a TLS 1.3 key schedule. BIP-352 is precedent only for binding E and an index k into the tweak (`t_k = hash(ser(S) || ser32(k))`); it derives one value from S, not four. `k` is in every PDA seed from day one so that several accounts per handshake are additive later; v1 uses `k = 0`.

`derive_confidential_keys_from_ikm(ikm: &[u8]) -> Result<(ElGamalKeypair, AeKey)>` in solana-zk-sdk 8.0.0 does a second HKDF-SHA512 (salt `solana-conf-bal/v1`, info `ae` and `elgamal`) [verified, research: arch-r2-verified]. Feeding it `ct_ikm` rather than raw S keeps the stages separate.

The payer knows S and every leaf of PRK. `d` is derived from `b_spend`, which never touches S, so the payer cannot derive D_owner. [open: reviewer to confirm using the signing secret as HKDF input is acceptable, or to prescribe a dedicated recipient master secret]

## Curve25519 checks

- X25519 clamping neutralises low-order points; still reject S = 0 on both sides.
- Ed25519 has cofactor 8: the register instruction checks `L·B_spend = identity` with the curve25519 syscalls (validate 159 CU plus multiply 2,177 CU) [verified]. A blacklist of the eight small-order encodings is not a substitute: it misses mixed-order points.
- Keep B_spend (Edwards) and B_scan (Montgomery) as separate keys. Converting one key between representations has a sign ambiguity.
- Scalar reduction of 512 bits mod ℓ: bias below 2^-260, fine.
- The open instruction checks that P decompresses (159 CU); a bad P only wastes the payer's money, but the error is cheap.

## Signing with the one-time key

P is a real Ed25519 public key and the recipient holds its scalar p, so the recipient signs the sweep transaction as P and pays the fee from P's system account, funded by the payer. The runtime verifies the signature algebraically (RFC 8032, pure `verify`; the pending ZIP-215 migration is more permissive, not less) [verified]; the program only checks that P is a signer and matches the stored key.

Raw-scalar signing uses ed25519-dalek's hazmat interface, the one implicated in RUSTSEC-2022-0093. The nonce must be `H(prefix || M)` with `prefix` from the "nonce-seed" leaf, never the leaf itself and never a caller-supplied value: two signatures with the same R and different messages leak p to anyone watching the chain, and a retried sweep after a dropped transaction is exactly two different messages. One signing function for every code path, and a mandatory regression test that two messages signed by the same p produce different R.

## Balance cache

Token-2022 stores an AE-encrypted copy of the available balance as a hint. It is not authenticated on-chain, and the payer holds the same AE key. The client always decrypts the ElGamal ciphertexts with its own key (16-bit and 32-bit halves, precomputed table) and never trusts the cache.

## Disclosure receipt

`{E, k, ct_ikm, signature}`. The verifier re-derives the ElGamal and AES keys, checks the ElGamal pubkey against the proof instructions in the referenced transaction, and decrypts the amount. A forged `ct_ikm` fails the pubkey check, not a decryption. Disclosing `ct_ikm` hands over that one account's full read access: one credit, one sweep, nothing else, because every payment has its own S and PRK. It reveals nothing about t, the tag, the nonce, b_spend or other payments, given HKDF-Expand is a PRF [likely, pending the review below].

## Known concerns and assessment

| Concern | Assessment |
|---|---|
| Payer holds the account's ElGamal and AES keys permanently | Cannot move funds (the account is owned by P after open; p needs b_spend, which never touches S). Learns only the amount it paid and the public sweep event. The account is one-time by Token-2022's own credit counter. Accepted. |
| Payer can follow the funds it paid | True on any public ledger: account addresses are plain in every instruction. The payer already knows who it paid. What must not leak is the recipient's other income; that holds as long as payer-visible accounts are never merged with accounts other payers can see. |
| Does the ElGamal key leak p or b_spend | No: distinct Expand labels; b_spend independent of S |
| Key-validity proof binds an owner | No: it proves knowledge of the ElGamal secret only; Token-2022 separately requires the owner signature |
| View tag leaks a byte of S | Accepted, standard (Monero, ERC-5564); domain-separated from the tweak |
| HKDF domain separation correctness | Recipe above; needs a second pair of eyes [open] |

## Alternatives

- **Registry path.** Token-2022's ConfigureAccountWithRegistry skips the proof and owner signature if a registry whose owner equals the token-account owner exists. The recipient would have to create that registry per one-time key, online. Removes the payer's key knowledge; kills offline receiving. Documented option, not default.
- **One ephemeral key per run** (BIP-352 style, one E and N tags). Cuts recipient scan work by a large factor and groups every recipient of a run under one E. Deferred behind the external review.
- **Pre-published one-time keys** with pre-verified proof context accounts. Refill bursts correlate a recipient's accounts; needs an always-on service. Rejected.

## Reviewer checklist

- Four shared-secret labels never overlap; `ct_ikm`, not raw S, feeds the zk-sdk derivation; `d` is derived from `b_spend`, not S.
- Torsion check by multiply-by-L at registration; zero-S rejection both sides; P decompresses at open.
- Raw Ed25519 nonce is `H(prefix || M)`; regression test present.
- One-time use rests on `maximum_pending_balance_credit_counter = 1`, disabled public credits, and apply, transfer and empty in one transaction; all three cited in design.md. After open the token account is owned by P and closable only by the announcement PDA.
- Client never trusts the AE balance cache.
- Re-verify crate versions before shipping.

## Open questions

1. External review of the derivation tree including `d`. Owner: Valentyn. Resolve by: week 1. Recommendation: post the tree to the sRFC-42 discussion; ask one Anza zk contributor.
2. Resolved 2026-09-14: `derive_confidential_keys_from_ikm` exists in solana-zk-sdk 8.0.0 with the signature above.
