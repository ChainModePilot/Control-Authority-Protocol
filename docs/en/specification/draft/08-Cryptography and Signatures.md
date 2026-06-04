# Chapter 8: Cryptography and Signatures

This chapter defines the signature algorithms, key formats, key lifecycle management, and distribution requirements used by the CAP protocol. The cryptographic requirements in this chapter are normative — any implementation conforming to the CAP protocol MUST implement per the definitions in this chapter.

## 8.1 Algorithm Set

CAP v1 defines two sets of mandatory signature algorithms and key formats:

| Algorithm ID | Algorithm | Public Key Length | Signature Length | Status |
|---------|------|---------|---------|------|
| `ed25519` | Ed25519 (RFC 8032) | 32 bytes | 64 bytes | Mandatory |
| `ecdsa-p256-sha256` | ECDSA P-256 + SHA-256 (RFC 6979) | 65 bytes (uncompressed) / 33 bytes (compressed) | 64 bytes (DER-encoded or raw) | Mandatory |

All CAP implementations MUST simultaneously support both algorithm sets. When Trusted_Ticket uses JWS, the corresponding algorithm names are:

| CAP Algorithm ID | JWS `alg` Field |
|------------|---------------|
| `ed25519` | `EdDSA` |
| `ecdsa-p256-sha256` | `ES256` |

### 8.1.1 Algorithm Selection Recommendations

- **Prefer `ed25519`**: Better performance, shorter signatures, simpler key management
- **`ecdsa-p256-sha256`**: For use under regulatory requirements such as FIPS 140-3

Issuers SHOULD use `ed25519` by default in all newly issued credentials, except when there are explicit compliance requirements.

### 8.1.2 Disallowed Algorithms

CAP v1 implementations MUST NOT use the following algorithms to issue or accept new credentials:

- Any RSA signature variants (poor performance, larger keys)
- ECDSA secp256k1 (non-FIPS standard curve)
- ECDSA P-384 / P-521 (not mandatory in v1, may be added in future versions)
- Any SHA-1 derived signature algorithms

## 8.2 Key Formats

### 8.2.1 Ed25519

Public keys are encoded per RFC 8032 §5.1.5:

- 32-byte little-endian compressed Edwards curve point

In DescriptorPayload, `signature.signature_value` is a 64-byte raw signature (without ASN.1 encoding).

In JWS, signatures are encoded per RFC 8037, with the 64-byte raw signature base64url-encoded.

### 8.2.2 ECDSA P-256

Public keys are encoded per RFC 5480:

- Uncompressed format: 65 bytes (0x04 || X || Y)
- Compressed format: 33 bytes (0x02/0x03 || X)

Implementations MUST simultaneously accept both uncompressed and compressed public keys.

Signature format:

- In DescriptorSignature: 64-byte raw signature (R || S, each 32 bytes big-endian), DER not used
- In JWS: encoded per RFC 7518 §3.4 (64 bytes R || S)

### 8.2.3 Binary Representation of Keys

The `VerificationKey.key_material` field directly stores the byte sequence of the above format:

- ed25519 → 32 bytes
- ecdsa-p256-sha256 (uncompressed) → 65 bytes
- ecdsa-p256-sha256 (compressed) → 33 bytes

## 8.3 Signature Input Construction

### 8.3.1 Authorization_Descriptor Signature Input

Construct the signature input following these steps:

1. Take `AuthorizationDescriptor.payload` (DescriptorPayload structure)
2. Serialize into a byte sequence using RFC 8949 CBOR **Deterministic Encoding**
3. The byte sequence becomes the input to the signature algorithm

Key constraints of CBOR Deterministic Encoding (from RFC 8949 §4.2):

- Map keys are sorted in lexicographic order
- Numeric values use the shortest encoding
- Strings and byte strings use the shortest length encoding
- Indefinite-length encoding is not used

Implementations MUST strictly adhere to these constraints to ensure cross-implementation signature verification consistency.

### 8.3.2 Trusted_Ticket Signature Input

Construct the JWS signature input per RFC 7515 §5.1:

```
Signing Input = base64url(UTF8(Header)) + "." + base64url(UTF8(Payload))
```

Where:

- Header and Payload are serialized into UTF-8 JSON per RFC 7159
- JSON field order: Implementations SHOULD sort keys in lexicographic order, but the strict field order requirement is determined by the base64url-encoded bytes

### 8.3.3 RevocationStatement Signature Input

The signature input construction for revocation statements is the same as for Authorization_Descriptor:

1. Remove the `signature` field, retaining the rest of the RevocationStatement fields
2. CBOR Deterministic Encoding serialization
3. Use the byte sequence as the signature input

## 8.4 Key Distribution

### 8.4.1 Distribution Modes

The terminal MUST support the following two Verification_Key distribution modes:

#### Offline Pre-installation

The initial Verification_Key is pre-installed at terminal factory or deployment. Pre-installed keys:

- Marked `source = "pre-installed"` in the `VerificationKey` structure
- MUST be written through a physical or controlled secure channel during manufacturing or deployment
- Typically correspond to root-level Descriptor_Issuers (e.g., device vendor authentication Issuers)

#### Online Distribution

Registration_Authority distributes new Verification_Keys through online interfaces:

- Marked `source = "ra-distributed"` in the `VerificationKey` structure
- MUST be transmitted over TLS/mTLS channels (see §1.3.4)
- The terminal MUST verify that the transmitter is a trusted Registration_Authority

### 8.4.2 Trust Anchor Configuration

The terminal's Registration_Authority public key (used to verify signatures of RA push messages) serves as a trust anchor:

- MUST be pre-installed during terminal manufacturing or initial deployment
- May only be replaced through physical or controlled channels
- Not updated through CAP protocol runtime mechanisms

### 8.4.3 Distribution Messages

Key distribution messages pushed by Registration_Authority:

```
VerificationKeyDistribution {
  required version       : uint32 (= 1)
  required key           : VerificationKey
  required ra_signature  : DescriptorSignature       // Signed by Registration_Authority
}
```

Terminal processing flow:

1. Verify that `ra_signature` was issued by a trusted Registration_Authority
2. Verify that `key.algorithm` is in the algorithm set permitted by §8.1
3. Verify that `key.valid_from` is reasonable relative to the terminal's current time (e.g., does not exceed current time + 24 hours)
4. Write to the terminal key store
5. Return a confirmation to Registration_Authority (the confirmation mechanism is outside the scope of this specification)

## 8.5 Key Storage

The terminal MUST securely store all Verification_Keys and local signing keys (see §4.4.3).

### 8.5.1 Storage Requirements

The terminal MUST:

1. Store the private key portion of all keys encrypted (public keys may be stored in plaintext)
2. Private key access is protected by OS process permissions; only Protocol_Engine can read
3. SHOULD store private keys in hardware secure elements (e.g., TPM, Secure Enclave, TEE)

The terminal MUST NOT:

1. Write private keys in plaintext to persistent media
2. Transmit private keys via CAP protocol messages
3. Expose private key contents in debug logs or error output

### 8.5.2 Key Leakage Handling

If the terminal detects that a key may have leaked (e.g., secure storage attacked):

1. Immediately stop using that key
2. Report to Registration_Authority through the §8.6 flow
3. Before the new key is fully distributed, reject all related credential validations (conservative policy)

## 8.6 Key Updates and Rotation

### 8.6.1 Smooth Transition

When rotating Verification_Keys, Registration_Authority MUST provide a smooth transition:

1. Distribute the new key to all terminals
2. During the transition period (default 30 days), the new and old keys are both valid
3. After the transition period ends, the old key's `valid_until` is reached, automatically invalidating it

During the transition period, the terminal MUST simultaneously maintain the new and old keys, selecting the corresponding key for validation per `key_id`.

### 8.6.2 Emergency Revocation

```
VerificationKeyRevocation {
  required version          : uint32 (= 1)
  required key_id           : string
  required revocation_time  : timestamp
  required reason           : enum["compromised", "superseded", "ra_decision"]
  required ra_signature     : DescriptorSignature
}
```

After receiving the revocation message, the terminal MUST:

1. Verify that the signature is from a trusted Registration_Authority
2. Immediately mark the key_id as revoked
3. Reject all credentials using this key_id in subsequent validation requests (return `E_VERIFICATION_KEY_INVALID`)
4. Proactively check all active Sessions: forcibly terminate Sessions using credentials issued with this key_id (see §5.5)

### 8.6.3 Rotation of Terminal Local Signing Keys

Local signing keys held by the terminal to support the §4.4 conversion flow:

- SHOULD be automatically rotated every 90 days
- During rotation, the terminal generates a new key pair; the old key continues to be used for validation of issued credentials until they all expire
- The terminal MAY simultaneously hold up to 4 local signing keys (covering the maximum validity period + smooth transition)

## 8.7 Cryptographic Requirements Summary

| Requirement | Scope |
|------|------|
| Signature algorithm MUST be chosen from ed25519 / ecdsa-p256-sha256 | All signature scenarios |
| Private keys MUST be securely stored | All issuers and terminals |
| Public key distribution MUST be via encrypted channels | Registration_Authority → terminal |
| Trust anchors MUST be pre-installed via physical or controlled channels | All terminals |
| CBOR Deterministic Encoding for offline credential signature input | Authorization_Descriptor, RevocationStatement |
| JWS Compact Serialization for online tickets | Trusted_Ticket |
