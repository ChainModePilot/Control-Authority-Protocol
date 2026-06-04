# Chapter 0: Introduction and Conformance

## 0.1 Document Status

This document is the draft version of the normative technical specification of the Control Authority Protocol (CAP). The draft version makes no commitment to backward compatibility and may undergo arbitrary changes prior to formal release. Once this draft stabilizes and passes complete test verification, it will be released as the first formal version under `specification/2025-10-25/`.

This document is intended for use alongside the CAP architecture blueprint (`docs/en/blueprint/`):

- The **blueprint** answers "what, why, and what for" — defining the protocol's design intent, capability boundaries, and core mechanisms
- The **specification** answers "how, and how to verify" — defining the protocol's message formats, flow steps, error handling, and conformance requirements

When non-normative descriptions in the blueprint conflict with normative provisions in this specification, this specification takes precedence.

## 0.2 Scope

This specification defines the technical details of CAP protocol v1, covering the 6 core capabilities listed in Section 3.1 of Chapter 3 of the blueprint:

1. Issuance, storage, validation, revocation, and renewal of offline authorization (Authorization_Descriptor)
2. Issuance, query, and conversion to offline authorization of online tickets (Trusted_Ticket)
3. Complete lifecycle of session management (Session)
4. Three policy types and atomicity guarantees of control authority handover policy (Handover_Policy)
5. Dual-determination mechanism of liveness detection (Liveness_Detection)
6. Read-write lock model of resource access mode (Resource_Access_Mode)

This specification explicitly excludes the features listed in Section 3.2 of Chapter 3 of the blueprint, including cross-terminal session migration, multi-terminal collaborative authorization, authorization delegation chains, ABAC, audit log standardized format, protocol message encrypted transmission specification, distributed revocation consensus, and dynamic permission elevation within a Session.

## 0.3 Conformance Terminology

This specification follows the keyword conventions of RFC 2119 and RFC 8174. The following keywords, when appearing in **all uppercase**, have normative meaning:

- **MUST**: Absolute requirement. Implementations that do not satisfy this requirement do not conform to this specification
- **MUST NOT**: Absolute prohibition. Implementations that violate this prohibition do not conform to this specification
- **SHOULD**: Strong recommendation. May be deviated from for valid reasons given full understanding of the consequences
- **SHOULD NOT**: Strong discouragement. May be deviated from for valid reasons given full understanding of the consequences
- **MAY**: Optional. Implementations may decide for themselves whether to provide

The same vocabulary not appearing in all uppercase carries only its literal meaning and has no normative force.

## 0.4 Terminology Alignment

The terminology used in this specification is fully consistent with the blueprint `00-Glossary.md`. When this specification references a term, it uses the identifier defined in the blueprint (e.g., `Authorization_Descriptor`, `Fay`, `Terminal_Resource`).

For ease of reference, this specification marks key terms in **bold** at their first occurrence in each chapter, with brief definitions from the blueprint glossary. The complete definitions of terms are governed by the blueprint.

## 0.5 Conformance Levels

This specification defines 3 implementation conformance levels. An implementation MUST satisfy at least the "Terminal Conformance Level" to claim compliance with CAP v1.

### 0.5.1 Terminal Conformance

Applies to terminal devices implementing `Descriptor_Validator`, `Protocol_Engine`, and session management logic. Terminal implementations MUST:

1. Fully implement the Authorization_Descriptor validation flow defined in Chapter 3
2. Fully implement the Session lifecycle and Liveness_Detection mechanism defined in Chapter 5
3. Fully implement the Resource_Access_Mode read-write lock semantics defined in Chapter 7
4. Reject all requests that do not pass validation and return standardized error codes per Chapter 9
5. Maintain a local revocation list and synchronize it per the policy defined in Chapter 3 when online

### 0.5.2 Issuer Conformance

Applies to trusted entities implementing `Descriptor_Issuer` or `Ticket_Issuer`. Issuer implementations MUST:

1. Generate legitimate Authorization_Descriptor and Trusted_Ticket per the field constraints defined in Chapter 2
2. Apply digital signatures to credentials per the cryptographic requirements defined in Chapter 8
3. Maintain status records of issued credentials and support revocation operations
4. Implement the Trusted_Ticket-to-Authorization_Descriptor conversion defined in Chapter 4

### 0.5.3 Runtime Conformance

Applies to Fay runtime environments implementing `iFay_Runtime`. Runtime implementations MUST:

1. Submit authorization validation requests to Protocol_Engine per the interface contract defined in Chapter 1
2. Maintain persistent connections and send application-layer heartbeats at the frequency defined in Chapter 5
3. Receive and forward session state change notifications pushed by Protocol_Engine
4. Proactively notify Protocol_Engine to release related Sessions when a Fay instance terminates

An implementation may satisfy multiple conformance levels simultaneously. For example, an integrated terminal may serve as both a terminal implementation and a runtime implementation.

## 0.6 Normative References

This specification normatively references the following documents:

- RFC 2119 / RFC 8174: Keyword conventions used in this specification
- RFC 8949 (CBOR): Compact binary serialization for Authorization_Descriptor (see Chapter 2)
- RFC 8032 (EdDSA): Default digital signature algorithm (see Chapter 8)
- RFC 7515 (JWS): JSON serialization and signing for Trusted_Ticket (see Chapter 4)
- RFC 5280 (X.509): Optional certificate format (see Chapter 8)

The CAP Schema definition file (`schema/{version}/schema.json`) serves as the formal supplement to Chapter 2 of this specification, with normative force equivalent to this specification. When schema.json conflicts with the textual description in this specification, schema.json takes precedence.

## 0.7 Document Structure

This specification is organized in the following order:

| Chapter | Topic | Key Content |
|------|------|---------|
| Chapter 1 | Architecture and Roles | Protocol roles, trust chain, external interface contracts |
| Chapter 2 | Data Model | Field-level definitions of core data structures |
| Chapter 3 | Offline Authorization Protocol | Complete Authorization_Descriptor flow |
| Chapter 4 | Online Ticket Protocol | Complete Trusted_Ticket flow and degradation |
| Chapter 5 | Session Management and Liveness Detection | Session state machine and heartbeat |
| Chapter 6 | Control Authority Handover Protocol | Three Handover_Policy policy types |
| Chapter 7 | Resource Access Mode | Semantics of read/write/execute/configure |
| Chapter 8 | Cryptography and Signatures | Algorithm set, key formats, distribution |
| Chapter 9 | Error Codes and Conformance Levels | Standard error codes, level declaration |
| Chapter 10 | Security Considerations | Threat model, known risks |

Readers are advised to read Chapters 0–2 in sequence to establish a foundation, then jump to relevant chapters based on implementation focus.
