---
status: draft
last_verified: 2026-09-11
---

# Cryptography

TL;DR: standard stealth-address construction on Curve25519, plus one extra derivation from the same shared secret for the Token-2022 confidential keys. That extra derivation is the novel part and the source of every open concern. Recipe below is source-verified; external review is still pending.

## Derivation tree

```
S      = ECDH(e, B_scan) = ECDH(b_scan, E)              reject all-zero S
PRK    = HKDF-SHA512-Extract(salt = "stealth-conf/v1", S)
t      = mod_wide(Expand(PRK, "tweak"      || E || k, 64))     P = B_spend + t·G
tag    = Expand(PRK, "view-tag"   || E || k, 1)
ct_ikm = Expand(PRK, "ct-ikm"     || E || k, 32)  ->  zk-sdk derive_confidential_keys_from_ikm(ct_ikm)
nonce  = Expand(PRK, "nonce-seed" || E || k, 32)  ->  hash prefix for raw-scalar Ed25519 signing
```

Why: RFC 5869 says distinct `info` under a PRF gives computationally independent outputs. Binding E and an index k follows BIP-352 (`t_k = hash(ser(S) || ser32(k))`) and allows several one-time accounts per handshake. The zk-sdk function itself does a second HKDF (salt "solana-conf-bal/v1", info "ae" and "elgamal"); feeding it `ct_ikm` rather than raw S keeps the stages separate.

## Curve25519 checks

- X25519 clamping neutralises low-order points; still reject S = 0 on both sides.
- Ed25519 has cofactor 8: check `B_spend` is torsion-free at registration, or P inherits ambiguity.
- Keep B_spend (Edwards) and B_scan (Montgomery) as separate keys. Converting one key between representations has a sign ambiguity.
- Scalar reduction of 512 bits mod ℓ: bias below 2^-260, fine.

## Signing with the one-time key

Solana's Ed25519 precompile verifies the RFC 8032 equation only, so a signature made with the raw scalar p is accepted. ed25519-dalek's hazmat interface signs with a raw scalar but is the interface implicated in RUSTSEC-2022-0093 (double public key signing oracle): the nonce must be bound to p. Use the "nonce-seed" label, never a fixed or caller-supplied value.

In-program verification reads the instructions sysvar: check the sysvar address is canonical, the referenced instruction's program is the Ed25519 program, the pubkey / message / signature match exactly, and the instruction index is fixed. This is the Wormhole 2022 checklist.

## Known concerns and assessment

| Concern | Assessment |
|---|---|
| Sender holds the account's ElGamal and AES keys permanently | Cannot forge sweeps (needs p; b_spend never touches S). In a strict one-time model learns only the amount it sent and the public sweep event. Accepted trade-off, enforced by one-time semantics. [decided, open to reversal after review] |
| Does the ElGamal key leak p or b_spend | No: distinct Expand labels; b_spend independent of S |
| Key-validity proof binds an owner | No: it proves knowledge of the ElGamal secret only; Token-2022 separately requires the owner signature |
| View tag leaks a byte of S | Accepted, standard (Monero, ERC-5564); must be domain-separated from the tweak |
| HKDF domain separation correctness | Recipe above; needs a second pair of eyes [open] |

## Alternatives

- **Registry path.** Token-2022's ConfigureAccountWithRegistry skips the proof and owner signature if a registry whose owner equals the token-account owner exists. The recipient would have to create that registry per one-time key, online. Removes the sender's key knowledge; kills offline receiving. Documented option, not default.
- **Pre-published one-time keys** with pre-verified proof context accounts, consumed by senders. Any party can reference them. Refill bursts correlate a recipient's accounts; needs an always-on service. Rejected for v1.

## Reviewer checklist

- Four labels never overlap; `ct_ikm`, not raw S, feeds the zk-sdk derivation.
- Torsion-free B_spend; zero-S rejection both sides.
- Raw Ed25519 nonce bound to p.
- Explicit decision on sender key knowledge, recorded in decisions.md.
- Re-verify crate and npm versions before shipping.

## Open questions

1. External review of the derivation tree. Owner: Valentyn. Resolve by: week 1. Recommendation: ask the sRFC-42 author and one Anza zk contributor; post the tree to the sRFC discussion.
2. Is `pda_wallet_public_seed` needed, or is our own seed enough? It is in npm but not in crate 7.0.1. Recommendation: own seed; treat the helper as reference.
