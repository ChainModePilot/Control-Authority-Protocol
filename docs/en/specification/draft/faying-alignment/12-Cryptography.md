# CAP Final §6 — Cryptography (Normative draft)

> **Status**: Drafted clause for H1-PROTO-01, Wave 2 (uplift of current Ch.8 + **new suite negotiation & PQC reservation, closing G-9**). Defines algorithm suites, suite negotiation, key formats, signature-input canonicalization, key distribution / storage / rotation, and the post-quantum evolvability reservation. Modeled on `Faying-Protocol-1.0` §6.
> **Style**: hard constraints `C-Crypto-<n>`, `Trace: R<n>.<m>.<k>` to `02`. This chapter is **independent of the open PO decisions** (no `[依赖 D-n]`).

---

## §6.1 Algorithm suites & negotiation

CAP v1 mandates two signature suites and reserves a post-quantum negotiation slot (parity with Faying §6.1 / G8).

| Use | Default | Alternative | PQC negotiation slot (reserved) |
|---|---|---|---|
| Signing (`Authorization_Descriptor` / `Trusted_Ticket` / `RevocationStatement`) | **Ed25519** (RFC 8032) | ECDSA-P256-SHA256 (RFC 6979) | **ML-DSA-65** (FIPS 204) |
| Hashing | **SHA-256** | — | SHA3-256 |
| Channel layer | TLS 1.3 (mandatory) | mTLS (optional) | TLS-1.3-PQ (hybrid X25519 + ML-KEM-768) |

JWS algorithm names for online tickets: `ed25519 → EdDSA`, `ecdsa-p256-sha256 → ES256`.

**Negotiation rules**:

- **C-Crypto-1 (mandatory minimum set)**: every conformant implementation SHALL support both `{Ed25519, ECDSA-P256-SHA256}`; an implementation SHALL NOT reject a peer merely because the peer does not support a PQC suite.
  - **Trace**: `R11.1`.
- **C-Crypto-2 (negotiate strongest common)**: at version/suite negotiation (reusing the §1.6 negotiation path), each side declares its supported signature / hash / channel suites; the strongest jointly-supported value is selected. If no common signature or hash exists, reject (`E_PROTOCOL` / `E_PROTOCOL_VERSION_UNSUPPORTED`).
  - **Trace**: `R11.1`, `R9.4`.
- **C-Crypto-3 (disallowed algorithms)**: implementations SHALL NOT issue or accept new credentials using RSA variants, ECDSA secp256k1, ECDSA P-384/P-521 (not mandatory in v1), or any SHA-1-derived signature (carried from Ch.8.1.2).
  - **Trace**: `R11.1`.

## §6.2 Suite negotiation & post-quantum reservation (new — closes G-9)

- **C-Crypto-4 (PQC negotiation slot reserved)**: the credential signature structure SHALL provide an algorithm slot able to carry **ML-DSA-65** (FIPS 204) as an optional signing algorithm, and the negotiation SHALL reserve optional support for **TLS-1.3-PQ** (hybrid X25519 + ML-KEM-768), so a future default suite can be rolled forward without a wire-breaking change (parity with Faying G8 / R8).
  - **Trace**: `R11.2`.
- **C-Crypto-5 (rolling upgrade preserves verifiability)**: when a new default suite is introduced via negotiation, existing valid credentials SHALL remain verifiable until their `not_after`; a suite upgrade SHALL NOT invalidate in-flight valid credentials (couples to §1.6 C-Ver-4).
  - **Trace**: `R11.3`.

> **PQC schedule note**: CAP v1 declares PQC suites OPTIONAL only; the promotion of an Ed25519 + ML-DSA-65 hybrid from OPTIONAL to MUST is deferred to the §12.5 security-evolution schedule, tracking NIST PQC standardization (this resolves the Ch.10.5 "v2+ deferral" by reserving the slot now rather than later). This is a *reservation*, not an open PO decision — no `[依赖]` tag.

## §6.3 Key formats & signature input

### §6.3.1 Key formats (reuse of Ch.8.2)

- **Ed25519**: 32-byte compressed Edwards point (RFC 8032 §5.1.5); raw 64-byte signature (no ASN.1); JWS per RFC 8037.
- **ECDSA P-256**: public key uncompressed (65 B, `0x04‖X‖Y`) or compressed (33 B); implementations SHALL accept both; signature raw 64 B (`R‖S`), JWS per RFC 7518 §3.4.
- `VerificationKey.key_material` stores the raw bytes of the above.

### §6.3.2 Signature input (reuse of Ch.8.3, restated normative)

- **C-Crypto-6 (deterministic CBOR signature input)**: for offline credentials and `RevocationStatement`, the signature input SHALL be the Deterministic-CBOR (RFC 8949 §4.2.1) serialization of the payload (map keys lexicographically sorted, shortest numeric/length encoding, no indefinite-length). For online tickets, the JWS signing input SHALL be `base64url(header) "." base64url(payload)` (RFC 7515 §5.1).
  - **Trace**: `R9.2`.
- **C-Crypto-7 (cross-encoding signature equivalence)**: the signature input hash SHALL be identical under CBOR and JSON encodings, so independent implementations verify identically.
  - **Trace**: `R9.1`, `R9.2`.

## §6.4 Key distribution, storage, rotation (reuse of Ch.8.4–8.6)

- **Distribution**: terminals SHALL support offline pre-installation (`source = "pre-installed"`) and online distribution (`source = "ra-distributed"`, over TLS/mTLS, verified to originate from a trusted `Registration_Authority`). The RA trust anchor SHALL be pre-installed via a physical/controlled channel and SHALL NOT be updated through CAP runtime mechanisms.
- **Storage**: private keys SHALL be stored encrypted, readable only by `Protocol_Engine`, SHOULD reside in a hardware secure element (TPM / Secure Enclave / TEE); private keys SHALL NOT be written in plaintext to persistent media, transmitted in CAP messages, or exposed in logs.
- **Rotation**: `Verification_Key` rotation SHALL provide a smooth transition (default 30-day overlap); on `VerificationKeyRevocation` the terminal SHALL immediately mark the key revoked, reject credentials using it (`E_VERIFICATION_KEY_INVALID`), and forcibly terminate sessions whose credentials were issued under it (couples to §10 lifecycle). Terminal local signing keys SHOULD rotate every 90 days.

- **C-Crypto-8 (revoked key → reject + session kill)**: on a verified `VerificationKeyRevocation`, the CAP terminal SHALL reject all subsequent validations under that `key_id` and terminate referencing Sessions.
  - **Trace**: `R8.3`.

---

## §6.5 Summary of hard constraints (this chapter)

| ID | One-line | Trace |
|---|---|---|
| C-Crypto-1 | Mandatory `{Ed25519, ECDSA-P256}`; don't reject non-PQC peers | R11.1 |
| C-Crypto-2 | Negotiate strongest common; none → reject | R11.1, R9.4 |
| C-Crypto-3 | Disallowed algorithms (RSA / secp256k1 / SHA-1 / P-384/521) | R11.1 |
| C-Crypto-4 | PQC slot reserved (ML-DSA-65 / TLS-1.3-PQ) | R11.2 |
| C-Crypto-5 | Rolling upgrade preserves in-flight credential verifiability | R11.3 |
| C-Crypto-6 | Deterministic-CBOR / JWS signature input | R9.2 |
| C-Crypto-7 | Cross-encoding signature-input equivalence | R9.1, R9.2 |
| C-Crypto-8 | Revoked key → reject + session kill | R8.3 |

## Changelog
- (draft) Initial Cryptography chapter with suite negotiation + PQC reservation, H1-PROTO-01 Wave 2.
