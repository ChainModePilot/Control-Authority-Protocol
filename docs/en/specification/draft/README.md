# CAP Protocol Technical Specification (Draft)

This directory contains the draft version of the Control Authority Protocol (CAP) v1 technical specification. The specification is developed based on the architecture blueprint in `docs/en/blueprint/`, covering the 6 core capabilities listed in §3.1 of Chapter 3 of the blueprint.

## Document Structure

| Chapter | File | Content |
|------|------|------|
| Chapter 0 | `00-Introduction and Conformance.md` | Document status, scope, RFC 2119 keywords, conformance levels, normative references |
| Chapter 1 | `01-Architecture and Roles.md` | Protocol roles, trust chain, external interface contracts |
| Chapter 2 | `02-Data Model.md` | Core data structures (Authorization_Descriptor, Trusted_Ticket, Session, Verification_Key) |
| Chapter 3 | `03-Offline Authorization Protocol.md` | Complete Authorization_Descriptor lifecycle protocol flow |
| Chapter 4 | `04-Online Ticket Protocol.md` | Complete Trusted_Ticket flow and degradation |
| Chapter 5 | `05-Session Management and Liveness Detection.md` | Session state machine, binding rules, heartbeats, dual determination, timeout reclamation |
| Chapter 6 | `06-Control Authority Handover Protocol.md` | Three Handover_Policy policy types, atomicity guarantees, timeout rollback |
| Chapter 7 | `07-Resource Access Mode.md` | Semantics of read/write/execute/configure, read-write lock matrix |
| Chapter 8 | `08-Cryptography and Signatures.md` | Algorithm set, key formats, distribution, and rotation |
| Chapter 9 | `09-Error Codes and Conformance Levels.md` | Standard error code table, conformance declaration |
| Chapter 10 | `10-Security Considerations.md` | Threat model, known risks, and mitigations |

## Recommended Reading Order

1. First read: Chapter 0 → Chapter 1 → Chapter 2 → Chapter 3
2. Implementing terminal: Chapters 0–3 → Chapters 5, 7 → Chapters 8, 9
3. Implementing issuer: Chapters 0–2 → Chapters 3, 4 → Chapter 8
4. Implementing iFay_Runtime: Chapters 0, 1 → Chapter 5 → Chapter 9
5. Security review: Chapter 10 + cross-reading of related chapters

## Draft Status

This draft is in the discussion phase. Prior to formal release:

- Field names, error codes, and constraint thresholds may be adjusted
- Chapter structure may be reorganized
- No backward compatibility is guaranteed

After discussions stabilize, the contents of this directory will be released as `docs/en/specification/2025-10-25/`, the first formal version of the CAP protocol.

## Related Assets

- Architecture blueprint: `docs/en/blueprint/`
- Schema definitions (draft): `schema/draft/`
- Other language versions: This specification currently has zh-CN, zh-TW, ja, and ko versions; remaining languages will be translated prior to formal release
