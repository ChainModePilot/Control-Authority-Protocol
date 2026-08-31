# CAP Final §5 — Protocol Messages (Normative draft)

> **Status**: Drafted clause for H1-PROTO-01, Wave 2 (uplift of current Ch.1.3 message tables + Ch.2.6 envelope). Defines encoding rules (CBOR primary + JSON fallback), the message envelope, the full message inventory including the **new** destructive-operation-confirmation and emergency-override messages, and the pointer to §11 error codes. Modeled on `Faying-Protocol-1.0` §5.
> **Style**: hard constraints `C-Msg-<n>`, `Trace: R<n>.<m>.<k>` to `02`. Open PO decisions are marked `[依赖 D-n]` (see `08`).
> **Type authoritativeness**: the structures referenced here (`Authorization_Descriptor`, `Grant`, `OperationDescriptor`, `Session`, …) have their **authoritative definitions in §4 Data Models (CDDL)**; this chapter's inline shapes are a human-reading view, and where they differ §4 CDDL prevails.

---

## §5.1 Encoding rules

- **Primary format (CBOR)**: the default on-the-wire encoding is CBOR (RFC 8949); HTTP `Content-Type` `application/cap+cbor`.
- **Interoperability fallback (JSON)**: JSON (RFC 8259) for debugging / browser / human-review; `Content-Type` `application/cap+json`. JSON and CBOR are **semantically equivalent** for every message and credential.
- **Deterministic encoding for signatures**: every signed object SHALL be serialized with RFC 8949 §4.2.1 Deterministic Encoding before signing; under JSON, the signature input remains the hash of the Deterministic-CBOR byte string (see §6.3). Offline credentials sign over Deterministic CBOR; online tickets use RFC 7515 JWS.
- **C-Msg-1 (semantic equivalence)**: The CAP terminal SHALL accept the CBOR primary encoding and the JSON fallback as semantically equivalent for every message and credential; an object's signature input SHALL be identical under both encodings.
  - **Trace**: `R9.1`, `R9.2`.

> **Uplift note**: current Ch.2.9 makes the `ProtocolMessage` envelope JSON-primary with optional CBOR. The Final direction (R9) is **CBOR-primary + JSON-fallback** for parity with Faying and for a single deterministic signature input. The exact envelope encoding default is finalized at the §4 data-model freeze (it is wire-visible, so a change after Final is a §1.6 MAJOR/MINOR event).

## §5.2 Envelope

Uplift of Ch.2.6 `ProtocolMessage`. All messages between `iFay_Runtime` and `Protocol_Engine` share one envelope.

```
ProtocolMessage {
  required version        : "CAP/MAJOR.MINOR"   // wire-visible protocol version (§1.6)
  required message_id     : uuid                // ULID/UUIDv7; time-ordered
  required message_type   : CapMsgType
  required timestamp      : timestamp
  required sender_id      : string              // runtime_id or terminal_id
  required body           : object              // structure determined by message_type
  optional correlation_id : uuid                // responses MUST set this to the request message_id
}

CapMsgType =
  | "AuthRequest" | "AuthResult"
  | "SessionRelease" | "SessionStateChanged"
  | "Heartbeat" | "HeartbeatAck"
  | "SessionTransferRequest"
  | "HandoverRequest" | "HandoverResponse" | "HandoverFailedNotification"
  | "ConfirmationRequest" | "ConfirmationResponse"     // new (§8 destructive-op)
  | "EmergencyOverride"                                  // new (§7.5 G0)
  | "Error"
```

**Envelope-level invariants**:

- **C-Msg-2 (well-formed envelope)**: any field missing or of the wrong type → reject with `E_INVALID_MESSAGE` (canonical name `[依赖 D-9]`); a `message_type` not in `CapMsgType` → reject likewise.
  - **Trace**: `R2.1`.
- **C-Msg-3 (version on the wire)**: `version` SHALL be `CAP/MAJOR.MINOR` only and SHALL be used for §1.6 negotiation; cross-MAJOR → `E_PROTOCOL_VERSION_UNSUPPORTED`.
  - **Trace**: `R9.4`.
- **C-Msg-4 (anti-replay)**: the CAP terminal SHOULD maintain a short-term seen-`message_id` cache and reject replays; `Heartbeat.sequence_number` monotonicity (§10) is the per-session anti-replay handle.
  - **Trace**: `R10.1`.

## §5.3 Message inventory

### §5.3.1 Existing messages (uplift of Ch.1.3)

| Message | Direction | Required `body` fields (summary) |
|---|---|---|
| `AuthRequest` | runtime → engine | `fay_id`, `resource_id`, `access_mode`, `credential`, **+ `mandate_ref` `[依赖 D-1]`**, **+ `operation_id?` for per-operation gating `[依赖 D-4]`** |
| `AuthResult` | engine → runtime | `request_id`, `status`, `session_id` (on success), `error_code` (on failure) |
| `SessionRelease` | runtime → engine | `session_id` (idempotent; §10) |
| `Heartbeat` / `HeartbeatAck` | both | `session_id`, `sequence_number` |
| `SessionStateChanged` | engine → runtime | `session_id`, `new_state`, `reason`, `details?` |
| `SessionTransferRequest` / `HandoverRequest` / `HandoverResponse` / `HandoverFailedNotification` | both | handover identifiers + target/credential + deadline (§9 handover) |

- **C-Msg-5 (mandate binding on AuthRequest)**: an `AuthRequest` SHALL carry the binding to a live Faying Mandate; absent / unverifiable → Default-Deny (`E_MANDATE_MISSING` / `E_MANDATE_INVALID`). The exact field carriage is `[依赖 D-1]`.
  - **Trace**: `R1.2`.

### §5.3.2 `ConfirmationRequest` / `ConfirmationResponse` (new — destructive-op confirmation, §8)

Carries the §8 structured disclosure to the Human-Prime view channel and returns the explicit per-operation human decision.

```
ConfirmationRequest (engine → runtime → HP view) {
  required confirmation_id   : uuid
  required session_id        : Session_ID
  required operation_id      : string
  required resource_id       : Resource_ID
  required mode              : AccessMode
  required criticality       : enum["high","critical"]
  required disclosure        : StructuredDisclosure   // §8.3 {reversibility, physical_risk,
                                                       //   affected_scope, terminal_attested_state, ...}
  required parameters_digest : bytes                  // binds the exact operation (§8 C-Dstr-10)
  required expires_at        : timestamp              // confirmation timeout (§8 C-Dstr-5)
}

ConfirmationResponse (HP → runtime → engine) {
  required confirmation_id   : uuid
  required decision          : enum["granted","denied"]
  required human_witness     : bytes                  // verifiable explicit-confirmation proof
}
```

- **C-Msg-6 (structured, non-free-text disclosure)**: a `ConfirmationRequest` for a `high`/`critical` operation SHALL carry the §8.3 structured disclosure fields; a free-text-only or fields-missing disclosure SHALL be rejected (`E_PROTOCOL`) and SHALL NOT be executed. The exact `criticality` field placement / source is `[依赖 D-4]` / `[依赖 D-5]`.
  - **Trace**: `R5.4`, `R5.5`.
- **C-Msg-7 (one confirmation = one operation)**: a `ConfirmationResponse` SHALL authorize exactly the operation identified by `(operation_id, resource_id, mode, parameters_digest)`; any divergence → reject (`E_PROTOCOL` / `E_OPAQUE_ACTION_BUNDLE`).
  - **Trace**: `R5.5`, `R10.2`.

### §5.3.3 `EmergencyOverride` (new — G0, §7.5)

The terminal-side realization of a Faying G0. Inbound to the CAP terminal (from the Revocation Registry push or a verified HP witness).

```
EmergencyOverride {
  required override_id    : uuid
  required target_fay_id  : Fay_ID
  required human_witness  : bytes        // verifiable HP witness, or authority/regulator signature
  optional reason         : string
}
```

- **C-Msg-8 (G0 is HP-only, non-forgeable)**: the CAP terminal SHALL accept an `EmergencyOverride` only with a verifiable Human-Prime witness (or authority/regulator signature per Faying RevokeReason); a G0 purportedly self-initiated by a Fay / runtime SHALL be rejected (`E_PROTOCOL`) and audited. On accept, it triggers §7.5 C-G0-1 (terminate all the Fay's sessions, refuse new requests, lock to `human`, request safe state where supported `[依赖 D-3]`).
  - **Trace**: `R4.3`, `R10.1`.

## §5.4 Error responses

Error responses use `message_type = "Error"` with an `ErrorBody` (§11.1). The full error table, consequence columns, and Default-Deny convergence are in **§11** (`06`). Canonical catch-all name (`E_PROTOCOL` vs `E_INVALID_MESSAGE`) is `[依赖 D-9]`; numeric companion codes are `[依赖 D-8]`.

---

## §5.5 Summary of hard constraints (this chapter)

| ID | One-line | Trace |
|---|---|---|
| C-Msg-1 | CBOR/JSON semantic + signature-input equivalence | R9.1, R9.2 |
| C-Msg-2 | Well-formed envelope or reject | R2.1 |
| C-Msg-3 | `CAP/MAJOR.MINOR` on the wire; cross-MAJOR reject | R9.4 |
| C-Msg-4 | Anti-replay (seen-id cache + heartbeat sequence) | R10.1 |
| C-Msg-5 | Mandate binding required on `AuthRequest` | R1.2 |
| C-Msg-6 | Structured (non-free-text) disclosure on confirmation | R5.4, R5.5 |
| C-Msg-7 | One confirmation = one exact operation | R5.5, R10.2 |
| C-Msg-8 | `EmergencyOverride` is HP-only, non-forgeable | R4.3, R10.1 |

## Open dependencies in this chapter
- `[依赖 D-1]` §5.3.1/§5.3 — `mandate_ref` carriage on `AuthRequest`.
- `[依赖 D-3]` §5.3.3 — safe-state request on `EmergencyOverride`.
- `[依赖 D-4]` §5.3 — placement of `operation_id` / `criticality` fields.
- `[依赖 D-5]` §5.3.2 — criticality source for the disclosure.
- `[依赖 D-8]` / `[依赖 D-9]` §5.4 — numeric companion codes; canonical catch-all name.

## Changelog
- (draft) Initial Protocol Messages chapter incl. ConfirmationRequest/Response + EmergencyOverride, H1-PROTO-01 Wave 2.
