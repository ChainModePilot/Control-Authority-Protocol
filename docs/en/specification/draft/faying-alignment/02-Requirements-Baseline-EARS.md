# CAP Requirements Baseline (EARS) — Traceability Anchor

> **Status**: Working draft for H1-PROTO-01. This is the **sole authoritative mapping** between the CAP Final specification body and its requirements, mirroring Faying §1.3's "Goal ↔ Requirement one-to-one" discipline. Every hard constraint in the drafted normative chapters (`03`–`06`) cites `Trace: R<n>.<m>.<k>` back to a clause here.
> **EARS forms used**: *Ubiquitous* ("The CAP terminal SHALL …"), *Event* ("WHEN …, the CAP terminal SHALL …"), *State* ("WHILE …, the CAP terminal SHALL …"), *Unwanted* ("IF …, THEN the CAP terminal SHALL …"), *Optional* ("WHERE …, the CAP terminal SHALL …").
> **Convention**: `SHALL` = absolute (RFC 2119 MUST); `SHALL NOT` = MUST NOT; `SHOULD`/`MAY` as RFC 2119. "CAP terminal" = the `Protocol_Engine` + `Descriptor_Validator` + session manager on the terminal acting as the Faying Relying Party for terminal resources.
> Open architecture decisions are tagged `[需人类决策 D-n]` and aggregated in `README.md`.

---

## Goal ↔ Requirement map

| Design goal (outline §1.3) | Requirement | Faying counterpart |
|---|---|---|
| D1 Accountability-Binding | R1 | Faying R1 (Unique Accountability) |
| D2 Default-Deny | R2 | Faying R2 |
| D3 Offline-First Availability | R3 | — (CAP-specific) |
| D4 Graded Control aligned to Faying | R4 | Faying R6 (Graded Authorization) |
| D5 Destructive-Operation Confirmation | R5 | Faying R13.4 (Attention-Drift / high-sensitivity ≥ G1) |
| D6 Atomic Handover | R6 | — (CAP-specific) |
| D7 Liveness / Zombie-Reclamation | R7 | Faying R5 (Continuous Calibration) |
| D8 Observable-Auditable-Revocable | R8 | Faying R3 / R4 |
| D9 Interoperability (CBOR+JSON) | R9 | Faying R10 |
| D10 AI-era Threat Resistance | R10 (STRIDE) + R12 (AI-era) | Faying R9 / R13 |
| — (crypto evolvability) | R11 | Faying R8 (PQC) |
| — (resource access model) | R13 | — (CAP-specific) |

---

## R1 — Accountability binding to a Faying Mandate
*Goal D1.* Every control authority CAP grants must be unambiguously attributable to exactly one Human Prime (Natural_Person) or Official_Post, via a live Faying Mandate.

- **R1.1** The CAP terminal SHALL treat each accepted `AuthRequest` as bound to exactly one accountability endpoint (Human Prime / Official_Post), reconstructable from the audit record as the tuple `(descriptor_id, mandate_ref, subject_fay_id, terminal_id, access_mode, operation_id?, ts)`.
- **R1.2** WHEN an `AuthRequest` is not accompanied by a verifiable binding to a live Faying Mandate, the CAP terminal SHALL reject it under Default-Deny (`E_MANDATE_MISSING`). `[需人类决策 D-1: exact binding model]`
- **R1.3** IF the bound Faying Mandate's attestation chain does not resolve to a single accountability endpoint, THEN the CAP terminal SHALL reject the request (`E_MANDATE_INVALID`) and write the event to the audit log.
- **R1.4** The CAP terminal SHALL NOT create or continue a Session whose accountable endpoint cannot be determined.

## R2 — Default-Deny
*Goal D2.* Absence, ambiguity, or unverifiability resolves to denial.

- **R2.1** IF any field required for a control decision is missing, malformed, or unverifiable, THEN the CAP terminal SHALL reject the request and SHALL NOT grant partial access.
- **R2.2** IF a required dependency (revocation status, presence freshness, mandate validity, operation-criticality classification) is unavailable, THEN the CAP terminal SHALL converge to denial rather than optimistic pass.
- **R2.3** The CAP terminal SHALL grant access only when **all** decision predicates (R1, R3–R6, R13) hold simultaneously.
- **R2.4** IF an operation's criticality cannot be determined, THEN the CAP terminal SHALL treat it as `critical` (the strictest class) and apply R5. `[需人类决策 D-5]`

## R3 — Offline-first availability
*Goal D3.* Loss of network SHALL NOT strip a previously-authorized Fay of legitimately granted, non-critical control.

- **R3.1** WHILE the terminal is offline, the CAP terminal SHALL continue to validate `Authorization_Descriptor`s locally and honor grants whose grade/criticality requirements can be satisfied offline.
- **R3.2** IF an operation's criticality requires online revocation confirmation or fresh online presence that cannot be obtained offline, THEN the CAP terminal SHALL reject that operation while still serving offline-satisfiable operations.
- **R3.3** The CAP terminal SHALL bound offline trust by descriptor validity ceilings (≤ 90 days offline, ≤ 7 days for converted tickets) so revocation delay is capped.

## R4 — Graded control aligned to Faying G0–G4
*Goal D4.* CAP control authority is gated by the Faying grade carried by the bound Mandate; CAP consumes the grade, it does not redefine it.

- **R4.1** The CAP terminal SHALL read the Faying grade `G ∈ {G0,G1,G2,G3,G4}` of the bound Mandate and SHALL reject any access whose grant `min_grade` is stricter (higher security / lower autonomy) than `G` (`E_GRADE_INSUFFICIENT`). Autonomy order per Faying §7.1: `G0 < G1 < G2 < G3 < G4`.
- **R4.2** WHEN the bound Mandate's presence signal is stale for the current grade (per Faying Appendix B `max_staleness_ms`: G1≤5s, G2≤60s, G3 key-step intent, G4≤30min), the CAP terminal SHALL tighten — suspending sessions whose criticality requires fresher presence — and SHALL NOT loosen (downgrade-monotonicity).
- **R4.3** WHEN the CAP terminal receives a Faying **G0 Emergency Override** for a Fay, it SHALL immediately terminate all of that Fay's active Sessions on the terminal, refuse new `AuthRequest`s from that Fay until re-bootstrapped, and route control to `human` (`E_EMERGENCY_OVERRIDE_ACTIVE`).
- **R4.4** The CAP terminal SHALL NOT allow a Session's effective grade or granted modes to be elevated within its lifecycle; a higher grade/mode requires a new `AuthRequest` under a fresh Mandate (no runtime escalation interface).
- **R4.5** WHEN the bound Mandate transitions to `Revoked` (any Faying RevokeReason), the CAP terminal SHALL terminate all Sessions referencing it (`E_MANDATE_REVOKED`).

## R5 — Destructive / irreversible operation confirmation
*Goal D5.* The most dangerous gate. Irreversible or safety/life-critical operations require the strongest evidence of a present, informed, accountable human.

- **R5.1** The CAP terminal SHALL classify every operation by `reversibility ∈ {reversible, hard-to-reverse, irreversible}` and `physical_risk ∈ {none, property, safety, life-critical}`, yielding `criticality ∈ {routine, sensitive, high, critical}`.
- **R5.2** WHEN an operation's `criticality = critical` (irreversible, or life-critical), the CAP terminal SHALL require **all** of: (a) bound Mandate grade ≥ G1 Strict; (b) a presence proof fresh within the G1 window (≤ 5 s); (c) an explicit, per-operation human confirmation obtained **before** execution; otherwise reject (`E_HUMAN_CONFIRMATION_REQUIRED` / `E_GRADE_INSUFFICIENT` / `E_PRESENCE_STALE`).
- **R5.3** WHEN an operation's `criticality = high` (hard-to-reverse, or safety-risk), the CAP terminal SHALL require grade ≥ G2 and a fresh presence proof for the grade, and SHALL require explicit human confirmation unless the bound grade is G1 with a current presence proof. `[需人类决策 D-6: high-class confirmation policy]`
- **R5.4** Before executing any `high` or `critical` operation, the CAP terminal SHALL present a **structured disclosure** containing at least `{operation, resource_id, reversibility, physical_risk, affected_scope, terminal_attested_state}`; free-text-only confirmation SHALL be rejected.
- **R5.5** The CAP terminal SHALL bind one human confirmation to exactly one atomic operation; IF a single confirmation is presented for a bundle containing a `high`/`critical` operation, THEN the CAP terminal SHALL reject it (`E_OPAQUE_ACTION_BUNDLE`).
- **R5.6** The CAP terminal SHALL NOT rely on Fay-supplied state for the disclosure of `high`/`critical` operations; it SHALL use terminal/driver-attested state (anti-CT1).
- **R5.7** The CAP terminal SHALL write every `high`/`critical` confirmation decision (granted/denied/timeout) to the audit log and SHALL surface it on the Human-Prime view channel.
- **R5.8** The criticality gate SHALL be history-independent: prior benign behavior SHALL NOT lower the requirements of R5.2–R5.5 (anti-CT6).
- **R5.9** IF human confirmation is not received within the confirmation timeout, THEN the CAP terminal SHALL default to denial (`E_HUMAN_CONFIRMATION_TIMEOUT`) and SHALL NOT execute the operation.

## R6 — Atomic control-authority handover
*Goal D6.* At any instant a resource has at most one active controller.

- **R6.1** The CAP terminal SHALL execute handover so that external observers never see two Sessions active on one `Resource_ID` simultaneously.
- **R6.2** IF any pre-release step fails, THEN the CAP terminal SHALL roll the source Session back to `active`; IF a post-release step fails, THEN it SHALL leave the resource idle and SHALL NOT auto-restore the source Session.
- **R6.3** WHEN a handover targets a `human` (Fay-to-Human), the CAP terminal SHALL reuse the §8 confirmation gate where the handed-over control includes `high`/`critical` operations.
- **R6.4** WHILE a resource is `handover_pending`, the CAP terminal SHALL serialize/reject concurrent handover requests for that resource.

## R7 — Liveness and zombie-session reclamation
*Goal D7.*

- **R7.1** The CAP terminal SHALL determine a Session failed only when both the persistent connection is broken beyond the heartbeat-timeout threshold **and** `now − last_heartbeat_at` exceeds it (dual determination).
- **R7.2** WHEN a Session is determined failed, the CAP terminal SHALL terminate it, revoke OS access, and release the resource.
- **R7.3** WHEN the bound Mandate's presence becomes stale such that an active Session's criticality is no longer permitted (R4.2), the CAP terminal SHALL suspend or terminate that Session.
- **R7.4** A terminated Session SHALL NOT be auto-restored; a new `AuthRequest` is required.

## R8 — Observable, auditable, revocable
*Goal D8.*

- **R8.1** The CAP terminal SHALL emit `SessionStateChanged` for every observable state transition to the owning `iFay_Runtime`.
- **R8.2** The CAP terminal SHALL record, for every grant/deny/handover/confirmation/revocation, an audit entry sufficient to reconstruct the R1.1 accountability tuple and the decision reason code.
- **R8.3** WHEN a revocation statement reaches the terminal, the CAP terminal SHALL reject subsequent validations of the revoked credential immediately and terminate referencing Sessions.
- **R8.4** Revocation reachability SHALL be ≥ establishment reachability (a path that could grant control must have an at-least-as-reachable path to revoke it).

## R9 — Interoperability (wire format)
*Goal D9.*

- **R9.1** The CAP terminal SHALL accept the protocol's CBOR primary encoding and the JSON fallback as semantically equivalent for every message and credential.
- **R9.2** The CAP terminal SHALL compute signature inputs over RFC 8949 **deterministic** CBOR (offline credentials) / RFC 7515 JWS (online tickets), so independent implementations verify identically.
- **R9.3** The protocol SHALL be defined by an authoritative CDDL schema (`schemas/cap-1.0.cddl`); WHEN CDDL and prose conflict, CDDL prevails.
- **R9.4** WHEN negotiating versions, implementations SHALL select the highest commonly-supported `CAP/MAJOR.MINOR`; cross-MAJOR SHALL be rejected (no best-effort interop).

## R10 — STRIDE security properties
*Goal D10.* (Spoofing/Tampering/Repudiation/Information-Disclosure/DoS/Elevation — full criteria authored with §12 next wave.)

- **R10.1** The CAP terminal SHALL reject requests whose descriptor signature, Mandate binding, or channel binding fails verification (anti-Spoofing/Tampering).
- **R10.2** IF an `AuthRequest`'s operation or mode differs from what a prior confirmation was bound to, THEN the CAP terminal SHALL reject execution (anti-Tampering/Elevation).
- **R10.3** Error responses SHALL NOT expose key fingerprints, internal IDs, or resource topology (anti-Information-Disclosure).
- **R10.4** The CAP terminal SHALL enforce session/connection/storage quotas and rate limits (anti-DoS).

## R11 — Cryptographic evolvability
*(crypto goal.)*

- **R11.1** The CAP terminal SHALL support the mandatory suite (ed25519, ecdsa-p256-sha256).
- **R11.2** The protocol SHALL reserve algorithm-suite negotiation bits for post-quantum algorithms (e.g., ML-DSA-65) without a wire-breaking change (parity with Faying G8).
- **R11.3** WHEN a new default suite is introduced via negotiation, existing valid credentials SHALL remain verifiable until their `not_after`.

## R12 — AI-era adversarial threat resistance (CAP-specific)
*Goal D10.* Anchors the CT1–CT7 catalog (outline §12.3). Full per-threat criteria authored with §12 next wave; the clauses below are the protocol-level commitments the drafted chapters already rely on.

- **R12.1** (CT1 Perception Spoofing) The CAP terminal SHALL base `high`/`critical` disclosures on terminal/driver-attested state, not Fay-supplied state.
- **R12.2** (CT2 Opaque Bundle) The CAP terminal SHALL reject bundling of `high`/`critical` operations under one confirmation (see R5.5).
- **R12.3** (CT3 Simulated-Operation Confusion) WHEN a `high`/`critical` operation is requested via a GUI-grounding/`execute` path, the CAP terminal SHALL apply R5.2/R5.3 with no exemption for "automation convenience".
- **R12.4** (CT4 Prompt Injection) The CAP terminal SHALL treat Fay-originated criticality claims as untrusted; terminal/driver-declared criticality prevails; unknown → critical (R2.4).
- **R12.5** (CT5 Presence-Replay / Attention-Drift) The CAP terminal SHALL require per-grade fresh presence and SHALL hold `high`/`critical` operations arriving during an attention-drift window until re-confirmation.
- **R12.6** (CT6 Cultivated Trust) See R5.8 (history-independence).
- **R12.7** (CT7 Confirmation/Audit Flooding) The CAP terminal SHALL priority-bucket the confirmation/audit channel and SHALL NOT drop `critical`-bucket events under rate limiting.

## R13 — Resource access model (read-write lock)
*(CAP-specific.)*

- **R13.1** The CAP terminal SHALL enforce the read-write-lock matrix: `read` is shareable; `write`/`execute`/`configure` are exclusive and mutually incompatible with any other occupancy.
- **R13.2** The CAP terminal SHALL NOT auto-queue an incompatible request; it SHALL return `E_RESOURCE_BUSY` (handover is the explicit path).
- **R13.3** The CAP terminal SHALL treat access modes as independent (no hierarchical inclusion); each mode SHALL be independently authorized.
- **R13.4** The CAP terminal SHALL confine a Session to the `granted_modes` derived from the matched `Grant`, which SHALL be a subset of the bound Mandate's scope. `[需人类决策 D-1]`

---

## Changelog
- (draft) Initial EARS baseline for the Faying-level uplift, H1-PROTO-01.
