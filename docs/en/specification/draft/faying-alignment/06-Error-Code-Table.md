# CAP Final §11 — Error Handling & Error-Code Table (Normative draft)

> **Status**: Drafted clause for H1-PROTO-01 (closes G-7). This uplifts the current Ch.9 flat error table into a Faying §10-style table with **consequence columns** (triggers session-termination / triggers tighten-or-downgrade / enters audit / reaches Human-Prime view) and adds the codes required by Faying alignment (`04`) and destructive-operation confirmation (`05`). It also states a unified **Default-Deny** convergence rule.
> **Convention**: CAP retains its string error codes `E_UPPER_SNAKE` (token separators are `_`, per the org code-constraints). Codes from the current draft Ch.9 are **retained verbatim**; new codes are marked **(new)**. `Trace:` cites `02`.
> **Columns**: **Term?** = does it trigger termination of the affected Session(s); **Tighten?** = triggers grade tightening / downgrade / hold; **Audit?** = mandatory audit-log entry; **HP-view?** = mandatory push to the Human-Prime view channel.

---

## 11.1 Error envelope *(reuse of Ch.9.1)*

```
ErrorBody {
  required error_code     : string
  required message        : string            // human-readable; no sensitive internals (§11.5)
  optional details        : map<string,string>
  optional retry_after_ms : uint32            // transient errors only
}
```

## 11.2 New — Faying alignment & destructive-operation codes

These are the codes introduced by `04` (control model) and `05` (destructive-op confirmation).

| Error code | Trigger scenario | Protocol response | Term? | Tighten? | Audit? | HP-view? | Trace |
|---|---|---|:--:|:--:|:--:|:--:|---|
| `E_MANDATE_MISSING` (new) | `AuthRequest` carries no verifiable binding to a live Faying Mandate | Reject under Default-Deny; no Session established | ✗ | ✗ | ✓ | ✗ | R1.2 |
| `E_MANDATE_INVALID` (new) | Mandate binding fails verification / attestation chain does not resolve to one accountability endpoint | Reject; write event | ✗ | ✗ | ✓ | ✓ | R1.3 |
| `E_MANDATE_REVOKED` (new) | Bound Mandate is `Revoked` (any RevokeReason) | Reject + terminate referencing Sessions; cascade same-chain | ✓ | ✗ | ✓ | ✓ | R4.5 |
| `E_GRADE_INSUFFICIENT` (new) | Effective grade is stricter-than-satisfied vs `Grant.min_grade` / operation's required grade | Reject + hint upgrade (upgrade path is Faying-side, needs HP witness) | ✗ | ✗ | ✓ | ✗ | R4.1 |
| `E_PRESENCE_STALE` (new) | Presence proof exceeds the grade's `max_staleness_ms` | Reject the operation; tighten per C-Grade-3; await fresh presence | ✗ | ✓ | ✓ | ✗ | R4.2 |
| `E_PRESENCE_REQUIRED` (new) | Operation needs a presence proof and none is bound | Reject under Default-Deny | ✗ | ✗ | ✓ | ✗ | R5.2 |
| `E_HUMAN_CONFIRMATION_REQUIRED` (new) | `high`/`critical` operation lacks a valid per-operation confirmation | Reject; await confirmation then re-request | ✗ | ✗ | ✓ | ✓ | R5.2, R5.3 |
| `E_HUMAN_CONFIRMATION_TIMEOUT` (new) | Confirmation not received within timeout | Default-deny; do not execute | ✗ | ✗ | ✓ | ✓ | R5.9 |
| `E_OPAQUE_ACTION_BUNDLE` (new) | A single confirmation/`execute` covers a bundle containing a `high`/`critical` op | Reject (anti-CT2) | ✗ | ✗ | ✓ | ✓ | R5.5 |
| `E_CAPABILITY_UNDECLARED` (new) | Operation has no declared criticality; treated as `critical` | Reject (default-deny on unknown) | ✗ | ✗ | ✓ | ✗ | R2.4 |
| `E_EMERGENCY_OVERRIDE_ACTIVE` (new) | A Faying G0 is in effect for the Fay | Reject all of the Fay's requests; sessions already terminated | ✓ | ✗ | ✓ | ✓ | R4.3 |

## 11.3 Credential-related codes *(retained from Ch.9.2, consequence columns added)*

| Error code | Trigger | Term? | Tighten? | Audit? | HP-view? |
|---|---|:--:|:--:|:--:|:--:|
| `E_DESCRIPTOR_NOT_FOUND` | Referenced `descriptor_id` not stored locally | ✗ | ✗ | ✓ | ✗ |
| `E_INVALID_STRUCTURE` | Credential structure ≠ §4.2 | ✗ | ✗ | ✓ | ✗ |
| `E_INVALID_SIGNATURE` | Signature verification failed | ✗ | ✗ | ✓ | ✓ |
| `E_VERIFICATION_KEY_INVALID` | Signing key unregistered/revoked/expired | ✗ | ✗ | ✓ | ✓ |
| `E_UNKNOWN_ISSUER` | `issuer_id` not in terminal trust list | ✗ | ✗ | ✓ | ✓ |
| `E_DESCRIPTOR_REVOKED` | Descriptor revoked | ✓ | ✗ | ✓ | ✓ |
| `E_TICKET_REVOKED` | Ticket revoked | ✓ | ✗ | ✓ | ✓ |
| `E_DESCRIPTOR_EXPIRED` | Past `not_after` | ✓ | ✗ | ✓ | ✗ |
| `E_DESCRIPTOR_NOT_YET_VALID` | Before `not_before` | ✗ | ✗ | ✓ | ✗ |
| `E_TICKET_MALFORMED` | JWS parse failure / wrong `typ` | ✗ | ✗ | ✓ | ✗ |
| `E_DUPLICATE_DESCRIPTOR_ID` | Same id, mismatched content | ✗ | ✗ | ✓ | ✗ |
| `E_VALIDITY_OUT_OF_RANGE` | `not_after − not_before` over limit | ✗ | ✗ | ✓ | ✗ |
| `E_REVOCATION_QUERY_TIMEOUT` | Online revocation query timeout | ✗ | ✓ | ✓ | ✗ |

## 11.4 Authorization-scope, resource/session, handover, protocol/system codes *(retained from Ch.9.3–9.6)*

Retained verbatim with consequence columns; the notable couplings:

| Error code | Group | Term? | Tighten? | Audit? | HP-view? |
|---|---|:--:|:--:|:--:|:--:|
| `E_SUBJECT_MISMATCH` / `E_TERMINAL_MISMATCH` / `E_AUTHORIZATION_INSUFFICIENT` | scope | ✗ | ✗ | ✓ | ✗ |
| `E_RATE_LIMIT_EXCEEDED` / `E_OUT_OF_TIME_WINDOW` / `E_OUT_OF_GEO_FENCE` / `E_UNSUPPORTED_CONSTRAINT` | scope | ✗ | ✗ | ✓ | ✗ |
| `E_RESOURCE_BUSY` / `E_RESOURCE_UNAVAILABLE` / `E_RESOURCE_NOT_FOUND` | resource | ✗ | ✗ | ✓ | ✗ |
| `E_SESSION_NOT_FOUND` / `E_SESSION_LIMIT_EXCEEDED` | session | ✗ | ✗ | ✓ | ✗ |
| `E_SESSION_TERMINATED` | session | ✓ | ✗ | ✓ | ✗ |
| `E_OS_INTEGRATION_FAILED` | resource | ✓ | ✗ | ✓ | ✗ |
| `E_HANDOVER_*` (8 codes) | handover | varies | ✗ | ✓ | ✗ |
| `E_PROTOCOL_VERSION_UNSUPPORTED` / `E_INVALID_MESSAGE` / `E_INTERNAL_ERROR` / `E_NOT_IMPLEMENTED` / `E_STORAGE_FULL` | system | ✗ | ✗ | ✓ | ✗ |
| `E_PROTOCOL` (new, alias of `E_INVALID_MESSAGE` for Faying parity) | system | ✗ | ✗ | ✓ | ✗* |

\* `E_PROTOCOL` additionally reaches the Human-Prime view when raised on a `high`/`critical` disclosure path (`05` C-Dstr-7/-10), matching Faying's DISCLOSURE-path note.

## 11.5 Observability constraints *(uplift of Ch.9.9 → normative)*

- **C-Err-1 (no internal disclosure)**: Error responses SHALL NOT expose key fingerprints, internal IDs, resource topology, or signature material; `details` SHALL carry only diagnostic field names (e.g., `details["field"]="not_after"`). **Trace**: `R10.3`.
- **C-Err-2 (most-specific code)**: The CAP terminal SHALL return the most specific applicable code (e.g., `E_DESCRIPTOR_REVOKED` over `E_INVALID_SIGNATURE`). 
- **C-Err-3 (mandatory HP-view for critical classes)**: The CAP terminal SHALL push to the Human-Prime view channel every code marked HP-view ✓, in a priority bucket no lower than `high`, and `critical`-bucket events SHALL NOT be dropped under rate-limiting (anti-CT7). **Trace**: `R12.7`, R5.7.
- **C-Err-4 (timing uniformity)**: The CAP terminal SHOULD keep response times comparable across validation-failure branches to limit side-channel inference.

## 11.6 Unified Default-Deny convergence *(new — Faying EH.4 parity)*

- **C-Err-5 (Default-Deny)**: For every error in this chapter, the CAP terminal's default outcome SHALL be denial of the requested control; no error path SHALL produce a partial grant, an optimistic pass, or a silent escalation. WHEN a dependency (mandate validity, revocation status, presence freshness, operation classification) is unavailable, the CAP terminal SHALL deny rather than assume. **Trace**: `R2.1`, `R2.2`, `R2.4`.

---

## 11.7 Open items

> **`[需人类决策 D-8]` — numeric companion codes.** Faying uses numeric codes (1001–1099) for language-neutral cross-impl diagnosis; CAP uses descriptive strings. Decide whether to add a stable numeric companion column (helps conformance vectors) or keep strings only. This draft keeps strings (minimizes churn vs current Ch.9) and recommends a numeric companion be added in the conformance-vector index rather than the wire format.
>
> **`[需人类决策 D-9]` — `E_PROTOCOL` vs `E_INVALID_MESSAGE`.** Whether to adopt Faying's `E_PROTOCOL` as the canonical catch-all (aliasing the current `E_INVALID_MESSAGE`) or keep CAP's existing name. This draft introduces `E_PROTOCOL` as an alias for parity; pick one before freeze.
>
> **Schema sync (carried from Ch.9.10)**: WHEN this table changes, `schemas/cap-1.0.cddl` and the conformance-vector error enumeration SHALL be updated in lockstep.

## Changelog
- (draft) Error table uplift with consequence columns + Faying-alignment/destructive-op codes, H1-PROTO-01.
