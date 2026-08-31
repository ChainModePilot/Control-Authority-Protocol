# CAP Final §1 — Overview (Normative draft)

> **Status**: Drafted clause for H1-PROTO-01, Wave 2 (uplift of current Ch.0 + blueprint Ch.1). This chapter gives CAP's naming, positioning, design-goal skeleton, core differences from existing schemes, the one-sentence definition, the deliverable scope / non-goals, and the version-management contract. It is the semantic anchor for §2–§15. Modeled on `Faying-Protocol-1.0` §1.
> **Style**: hard constraints carry IDs `C-<area>-<n>` and `Trace: R<n>.<m>.<k>` to the requirements baseline (`02`). Items that depend on an open PO decision are marked `[依赖 D-n]` (adjudicated separately as PO decisions D-1…D-12); these placeholders are **not** pre-decided here.
> **Authority**: CAP is authoritative for itself (iAccord authoritative-source-map). It **consumes** Faying grades / Mandate / presence and the umbrella `ifay` terminology; it does not redefine them (see §3).

---

## §1.1 Protocol naming & positioning

Protocol specification name: **Control Authority Protocol**. The version is written **CAP/1.0**.

CAP is the **terminal-resource control-authority layer**: the gate that decides whether a specific Fay may exercise a specific access mode — and a specific possibly-irreversible operation — on a specific `Terminal_Resource`, under an accountable Faying Mandate. CAP sits **above** the OS access-control layer and **below** Fay business logic, and **behind** the Faying continuous-authorization gate.

> **Positioning relative to Faying**: Faying answers "under what presence, at what grade, may a Human Prime's iFay act at all"; CAP answers "given that the iFay may act, may *this terminal resource* be driven in *this mode* for *this operation*, and is the operation dangerous enough to require an explicit, present human." CAP is a specialized **Faying Relying Party** (§2.2).

## §1.2 One-sentence definition

> **CAP = the gate that decides whether a specific Fay may exercise a specific access mode — and a specific possibly-irreversible operation — on a specific `Terminal_Resource`, under an accountable Faying Mandate, with destructive actions held behind presence + explicit per-operation human confirmation.**

## §1.3 Design goals (D1–D10)

The table lists CAP's 10 design goals; **each maps one-to-one to a Requirement in `02`**, forming the bidirectional traceability anchor between this spec body and the requirements baseline (mirrors Faying §1.3's "Goal ↔ Requirement" discipline).

| Goal | Meaning | Requirement | Faying / origin |
|---|---|---|---|
| **D1 Accountability-Binding** | Every grant is attributable to exactly one Human Prime / Official_Post via a live Mandate | R1 | Faying R1 |
| **D2 Default-Deny** | Absence / ambiguity / unverifiability → denial; never a partial grant | R2 | Faying R2 |
| **D3 Offline-First Availability** | Loss of network does not strip a legitimately-granted, non-critical control | R3 | CAP-specific |
| **D4 Graded Control aligned to Faying** | Control authority gated by the consumed Faying grade G0–G4 | R4 | Faying R6 |
| **D5 Destructive-Operation Confirmation** | Irreversible / safety-critical operations require a present, informed, accountable human | R5 | Faying R13.4 |
| **D6 Atomic Handover** | At any instant a resource has at most one active controller | R6 | CAP-specific |
| **D7 Liveness / Zombie-Reclamation** | Failed sessions are reclaimed by dual determination | R7 | Faying R5 |
| **D8 Observable-Auditable-Revocable** | Transitions, grants, confirmations, revocations are observable and reconstructable | R8 | Faying R3 / R4 |
| **D9 Interoperability (CBOR + JSON)** | Semantically-equivalent encodings; CDDL authoritative | R9 | Faying R10 |
| **D10 AI-era Threat Resistance** | STRIDE + AI-era adversarial threats resisted at the protocol layer | R10 + R12 | Faying R9 / R13 |
| — (crypto evolvability) | Negotiable suites; PQC reservation | R11 | Faying R8 |
| — (resource access model) | Read-write-lock concurrency on access modes | R13 | CAP-specific |

> **Goal ↔ Requirement one-to-one principle**: `02` is the sole authoritative mapping between this spec body and the requirements. Any `R<n>.<m>.<k>` cited in later chapters must have its root requirement `<n>` in the table above.

## §1.4 Core differences from existing schemes

| Existing scheme | Solves | Does NOT solve — what CAP adds |
|---|---|---|
| OAuth 2.x scopes | App-level access authorization | No continuous presence, no per-operation reversibility awareness, no terminal-resource concurrency |
| Linux capabilities / POSIX caps | Process-level privilege partitioning | No accountable-human binding, no grade, no destructive-op confirmation |
| OS sandbox (seccomp / SELinux / AppArmor) | Confines a process's syscalls | No notion of "who, under what presence, at what Faying grade"; no handover |
| MDM / EMM | Device-fleet policy management | Coarse-grained, not per-operation, not presence-gated, no unique accountability per action |
| WebAuthn / FIDO2 | Hardware presence proof at a moment | Only the login moment; not continuous per-operation terminal control |

CAP does not compete with these — it adds **continuous, presence-gated, per-operation-reversibility-aware control with unique accountability** at the terminal-resource boundary.

## §1.5 Deliverable scope & non-goals

### §1.5.1 In scope (Normative)

CAP specifies, and only specifies:

- The six core capabilities (uplift of Ch.0.2): (1) offline `Authorization_Descriptor` issuance / storage / validation / revocation / renewal; (2) online `Trusted_Ticket`; (3) `Session` lifecycle; (4) control-authority `Handover_Policy` (3 policy types + atomicity); (5) `Liveness_Detection` (dual determination); (6) `Resource_Access_Mode` (read-write lock).
- The **Faying alignment layer** (new): mandate binding, grade mapping, presence gating, G0 emergency override (§7).
- The **operation-criticality & destructive-operation confirmation** model (new, §8).
- Protocol data objects, messages, cryptography, error handling, STRIDE + AI-era threat model, conformance.

### §1.5.2 Out of scope (Non-normative)

- Identity creation (FayID issuance — owned by the FayID system) and **the Faying authorization semantics themselves** (grade / Mandate / presence are *consumed*, not redefined).
- Fay reasoning / business logic; resource business semantics; terminal-driver internals; transport-layer details (TLS 1.3 itself); OS internals.
- UI/UX of the Human-Prime view channel; operations/deployment topology; audit-log canonical format (v1 exclusion).
- **`[依赖 D-7]`** — whether **fund transfer** (money movement) is in CAP scope or delegated to a dedicated payment protocol is an open scope decision; until resolved this chapter does not assert it either way.

### §1.5.3 Delivery model

**`[依赖 D-12]`** — single-file EN body (Faying EN pattern) vs. multi-file EN (current CAP). This chapter does not fix the choice; the recommendation (multi-file EN + Faying section-anchor discipline) is recorded in `08` D-12.

## §1.6 Version management & compatibility (new — Faying §1.7 pattern; closes part of G-11)

This section is the authoritative definition of CAP version numbering. The version number is part of the **interoperability contract**: it appears in the wire envelope, in schema filenames, and in negotiation, and determines whether two independent implementations can interoperate. Implementations **SHALL** interpret version numbers and negotiate per the rules below.

### §1.6.1 Three orthogonal version axes

| Axis | Question | Form | Carrier | Example |
|---|---|---|---|---|
| **Protocol version** (interop contract) | Can two implementations talk | `MAJOR.MINOR` | prose / wire envelope `version` field / schema filename | `CAP/1.0` |
| **Document status** (lifecycle) | Is this spec finalized | enum | front matter `status` | `Draft` / `Final` / `Deprecated` |
| **Editorial edition** (errata) | Which textual revision under the same protocol version | release date | release directory / front matter `date` | `2025-10-25` |

- **C-Ver-1 (only `MAJOR.MINOR` is wire-visible)**: Only the protocol version `CAP/MAJOR.MINOR` SHALL enter the wire format (envelope `version`, schema filename). Document status and editorial edition SHALL NOT enter the wire format; an implementation SHALL NOT change message handling because a peer's status or edition differs.
  - **Trace**: `R9.4`.

### §1.6.2 `MAJOR.MINOR` semantics (two levels, no PATCH)

- **MAJOR (breaking)**: any change that would make a peer built strictly to the old spec reject or misinterpret new messages (e.g., changing an authoritative CDDL field's semantics, removing a message type, redefining an error code's semantics, changing an absorbing state of a state machine). Bump MAJOR, reset MINOR to 0. Cross-MAJOR interop is **not** guaranteed.
- **MINOR (backward-compatible addition)**: additions an old implementation can safely ignore (optional fields, new messages / error codes, new AI-era threat entries, new appendix parameters). Increment MINOR; within one MAJOR, a higher MINOR SHALL remain backward compatible to `MAJOR.0`.

Decision test: *would a peer built strictly to the old spec reject or misinterpret new messages?* Yes → MAJOR; no but adds wire-visible content → MINOR; text-only → editorial edition (§1.6.4).

### §1.6.3 Version negotiation (Normative)

Reuses the §6.1 suite-negotiation approach (each side declares, the highest commonly-supported value is chosen). Implementations **SHALL**:

- **C-Ver-2 (declare + highest common MINOR)**: declare supported `CAP/MAJOR.MINOR` sets; on a shared MAJOR, select the highest commonly-supported MINOR; a higher-MINOR implementation SHALL serve a lower-MINOR peer.
  - **Trace**: `R9.4`.
- **C-Ver-3 (cross-MAJOR → reject)**: when no common MAJOR exists, reject (`E_PROTOCOL_VERSION_UNSUPPORTED`); no best-effort degraded interop.
  - **Trace**: `R9.4`.
- **C-Ver-4 (rolling upgrade preserves in-flight credentials)**: introducing a new MINOR or default suite via negotiation SHALL NOT invalidate `Authorization_Descriptor`s / `Trusted_Ticket`s valid before their `not_after`.
  - **Trace**: `R11.3`.
- **C-Ver-5 (unknown higher MINOR → interpret-and-ignore)**: on a same-MAJOR higher-MINOR message, interpret at the implementation's own highest MINOR and ignore unrecognized optional fields rather than reject.
  - **Trace**: `R9.4`.

### §1.6.4 Editorial edition & errata (Non-normative changes)

Purely textual errata (typos, examples, clarifications, fixed links) SHALL NOT change the protocol version; they are anchored by **release date** (new release directory; `schemas/cap-1.0.cddl` filename unchanged) and logged in the changelog as `Editorial`. An editorial edition SHALL carry only Non-normative changes; any change touching wire-format semantics SHALL bump MINOR or MAJOR.

### §1.6.5 Where the version number lands

| Landing | What is written | Notes |
|---|---|---|
| Wire envelope `version` (§5.2) | `"CAP/MAJOR.MINOR"` | MAJOR.MINOR only |
| Schema filename (`cap-1.0.cddl`) | follows `MAJOR.MINOR` | unchanged across editorial editions |
| content-type (§5.1) | `application/cap+cbor` / `+json` | identifies the protocol family |
| front matter `status` / `date` | doc status / edition | governance markers; not wire-visible |
| git tag | `cap-v<MAJOR.MINOR>-<status>` | a tag covers body + `vectors/` + `schemas/` together |

---

## §1.7 Summary of hard constraints (this chapter)

| ID | One-line | Trace |
|---|---|---|
| C-Ver-1 | Only `MAJOR.MINOR` is wire-visible | R9.4 |
| C-Ver-2 | Declare + select highest common MINOR | R9.4 |
| C-Ver-3 | Cross-MAJOR → reject (no best-effort) | R9.4 |
| C-Ver-4 | Rolling upgrade preserves in-flight credentials | R11.3 |
| C-Ver-5 | Unknown higher MINOR → interpret-and-ignore | R9.4 |

## Open dependencies in this chapter
- `[依赖 D-7]` §1.5.2 — fund-transfer scope (in CAP vs delegated to a payment protocol).
- `[依赖 D-12]` §1.5.3 — EN delivery model (single-file vs multi-file).

## Changelog
- (draft) Initial Overview chapter (naming, goals, scope, version contract) for the Faying-level uplift, H1-PROTO-01 Wave 2.
