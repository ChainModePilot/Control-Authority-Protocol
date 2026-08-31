# CAP Final §2 — Architecture (Normative draft)

> **Status**: Drafted clause for H1-PROTO-01, Wave 2 (uplift of current Ch.1; **§2.2 is new and closes the P0 gap G-1**). Defines CAP's stack position, its relationship to Faying as a Relying Party, the roles and trust chains, the external interface contracts, and the design principles. Modeled on `Faying-Protocol-1.0` §2.
> **Style**: hard constraints carry IDs `C-<area>-<n>` and `Trace: R<n>.<m>.<k>` to `02`. Open PO decisions are marked `[依赖 D-n]` (see `08`).

---

## §2.1 Protocol stack position

CAP sits at the **terminal Relying-Party boundary**: above the OS access-control layer, below Fay business logic. **Every resource operation passes the CAP gate, and the CAP gate itself sits behind the Faying gate.**

```
┌───────────────────────────────────────────────┐
│ Fay business logic (iFay_Runtime drives a Fay)  │
└───────────────────────┬───────────────────────┘
                        │ every resource operation
                        ▼
┌───────────────────────────────────────────────┐
│ Faying continuous-authorization gate (consumed)│  ← Mandate / grade / presence / revocation
└───────────────────────┬───────────────────────┘
                        ▼
┌───────────────────────────────────────────────┐
│ CAP gate  (this protocol)                       │  ← descriptor validity, grade floor, rwlock,
│  P1..P7 composite control decision (§7.1)       │     operation-criticality confirmation (§8)
└───────────────────────┬───────────────────────┘
                        ▼
┌───────────────────────────────────────────────┐
│ OS access-control layer  →  Terminal_Resource   │
└───────────────────────────────────────────────┘
```

- **C-Stack-1 (no operation bypasses the CAP gate)**: The CAP terminal SHALL NOT permit any Fay-originated operation on a `Terminal_Resource` to reach the OS access-control layer without a passing CAP decision (§7.1 P1–P7); any such operation SHALL be treated as a rogue action and rejected under Default-Deny. *(Restated from `04` C-Stack-1 for architectural context.)*
  - **Trace**: `R2.1`, `R2.3`.
- **C-Stack-2 (separation of duties with transport)**: Channel confidentiality and peer identity are borne by TLS 1.3 / optional mTLS; CAP SHALL NOT reimplement them. CAP is responsible only for terminal-resource control-authority semantics.
  - **Trace**: `R9.2`.

## §2.2 Relationship to Faying (new — P0, closes G-1)

CAP is a **specialized Faying Relying Party (RP)** for terminal resources. This section defines, precisely, what CAP **consumes** from Faying and what CAP **owns**.

### §2.2.1 What CAP consumes from Faying (read-only)

| Consumed | Use in CAP | CAP SHALL NOT |
|---|---|---|
| **Mandate** (`mandate_ref`) | Establishes the single accountable endpoint + the effective grade for a control decision (§7.2) | issue, mint, or mutate a Mandate |
| **Authorization grade G0–G4** | Gates permitted access modes / operation criticalities (§7.3) | redefine the grade ladder or its autonomy order |
| **Presence signal + `max_staleness_ms`** | Gate predicate, especially for `high`/`critical` operations (§7.4, §8.2) | invent its own presence budgets (reuses Faying Appendix B) |
| **Attestation chain** | Confirms exactly one accountable Human Prime / Official_Post (R1.3) | shorten or bypass chain verification |
| **Revocation Registry / RevokeReason** | Terminates referencing Sessions on revocation (R4.5) | treat a revoked Mandate as live |
| **Emergency Override (G0)** | Realized at the terminal as immediate reclamation (§7.5) | originate a G0 itself (HP-only) |

### §2.2.2 What CAP owns (authoritative)

`Authorization_Descriptor` / `Trusted_Ticket` / `Grant`; `Session` lifecycle and `Liveness_Detection`; `Resource_Access_Mode` and the read-write lock; control-authority `Handover`; and the **operation-criticality classification + destructive-operation confirmation** model (§8).

### §2.2.3 The binding (CAP terminal as RP)

- **C-Bind-1 (bound, live Mandate required)**: every accepted `AuthRequest` SHALL carry a verifiable binding to a live Faying Mandate; absence / expiry / unverifiability → Default-Deny (`E_MANDATE_MISSING` / `E_MANDATE_INVALID`). *(Authoritative statement in `04`; referenced here for the architecture picture.)*
  - **Trace**: `R1.2`, `R1.3`.

> **`[依赖 D-1]` — the exact binding model** (Derivation / Reference+live-check / Embedded-grade) is an open PO decision (`08` D-1). This chapter is written to be correct under any of the three; the concrete `mandate_ref` carriage and the data-model fields (§4.2) are frozen only once D-1 is resolved. **Do not assume a model here.**

## §2.3 Entity relationships

```
        Faying external roles (referenced, not owned by CAP)
   Human Prime ── Mandate Service ── Presence Channel ── Revocation Registry
        │  (accountable endpoint)            │ (presence freshness)   │ (revocation)
        ▼                                    ▼                        ▼
  ┌─────────────────────────── CAP terminal (the Faying RP) ───────────────────────────┐
  │  Protocol_Engine  +  Descriptor_Validator  +  Session manager  +  local revocation  │
  └───────────────▲────────────────────────────────────────────────┬──────────────────┘
                  │ AuthRequest (+ mandate_ref + presence proof)     │ OS access-control delivery
        iFay_Runtime (drives a Fay)                          Terminal_Resource (HW / SW)

  Descriptor_Issuer / Ticket_Issuer ──issue──▶ credentials      Registration_Authority ──VerificationKey──▶ terminal
```

CAP roles (uplift of Ch.1.1): **`Descriptor_Issuer` / `Ticket_Issuer`**, **`Descriptor_Validator`**, **`Protocol_Engine`**, **`Registration_Authority`**, **`iFay_Runtime`**, the **terminal** (= Protocol_Engine + Descriptor_Validator + session manager, i.e. the **CAP terminal**). Faying entities (Human Prime, Mandate Service, Presence Channel, Revocation Registry) are **referenced external roles**; CAP maps onto the Faying RP/Authorization-Context picture.

## §2.4 Trust chains

CAP carries two CAP-native trust paths (uplift of Ch.1.2) **plus** the consumed Faying accountability path.

- **Authorization trust path** — determines whether a Fay is authorized for a resource: `Authorizer → Descriptor_Issuer → (signed Authorization_Descriptor) → Descriptor_Validator`.
- **Key trust path** — determines whether the terminal trusts an issuer signature: `Registration_Authority (anchor) → Verification_Key → terminal key store → Descriptor_Validator`.
- **Faying accountability path (new)** — the attestation chain the bound `mandate_ref` resolves to a single Human Prime / Official_Post (consumed; §2.2.1).

- **C-Bind-2 (single accountability endpoint)**: The CAP terminal SHALL confirm the bound Mandate's attestation chain resolves to exactly one accountable endpoint; multiple or none → reject (`E_MANDATE_INVALID`) + audit. *(Authoritative in `04`.)*
  - **Trace**: `R1.1`, `R1.3`.

## §2.5 External interface contracts

Uplift of Ch.1.3 — CAP interacts with four external systems, **plus** a new Faying-context interface.

| Interface | Direction | Contract (summary) |
|---|---|---|
| **iFay_Runtime ↔ Protocol_Engine** | bidirectional | `AuthRequest` / `AuthResult` / `SessionRelease` / `Heartbeat`/`Ack` / `SessionStateChanged` / handover + **new** `ConfirmationRequest`/`Response`, `EmergencyOverride` (§5.3). Persistent connection; TLS; serialization per the CDDL schema. |
| **Protocol_Engine ↔ terminal OS** | local bidirectional | access-control delivery, resource-state query, resource-event subscription (synchronous delivery, async events, OS-native protection). |
| **Protocol_Engine ↔ hardware drivers** | indirect | All hardware control is forwarded through the OS; Protocol_Engine MUST NOT talk to drivers directly. Drivers SHOULD implement hardware-level locking. *(This is the path the `ResourceCapabilityDescriptor` of §8 rides on — see `[依赖 D-5]`.)* |
| **Registration_Authority → terminal** | unidirectional | `VerificationKeyDistribution` / `VerificationKeyRevocation` / `RegistrationStatusUpdate` over TLS/mTLS. |
| **Faying-context interface (new)** | inbound with each `AuthRequest` | how a `mandate_ref` + presence proof accompany an `AuthRequest`, and how revocation/G0 pushes reach the terminal. **`[依赖 D-1]`** (carriage shape). |

## §2.6 Design principles (new — Faying §2.5 pattern)

The 7 protocol-layer principles CAP always follows. Each has a protocol-layer landing and EARS sub-clauses; principles are normative-grade commitments and SHALL NOT be weakened by any implementation for "engineering convenience".

| Principle | Protocol-layer landing | Trace |
|---|---|---|
| **Default-Deny** | No bound live Mandate / missing field / unavailable dependency → reject; never a partial grant | R2.1, R2.2, R2.3 |
| **Least-Privilege** | `granted_modes ⊆ Grant ⊆ bound Mandate scope`; access modes are independent (no hierarchical inclusion) | R13.3, R13.4 |
| **Continuous-Verification** | Every operation re-checks grade + presence freshness + revocation; trust is never cached past `max_staleness_ms` | R4.2, R8.3 |
| **Auditable-by-Construction** | Every grant / deny / handover / confirmation / revocation yields an audit entry reconstructing the R1.1 accountability tuple | R8.2 |
| **Human-Override** | G0 Emergency Override always has highest priority; HP-only; non-forgeable (§7.5) | R4.3 |
| **Graceful-Degradation** | Presence loss → tighten (suspend high-criticality), never loosen (downgrade-monotonicity) | R4.2, R7.3 |
| **Irreversibility-Caution (CAP-specific 7th)** | The entire safety budget for irreversible / life-critical operations is spent *before* execution (§8); the protocol never claims to undo a fired actuator | R5.2 |

- **C-Arch-1 (principles not weakenable)**: No reference implementation, SDK, or conformance suite SHALL weaken any of the 7 principles above; a change requires a new protocol version via the §1.6 process.
  - **Trace**: `R2.3`.

---

## §2.7 Summary of hard constraints (this chapter)

| ID | One-line | Trace |
|---|---|---|
| C-Stack-1 | No operation bypasses the CAP gate | R2.1, R2.3 |
| C-Stack-2 | Separation of duties with TLS | R9.2 |
| C-Bind-1 | Bound, live Mandate required (ref. `04`) | R1.2, R1.3 |
| C-Bind-2 | Single accountability endpoint (ref. `04`) | R1.1, R1.3 |
| C-Arch-1 | Design principles not weakenable | R2.3 |

## Open dependencies in this chapter
- `[依赖 D-1]` §2.2.3 / §2.5 — exact CAP↔Faying binding model and the `mandate_ref` carriage.
- `[依赖 D-5]` §2.5 — driver capability manifest as the criticality source rides the hardware-driver (indirect) interface.

## Changelog
- (draft) Initial Architecture chapter incl. the new §2.2 Faying-RP relationship, H1-PROTO-01 Wave 2.
