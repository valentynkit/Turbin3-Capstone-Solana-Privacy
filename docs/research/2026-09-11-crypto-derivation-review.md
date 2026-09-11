# Key derivation review (source-verified, 2026-09-11)

Sources read: zk-elgamal-proof zk-sdk derivation.rs / elgamal.rs / auth_encryption.rs / sigma_proofs/pubkey_validity.rs; token-2022 confidential instruction.rs / processor.rs; susruth/slnt; BIP-352; ERC-5564/6538; RFC 5869; Monero MRL-0006, PR #8061, issue #73.

## zk-sdk derivation today
```
prk        = HKDF-SHA512-Extract(salt="solana-conf-bal/v1", ikm)
ae_key     = HKDF-Expand(prk, info="ae", 16)
elgamal_sk = from_bytes_mod_order_wide(HKDF-Expand(prk, info="elgamal", 64))
```
`derive_confidential_keys_from_ikm(ikm)` (ikm 32..65535 bytes). **Published**: crates.io solana-zk-sdk 7.0.1 (2026-06-12) and npm @solana/zk-sdk 0.5.2 (`ConfidentialKeys.fromIkm`). `pda_wallet_public_seed` is on main + npm, not in crate 7.0.1.

## Proposed derivation tree
```
S        = ECDH(e, B_scan) = ECDH(b_scan, E)          reject all-zero S
PRK      = HKDF-Extract(salt="stealth-conf/v1", S)
t        = mod_wide(Expand(PRK, "tweak"    || E || k, 64))   P = B_spend + t·G
view_tag = Expand(PRK, "view-tag" || E || k, 1)
ct_ikm   = Expand(PRK, "ct-ikm"   || E || k, 32)  → derive_confidential_keys_from_ikm(ct_ikm)
nonce    = Expand(PRK, "nonce-seed" || E || k, 32) → hash_prefix for raw-scalar Ed25519 signing (binds nonce to p)
```
RFC 5869 §3.2: distinct `info` under a PRF gives computationally independent outputs. Binding E and index k follows BIP-352 (`t_k = hash(ser(S) || ser32(k))`).

## Concerns and assessment
- (a) Sender knows the account's ElGamal + AES keys permanently. Cannot forge sweeps (needs p = b_spend + t; b_spend never touches S). PubkeyValidityProof binds only the pubkey, no owner; Token-2022 separately requires the owner signature. In a strict one-time model the sender learns nothing beyond the amount it sent and the public sweep event. Accepted trade-off if one-time semantics are enforced; alternative below.
- (b) ElGamal secret does not leak p or b_spend (distinct Expand labels; b_spend independent of S).
- (c) mod-ℓ reduction of 512 bits: bias ≤ 2^-260, fine.
- (d) X25519 clamping neutralises low-order points; check S ≠ 0. Ed25519: check `B_spend.is_torsion_free()` at registration (cofactor 8).
- (e) Raw-scalar signing via ed25519-dalek hazmat: nonce must be bound to the scalar (RUSTSEC-2022-0093, "Taming the many EdDSAs"). Use the "nonce-seed" label.
- (f) Token-2022 does not require the proof generator to be the owner. OK.
- (g) Shared AES key adds no new leak beyond (a).
- (h) 1-byte view tag under its own label; Monero issue #73: must be domain-separated from the scalar derivation.
- Keep B_spend (Edwards) and B_scan (Montgomery) as separate keys; converting one key between representations has a sign ambiguity.

## Alternatives to remove (a)
1. **Pre-published one-time ElGamal keys** with pre-verified PubkeyValidity context-state accounts: any party can reference them in ConfigureAccount (context authority only governs close). Cost: refill bursts and exhaustion correlate a recipient's accounts; needs an always-online refill service.
2. **`ConfigureAccountWithRegistry`**: skips proof and owner signature if an ElGamalRegistry whose owner == token_account.owner exists. Recipient must have created the registry for that specific one-time owner (needs the secret + online). Eliminates sender key knowledge but requires recipient liveness per payment. Registry creation flow not independently re-checked (unverified).

## Reviewer checklist
- Four labels never overlap; ct_ikm output, not raw S, feeds the zk-sdk derivation.
- Torsion-free B_spend; zero-S rejection both sides.
- Raw Ed25519 nonce bound to p.
- Explicit decision on (a): accepted one-time trade-off vs registry path.
- Re-verify crate/npm versions before shipping.

## Novel vs standard
ECDH-tweak stealth construction is standard (BIP-352 / ERC-5564 / Monero subaddress family). Novel: folding a second derivation purpose, the Token-2022 confidential-balance keys, into the same shared secret, which no surveyed design does; that is also the source of concern (a).
