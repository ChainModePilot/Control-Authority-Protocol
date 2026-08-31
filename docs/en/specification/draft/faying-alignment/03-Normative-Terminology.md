# CAP Final §3 — Terminology (Normative draft)

> **Status**: Drafted clause for H1-PROTO-01. This is a Faying-style normative terminology chapter. It (a) restates CAP's existing terms as normative, (b) introduces the new terms required by Faying alignment and destructive-operation confirmation, and (c) declares the alignment/authority order.
> **Authority order (binding)**: for umbrella terms (Human Prime, Fay, iFay, coFay, Host, MeriToken …) the **umbrella `ifay` blueprint terminology authority prevails**; for Faying terms (Mandate, grade, presence, attestation, revocation) the **`Faying-Protocol-1.0` Appendix A prevails** and CAP SHALL NOT redefine them; for CAP-proprietary terms (Authorization_Descriptor, Session, Resource_Access_Mode, Operation criticality …) **this specification prevails**. WHEN this chapter and a higher authority appear to conflict, the higher authority prevails and this chapter is corrected at the next review.

---

## 3.1 Notation

- Identifiers in `Code_Case` are protocol entities/structures defined normatively.
- `SHALL`/`SHALL NOT`/`SHOULD`/`MAY` follow RFC 2119 / RFC 8174.
- Each term marked **(consumed)** is owned by another protocol and referenced read-only; CAP SHALL use it exactly as defined upstream.

## 3.2 Umbrella terms (consumed — umbrella authority)

| Term | Meaning in CAP context |
|---|---|
| **Human Prime** (Natural_Person) | The natural person an iFay is the digital counterpart of; the ultimate accountability endpoint and authorization source for an iFay. CAP attributes every grant to a Human Prime. Never the "host". |
| **Official_Post** | The organizational post a coFay is attached to; the accountability/authorization source for a coFay. |
| **Fay** | Umbrella term for iFay and coFay; the holder of CAP credentials and requester of resource access. |
| **iFay / coFay** | Individual Fay (bound to a Human Prime) / Common Fay (bound to an Official_Post). |
| **Host** | **The terminal device or client software** that a Fay inhabits and controls. In CAP "host" SHALL refer only to the terminal, never to a person. |
| **iFay_Runtime** | The runtime that manages Fay instances and sends/receives CAP protocol messages on a Fay's behalf. |

## 3.3 Faying terms (consumed — Faying Appendix A authority)

> CAP is a **Faying Relying Party** for terminal resources. The following are referenced exactly as Faying defines them.

| Term | Meaning / CAP usage |
|---|---|
| **Mandate** (consumed) | The Faying continuous-authorization object ("who, under what presence, at what grade, delegated whom to do what"). A CAP `AuthRequest` SHALL be bound to a live Mandate (`mandate_ref`); CAP reads but never issues a Mandate. |
| **Authorization Grade `G0–G4`** (consumed) | Faying's graded authorization. `G0` Emergency Override (orthogonal HP force-recall), `G1` Strict, `G2` Moderate, `G3` Loose, `G4` Autonomous. Autonomy order `G0 < G1 < G2 < G3 < G4`. CAP maps each grade to permitted access modes and operation criticalities (§7.2). |
| **Presence Signal / freshness** (consumed) | The PD-signed proof that the Human Prime is present, with a grade-dependent `max_staleness_ms`. CAP uses presence freshness as a gate predicate, especially for `high`/`critical` operations. |
| **Attestation Chain** (consumed) | The reverse-traceable signature chain rooting a Mandate at one Human Prime. CAP uses it to confirm a single accountability endpoint (R1.3). |
| **Revocation Registry / RevokeReason** (consumed) | Faying's real-time revocation service + reason enum. WHEN a bound Mandate is revoked, CAP terminates referencing Sessions (R4.5). |
| **Emergency Override (G0)** (consumed) | The HP-only channel to unilaterally reclaim control. At the terminal CAP realizes it as immediate termination of all of a Fay's Sessions + lock to `human` (§7.3). |

## 3.4 CAP-proprietary terms (this specification is authoritative)

### 3.4.1 Existing terms (restated normative)

| Term | Normative meaning |
|---|---|
| **CAP** | Control Authority Protocol — the terminal-resource control-authority layer that decides whether a Fay may exercise a given access mode and operation on a given `Terminal_Resource` under an accountable Faying Mandate. |
| **Terminal_Resource** | A hardware device or client software on the terminal that can be accessed/operated under CAP control. |
| **Authorization_Descriptor** | The offline, terminal-stored, signed credential describing a Fay's authorized resource grants and validity. Core offline mechanism. |
| **Trusted_Ticket** | The online JWS credential supplementing offline authorization with real-time issuance/revocation. |
| **Grant** | An element of a credential binding a `resource_pattern` to a set of `AccessMode`s (and optional `constraints`). Extended (§4.2) with `min_grade` and `mandate_binding`. |
| **Descriptor_Issuer / Ticket_Issuer** | The trusted entity that issues `Authorization_Descriptor` / `Trusted_Ticket` under authorizer delegation. |
| **Descriptor_Validator** | The terminal component that verifies a credential's legitimacy and validity. |
| **Protocol_Engine** | The terminal component executing CAP core logic; together with `Descriptor_Validator` + session manager it is the **CAP terminal** (the Faying RP). |
| **Registration_Authority** | The trust-infrastructure root that registers terminals and distributes `Verification_Key`s. |
| **Verification_Key** | A terminal-held public key used to verify credential signatures. |
| **Session** | The control session from validation-pass to access-termination, bound one-to-one to a `Resource_ID`. |
| **Handover_Policy** | The decision rules (priority-script / ai-model / human-decision) for transferring control authority. |
| **Resource_Access_Mode** | The `{read, write, execute, configure}` model with read-write-lock concurrency semantics. |
| **Liveness_Detection** | The persistent-connection + application-heartbeat dual-determination mechanism for reclaiming zombie Sessions. |

### 3.4.2 New terms (introduced by this uplift)

| Term | Normative meaning | Trace |
|---|---|---|
| **CAP terminal** | The composite Faying Relying Party at the terminal: `Protocol_Engine` + `Descriptor_Validator` + session manager. The subject of all `SHALL` clauses. | R1–R13 |
| **Mandate binding** (`mandate_binding` / `mandate_ref`) | The verifiable association between a CAP credential/`AuthRequest` and a live Faying Mandate, establishing the accountability endpoint and the effective grade. | R1.1, R1.2 |
| **Effective grade** | The Faying grade of the bound Mandate at decision time, after applying presence-staleness tightening (R4.2). The grade CAP actually enforces. | R4.1, R4.2 |
| **`min_grade`** | The minimum Faying grade a `Grant` requires to be exercised. The CAP terminal rejects if `effective grade` is stricter than `min_grade` is not met. | R4.1 |
| **Operation** (`OperationDescriptor`, `operation_id`) | The minimal indivisible (atomic) control action a Fay requests on a resource within an access mode (e.g., `drone.takeoff`, `infusion.set_rate`, `file.delete`). The unit of criticality classification and confirmation. | R5.1, R5.5 |
| **Reversibility** | `reversible` (state restorable by the protocol's normal means), `hard-to-reverse` (restorable only with significant cost/side effects), `irreversible` (no protocol-level undo; e.g., physical actuation, data wipe, fund transfer). | R5.1 |
| **Physical_risk** | `none` / `property` (damage to assets) / `safety` (risk of injury) / `life-critical` (risk to life; e.g., medical, aviation). | R5.1 |
| **Criticality** | The derived class `{routine, sensitive, high, critical}` from `reversibility × physical_risk`; governs the confirmation gate. | R5.1–R5.3 |
| **Structured disclosure** | The terminal-attested, machine-parseable description shown before a `high`/`critical` operation (`{operation, resource_id, reversibility, physical_risk, affected_scope, terminal_attested_state}`). Free-text-only disclosure is non-conformant. | R5.4, R5.6 |
| **Per-operation confirmation** | An explicit human confirmation bound to exactly one atomic operation, obtained before execution. SHALL NOT be reused across operations or bundled. | R5.2, R5.5 |
| **Resource Capability Descriptor** | The terminal/driver-declared manifest assigning `reversibility`/`physical_risk`/`criticality` (and required `min_grade`) to each operation a resource exposes. The trusted classification source. `[需人类决策 D-5]` | R2.4, R5.6, R12.4 |
| **Emergency Override (terminal realization)** | The CAP-side effect of a Faying G0: immediate termination of all of a Fay's Sessions on the terminal + refusal of new requests + lock to `human`. | R4.3 |
| **Attention-drift hold** | The state in which a `high`/`critical` operation arriving during a human-inattention window is held pending re-confirmation rather than passed. | R12.5 |
| **Human-Prime view channel** | The audit/confirmation channel that surfaces `high`/`critical` decisions and security-critical errors to the Human Prime; priority-bucketed and rate-limited (anti-flood). | R5.7, R12.7 |

## 3.5 Alignment quick reference (CAP ↔ Faying)

| CAP concept | Faying counterpart | Relationship |
|---|---|---|
| CAP terminal | Relying Party (RP) | CAP terminal **is** an RP specialized for terminal resources |
| `mandate_ref` / Mandate binding | Mandate | CAP consumes; never issues |
| `min_grade` / effective grade | Grade G0–G4 | CAP enforces upstream grade |
| Presence gate (R4.2, R5.2) | Presence Signal + `max_staleness_ms` | CAP reuses Faying budgets (Appendix B) |
| Emergency Override realization | G0 Emergency Override | CAP realizes the terminal-side effect |
| `high`/`critical` confirmation | T4 "high-sensitivity ≥ G1" + DISCLOSURE | CAP specializes for physical/irreversible ops |
| Anti-opaque-bundle (R5.5) | T2 Opaque Action Bundle / Atomic_Action | CAP applies to hardware commands |
| Mandate `Revoked` → session kill | Symmetric Revocation | CAP is the enforcement endpoint |

---

## Changelog
- (draft) Initial normative terminology with Faying alignment, H1-PROTO-01.
