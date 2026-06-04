# Chapter 2: Data Model

This chapter defines the core data structures in the CAP protocol, including field names, types, constraints, and default values. The field definitions in this chapter are normative — any implementation conforming to the CAP protocol MUST generate and parse these data structures according to the definitions in this chapter.

`schema/{version}/schema.json` provides a formal supplement to the data structures in this chapter. When schema.json conflicts with the description in this chapter, schema.json takes precedence.

## 2.1 Data Type Conventions

This specification uses the following base types to describe fields:

| Type | Description | Encoding |
|------|------|------|
| `string` | UTF-8 string | UTF-8 byte sequence |
| `bytes` | Byte sequence | Raw bytes |
| `uint32` / `uint64` | Unsigned integer | Big-endian |
| `timestamp` | Unix timestamp (seconds) | uint64 |
| `uuid` | RFC 4122 UUID v7 | 16 bytes |
| `enum` | Enumerated value | String literal |
| `array<T>` | Ordered collection of elements of type T | Array |
| `map<K,V>` | Mapping from K to V | Object |

Field constraints use the following notation:

- `required`: The field MUST be present, with a non-null value
- `optional`: The field MAY be present; absence is treated as unset
- `unique`: The field value is unique within the system scope
- `len(N..M)`: String/byte length is between N and M (inclusive)
- `regex(...)`: The field value MUST match the specified regular expression

## 2.2 Identifiers

Core identifiers in the CAP protocol MUST satisfy global uniqueness. This section defines the format and generation rules of various identifiers.

### 2.2.1 Fay_ID

`Fay_ID` uniquely identifies a Fay instance.

| Attribute | Value |
|------|------|
| Type | `string` |
| Format | `"fay:" + uuid_v7` |
| Length | 40 characters (including prefix) |
| Uniqueness | Globally unique |
| Generator | Identity management subsystem (outside CAP protocol scope) |

Example: `fay:01927b34-7e21-7c4d-a89f-1234567890ab`

### 2.2.2 Terminal_ID

`Terminal_ID` uniquely identifies a terminal device.

| Attribute | Value |
|------|------|
| Type | `string` |
| Format | `"terminal:" + uuid_v7` |
| Length | 45 characters (including prefix) |
| Uniqueness | Globally unique |
| Generator | Registration_Authority |

### 2.2.3 Resource_ID

`Resource_ID` identifies a specific resource on the terminal.

| Attribute | Value |
|------|------|
| Type | `string` |
| Format | `terminal_id + "/" + resource_path` |
| Length | At most 256 characters |
| Uniqueness | Unique within terminal scope |
| Generator | Terminal operating system |

`resource_path` MUST satisfy the regular expression `^[a-zA-Z0-9._\-/]+$`.

Example: `terminal:01927b34-.../device/camera/front`

### 2.2.4 Descriptor_ID

`Descriptor_ID` uniquely identifies an Authorization_Descriptor.

| Attribute | Value |
|------|------|
| Type | `uuid` |
| Format | UUID v7 |
| Uniqueness | Globally unique (across all Descriptor_Issuers) |
| Generator | Descriptor_Issuer |

The revocation list uses Descriptor_ID to identify revoked credentials. Descriptor_Issuer MUST NOT reuse a previously used Descriptor_ID.

### 2.2.5 Session_ID

`Session_ID` uniquely identifies an active session.

| Attribute | Value |
|------|------|
| Type | `uuid` |
| Format | UUID v7 |
| Uniqueness | Unique within terminal scope |
| Generator | Terminal Protocol_Engine |
| Lifecycle | Valid only while the session is active; the ID is not reused after the session terminates |

## 2.3 Authorization_Descriptor

Authorization_Descriptor is the core data structure for offline authorization. An Authorization_Descriptor consists of two parts: **payload** and **signature**.

### 2.3.1 Top-Level Structure

```
AuthorizationDescriptor {
  required version       : uint32
  required payload       : DescriptorPayload
  required signature     : DescriptorSignature
}
```

| Field | Description |
|------|------|
| `version` | Protocol version number; v1 implementations MUST set to `1` |
| `payload` | Authorization information payload (see §2.3.2) |
| `signature` | Digital signature over the payload (see §2.3.3) |

### 2.3.2 DescriptorPayload

```
DescriptorPayload {
  required descriptor_id   : Descriptor_ID
  required issuer_id       : string
  required subject_fay_id  : Fay_ID
  required terminal_id     : Terminal_ID
  required grants          : array<Grant> (len 1..256)
  required issued_at       : timestamp
  required not_before      : timestamp
  required not_after       : timestamp
  optional grantor_id      : string
  optional metadata        : map<string, string>
}
```

| Field | Constraints | Description |
|------|------|------|
| `descriptor_id` | required, unique | Globally unique identifier of this credential |
| `issuer_id` | required | Issuer identifier, corresponding to Descriptor_Issuer in the key trust path |
| `subject_fay_id` | required | Fay identifier being authorized |
| `terminal_id` | required | Terminal identifier limiting the authorization scope |
| `grants` | required, 1..256 elements | List of specific authorization items (see §2.3.4) |
| `issued_at` | required | Issuance time |
| `not_before` | required, ≥ issued_at | Validity start time |
| `not_after` | required, > not_before | Expiration time |
| `grantor_id` | optional | Authorizer identifier (Natural_Person or Official_Post) |
| `metadata` | optional | Issuer-defined custom metadata, with no impact on protocol semantics |

Implementations MUST reject Authorization_Descriptors that fail to satisfy the following conditions:

1. `not_after - not_before > 90 days`: This specification limits the maximum validity period of a single credential to 90 days
2. `not_before > current time + 24 hours`: Issuing credentials with an excessively early effective time is forbidden (preventing pre-issuance abuse)
3. `grants` array is empty: A credential without authorization items is meaningless

### 2.3.3 DescriptorSignature

```
DescriptorSignature {
  required algorithm       : enum["ed25519", "ecdsa-p256-sha256"]
  required key_id          : string
  required signature_value : bytes
}
```

| Field | Description |
|------|------|
| `algorithm` | Signature algorithm; see Chapter 8 |
| `key_id` | Identifier of the key used for signing, corresponding to the Verification_Key identifier |
| `signature_value` | Result of signing the CBOR-serialized payload bytes |

Signature input: Serialize `DescriptorPayload` into a byte sequence using RFC 8949 CBOR Deterministic Encoding, and use it as the signature algorithm input.

### 2.3.4 Grant

```
Grant {
  required resource_pattern : string
  required modes            : array<AccessMode> (len 1..4)
  optional constraints      : map<string, string>
}
```

| Field | Description |
|------|------|
| `resource_pattern` | Resource matching pattern (see §2.3.5) |
| `modes` | List of authorized access modes; element type `AccessMode` |
| `constraints` | Additional constraints (e.g., time windows, geofences); see Chapter 7 for impact on protocol semantics |

### 2.3.5 Resource Matching Pattern

`resource_pattern` supports the following matching syntax:

- Exact match: `terminal:xxx/device/camera/front`
- Wildcard match: `terminal:xxx/device/camera/*` (matches all cameras under that terminal)
- Full-terminal match: `terminal:xxx/device/camera/**` (matches all levels under that path)

Implementations MUST:

1. Support only the three syntaxes above; reject patterns containing other special characters
2. Match wildcard `*` to a single-level path segment only
3. Allow wildcard `**` only at the end of the pattern

### 2.3.6 AccessMode

```
AccessMode = enum["read", "write", "execute", "configure"]
```

Semantics of each access mode are described in Chapter 7.

## 2.4 Trusted_Ticket

Trusted_Ticket is an online authorization credential. Its structure is based on RFC 7515 JWS Compact Serialization.

### 2.4.1 Top-Level Structure

A Trusted_Ticket is a JWS string consisting of three parts separated by `.`:

```
base64url(header) . base64url(payload) . base64url(signature)
```

### 2.4.2 Header

```
TicketHeader {
  required alg : enum["EdDSA", "ES256"]
  required typ : "cap-ticket+jws"
  required kid : string
}
```

| Field | Description |
|------|------|
| `alg` | Signature algorithm (consistent with §8) |
| `typ` | Fixed value `"cap-ticket+jws"`, used to distinguish ticket types |
| `kid` | Key identifier, used for signature verification |

### 2.4.3 Payload

```
TicketPayload {
  required jti  : uuid                    // Unique ticket ID
  required iss  : string                  // Ticket_Issuer identifier
  required sub  : Fay_ID                  // Authorized Fay
  required aud  : Terminal_ID             // Target terminal
  required iat  : timestamp               // Issuance time
  required nbf  : timestamp               // Validity start time
  required exp  : timestamp               // Expiration time
  required grants : array<Grant>          // Same structure as §2.3.4
  optional convertible : boolean (default true)  // Whether convertible to Authorization_Descriptor
}
```

Implementations MUST reject Trusted_Tickets where `exp - nbf > 7 days`. The maximum validity period of online tickets is shorter than offline authorization, ensuring that the online revocation mechanism can take effect promptly.

### 2.4.4 Conversion from Trusted_Ticket to Authorization_Descriptor

When `convertible == true`, the terminal MAY convert a Trusted_Ticket to a local Authorization_Descriptor format for offline use. Conversion rules:

| TicketPayload Field | Mapped to DescriptorPayload Field |
|-------------------|-----------------------------|
| `jti` | `descriptor_id` |
| `iss` | `issuer_id` |
| `sub` | `subject_fay_id` |
| `aud` | `terminal_id` |
| `iat` | `issued_at` |
| `nbf` | `not_before` |
| `exp` | `not_after` (but MUST NOT exceed `iat + 7 days`) |
| `grants` | `grants` |

The converted Authorization_Descriptor is re-signed by the terminal using its locally stored key, with the signature `key_id` marked as a conversion source (see Chapter 4). The original Trusted_Ticket's signature information MUST be retained in metadata for audit purposes.

## 2.5 Session

Session is the internal session state structure of the terminal; the complete structure is not transmitted in protocol messages. This section defines the fields of Session to standardize the state machine and interface conventions.

```
Session {
  required session_id          : Session_ID
  required fay_id              : Fay_ID
  required runtime_id          : string
  required resource_id         : Resource_ID
  required access_mode         : AccessMode
  required granted_modes       : array<AccessMode>
  required state               : SessionState
  required created_at          : timestamp
  required last_heartbeat_at   : timestamp
  required credential_ref      : CredentialRef
}

SessionState = enum[
  "creating",
  "active",
  "handover_pending",
  "terminating",
  "terminated"
]

CredentialRef {
  required type   : enum["descriptor", "ticket"]
  required id     : string                     // descriptor_id or jti
  required not_after : timestamp
}
```

The `SessionState` state machine is described in Chapter 5.

## 2.6 Protocol Message Encapsulation

All messages between iFay_Runtime and Protocol_Engine share the following encapsulation structure:

```
ProtocolMessage {
  required version         : uint32 (= 1)
  required message_id      : uuid
  required message_type    : string
  required timestamp       : timestamp
  required sender_id       : string
  required body            : object
  optional correlation_id  : uuid
}
```

| Field | Description |
|------|------|
| `version` | Protocol version number; v1 sets to `1` |
| `message_id` | Unique identifier of this message |
| `message_type` | Message type literal (e.g., `"AuthRequest"`) |
| `timestamp` | Message send time |
| `sender_id` | Sender identifier (runtime_id or terminal_id) |
| `body` | Message body, with structure determined by message_type |
| `correlation_id` | Associated request message ID (response messages MUST set this field) |

The `body` structure corresponding to each `message_type` is defined in the corresponding chapter.

## 2.7 Verification_Key

Verification_Key is the signature verification key held by the terminal.

```
VerificationKey {
  required key_id        : string
  required algorithm     : enum["ed25519", "ecdsa-p256-sha256"]
  required key_material  : bytes              // Public key bytes
  required issuer_id     : string             // Issuer identifier corresponding to this key
  required valid_from    : timestamp
  optional valid_until   : timestamp
  required source        : enum["pre-installed", "ra-distributed"]
}
```

| Field | Description |
|------|------|
| `key_id` | Key identifier, corresponding to DescriptorSignature.key_id |
| `algorithm` | Signature algorithm supported by this key |
| `key_material` | Raw bytes of the public key; encoding determined by algorithm (see Chapter 8) |
| `issuer_id` | Descriptor_Issuer identifier corresponding to this key |
| `valid_from` | Key validity start time |
| `valid_until` | Key expiration time; unset means long-term valid |
| `source` | Key source: `pre-installed` (terminal factory pre-installation) or `ra-distributed` (Registration_Authority online distribution) |

The terminal MUST securely store all Verification_Keys (see Chapter 8).

## 2.8 Revocation Statement

A revocation statement is used to notify the terminal that a particular credential has been revoked.

```
RevocationStatement {
  required version              : uint32 (= 1)
  required revocation_id        : uuid
  required target_descriptor_id : Descriptor_ID
  required issuer_id            : string
  required revoked_at           : timestamp
  optional reason               : enum["unspecified", "compromised", "superseded", "no_longer_needed"]
  required signature            : DescriptorSignature
}
```

The revocation statement MUST be issued and signed by the `issuer_id` of the original Authorization_Descriptor.

## 2.9 Serialization and Transmission

The CAP protocol uses the following serialization formats:

| Data Structure | Serialization Format | Purpose |
|---------|-----------|------|
| AuthorizationDescriptor | RFC 8949 CBOR (Deterministic Encoding) | Offline storage and transmission |
| Trusted_Ticket | RFC 7515 JWS Compact Serialization | Online transmission |
| ProtocolMessage | JSON (UTF-8) | iFay_Runtime ↔ Protocol_Engine interaction |
| RevocationStatement | RFC 8949 CBOR (Deterministic Encoding) | Revocation list distribution |

Implementations MAY use CBOR instead of JSON for the transport layer of ProtocolMessage to reduce overhead, but the field names and semantics defined in schema.json MUST remain consistent.
