# CAP Final §8 — Destructive & Irreversible Operation Confirmation (Normative draft)

> **Status**: Drafted clause for H1-PROTO-01 (closes the headline gap G-2). This is the single most safety-critical chapter in CAP: it is the gate that stands between an authorized-but-misaligned Fay and an irreversible physical action (drone, medical device, lock, data wipe). It specializes Faying's "high-sensitivity actions ≥ G1 Strict" (T4) + "Atomic_Action / no opaque bundle" (T2) + "structured DISCLOSURE" (T3) for the physical-control domain.
> **Style**: hard constraints `C-Dstr-<n>`, `Trace: R<n>.<m>.<k>` to `02`.
> **Core stance**: *irreversibility is irreversible* — the protocol cannot undo a fired actuator, a wiped disk, or a transferred fund. Therefore the entire safety budget is spent **before** execution, on proving a present, informed, accountable human authorized **this exact operation**.

---

## 8.1 Operation criticality classification

Every Fay-requested operation on a `Terminal_Resource` SHALL be assigned two orthogonal attributes and a derived criticality.

**Reversibility** — can the protocol's normal means restore the prior state?
- `reversible` — yes, routinely (e.g., set screen brightness, open a read stream).
- `hard-to-reverse` — only with significant cost/side effects (e.g., send an email, overwrite a file with backup).
- `irreversible` — no protocol-level undo (e.g., physical actuation, secure-erase, fund transfer, firmware flash).

**Physical_risk** — worst-case consequence to the physical world:
- `none` — informational only.
- `property` — possible damage to assets.
- `safety` — possible injury.
- `life-critical` — possible loss of life (medical, aviation, automotive, industrial).

**Derived criticality** (`C-Dstr-1`):

| reversibility ＼ physical_risk | none | property | safety | life-critical |
|---|---|---|---|---|
| **reversible** | routine | sensitive | high | high |
| **hard-to-reverse** | sensitive | high | high | critical |
| **irreversible** | high | high | critical | critical |

- **C-Dstr-1 (classification is mandatory)**: The CAP terminal SHALL classify every operation per the matrix above before any execution decision; an operation whose classification is unavailable SHALL be treated as `critical` (§8.4).
  - **Trace**: `R5.1`, `R2.4`.

## 8.2 Confirmation rules by criticality

The gate combines three independent factors — **grade floor**, **presence freshness**, **explicit human confirmation** — escalating with criticality. All conditions for a class are conjunctive (Default-Deny on any miss).

| Criticality | Grade floor | Presence freshness | Per-operation human confirmation | Structured disclosure |
|---|---|---|---|---|
| **routine** | ≥ G4 | per grade | not required | not required |
| **sensitive** | ≥ G3 | per grade | not required (SHOULD log) | not required |
| **high** | ≥ G2 `[D-6]` | fresh for grade | **required** unless G1 + current presence `[D-6]` | **required** |
| **critical** | **G1 only** | **≤ 5 s** | **always required, per operation** | **required** |

- **C-Dstr-2 (critical gate)**: WHEN `criticality = critical`, the CAP terminal SHALL require **all** of: (a) effective grade = G1 Strict; (b) a presence proof ≤ 5 s old; (c) an explicit human confirmation bound to this exact operation, obtained **before** execution. IF any is missing, THEN reject (`E_HUMAN_CONFIRMATION_REQUIRED` / `E_GRADE_INSUFFICIENT` / `E_PRESENCE_STALE`) and SHALL NOT execute.
  - **Trace**: `R5.2`, `R12.3`.
- **C-Dstr-3 (high gate)**: WHEN `criticality = high`, the CAP terminal SHALL require effective grade ≥ G2 and presence fresh for the grade, and SHALL require explicit human confirmation unless the effective grade is G1 with a current presence proof. `[需人类决策 D-6]`
  - **Trace**: `R5.3`.
- **C-Dstr-4 (criticality is history-independent)**: The CAP terminal SHALL NOT lower any requirement of C-Dstr-2/3 based on the Fay's prior behavior, reputation, or session age (anti-cultivated-trust).
  - **Trace**: `R5.8`, `R12.6`.
- **C-Dstr-5 (confirmation freshness & timeout)**: A per-operation confirmation SHALL be valid only for the single operation it was issued for and SHALL expire at a confirmation timeout; IF the timeout elapses without confirmation, THEN the CAP terminal SHALL deny (`E_HUMAN_CONFIRMATION_TIMEOUT`).
  - **Trace**: `R5.9`.
- **C-Dstr-6 (attention-drift hold)**: WHEN a `high`/`critical` operation arrives while the Human-Prime view channel has been inactive beyond the attention-drift threshold (Appendix B), the CAP terminal SHALL hold the operation pending fresh confirmation rather than passing it.
  - **Trace**: `R12.5`.

## 8.3 Pre-execution structured disclosure & anti-opaque-bundle

- **C-Dstr-7 (structured disclosure)**: Before executing any `high`/`critical` operation, the CAP terminal SHALL present a machine-parseable disclosure containing at least `{operation_id, resource_id, mode, reversibility, physical_risk, affected_scope, terminal_attested_state}`; a free-text-only or fields-missing disclosure SHALL be rejected (`E_PROTOCOL`) and not executed.
  - **Trace**: `R5.4`.
- **C-Dstr-8 (terminal-attested state, not Fay-supplied)**: The `terminal_attested_state` in a `high`/`critical` disclosure SHALL originate from the terminal/driver, not from the requesting Fay; the CAP terminal SHALL NOT confirm a destructive operation on the basis of Fay-asserted state (anti perception-spoofing).
  - **Trace**: `R5.6`, `R12.1`.
- **C-Dstr-9 (one confirmation = one atomic operation)**: The CAP terminal SHALL bind each human confirmation to exactly one atomic operation; IF a single `execute`/confirmation would cover a bundle that includes any `high`/`critical` operation, THEN reject (`E_OPAQUE_ACTION_BUNDLE`).
  - **Trace**: `R5.5`, `R12.2`.
- **C-Dstr-10 (operation re-binding / anti-tamper)**: The CAP terminal SHALL bind the confirmation to the exact `(operation_id, resource_id, mode, parameters-digest)`; IF the executed operation differs from the confirmed one, THEN reject execution (`E_PROTOCOL`).
  - **Trace**: `R10.2`.

## 8.4 Classification source & default-deny on unknown

- **C-Dstr-11 (trusted classification source)**: The authoritative criticality of an operation SHALL come from the terminal/driver-declared **Resource Capability Descriptor**; a Fay or issuer MAY request a *stricter* treatment but SHALL NOT *lower* the declared criticality.
  - **Trace**: `R5.6`, `R12.4`.
- **C-Dstr-12 (default-deny on unknown)**: IF an operation has no declared criticality (unknown resource/operation, or undeclared capability), THEN the CAP terminal SHALL classify it `critical` and apply C-Dstr-2 (`E_CAPABILITY_UNDECLARED` MAY be returned to aid diagnosis).
  - **Trace**: `R2.4`, `R5.1`.

> **`[需人类决策 D-5]` — who owns the classification, and its trust anchor.** Options: (i) **terminal/driver capability manifest** (recommended; closest to ground truth, anti-prompt-injection), signed and registered like a `Verification_Key`; (ii) a **CAP-standard operation registry** keyed by resource type (portable but coarse); (iii) **issuer-declared in `Grant.constraints`** (flexible but Fay-influence-adjacent). This draft fixes the *trust rule* (driver-declared is authoritative; Fay/issuer may only raise; unknown ⇒ critical) but the *manifest format, signing authority, and registry governance* require human decision and cross-repo coordination with `fayger` (the runtime that loads drivers).

## 8.5 No-rollback acknowledgement & audit

- **C-Dstr-13 (no execute rollback)**: The CAP terminal SHALL NOT represent that it can undo an executed `irreversible` operation; the protocol provides no rollback for `execute`/actuation side effects (consistent with current Ch.7.8.3). The gate is therefore strictly pre-execution.
  - **Trace**: `R5.2`.
- **C-Dstr-14 (confirmation audit + Human-Prime view)**: The CAP terminal SHALL write every `high`/`critical` confirmation decision (granted / denied / timeout, with the disclosure contents and the accountability tuple) to the audit log and SHALL surface it on the Human-Prime view channel, in the `critical` priority bucket where applicable.
  - **Trace**: `R5.7`, `R12.7`.

## 8.6 Worked classification examples (informative)

| Operation | mode | reversibility | physical_risk | ⇒ criticality | Gate |
|---|---|---|---|---|---|
| Read temperature sensor | read | reversible | none | routine | grade ≥ G4 |
| Tail an application log | read | reversible | none | routine | grade ≥ G4 |
| Set screen brightness | write | reversible | none | routine | grade ≥ G4 |
| Capture a photo | execute | hard-to-reverse | none | sensitive | grade ≥ G3 |
| Send an email on the Prime's behalf | execute | hard-to-reverse | none | sensitive | grade ≥ G3 |
| Overwrite a document (backup exists) | write | hard-to-reverse | property | high | G2 + confirm `[D-6]` |
| Change router firewall rule | configure | hard-to-reverse | property | high | G2 + confirm |
| Unlock a smart door lock | execute | irreversible* | safety | critical | **G1 + ≤5s presence + per-op confirm** |
| Secure-erase a storage device | execute | irreversible | property | high→critical | per matrix (irreversible+property = high; treat as critical if data is unrecoverable) |
| Flash device firmware | configure | irreversible | property | high | (consider critical if bricking-risk) `[D-6]` |
| Drone takeoff / motor arm | execute | irreversible | safety | critical | **G1 + ≤5s presence + per-op confirm** |
| Drone flight-termination | execute | irreversible | life-critical | critical | **G1 + ≤5s presence + per-op confirm** |
| Set infusion-pump rate | configure | irreversible | life-critical | critical | **G1 + ≤5s presence + per-op confirm** |
| Execute a fund transfer | execute | irreversible | property | high→critical | **treat as critical** (matches Faying G1 examples) |

\* "Unlocking a door" is physically reversible (re-lock) but its *security consequence* (granting access) is treated as irreversible for risk purposes — a reminder that classification is about *consequence*, not just state-restorability. This nuance is a `[需人类决策 D-7]` for the criticality rubric.

> **`[需人类决策 D-7]` — the criticality rubric edge cases.** (a) Should "consequence-irreversibility" (door unlock, message sent to many recipients) be modeled as a third reversibility axis or folded into `physical_risk`? (b) Where exactly do `secure-erase` / `firmware-flash` land (high vs critical)? (c) Is `fund transfer` in CAP scope at all, or delegated to a payment protocol? These are domain-expert calls.

---

## 8.7 Summary of hard constraints (this chapter)

| ID | One-line | Trace |
|---|---|---|
| C-Dstr-1 | Classification mandatory; unknown ⇒ critical | R5.1, R2.4 |
| C-Dstr-2 | Critical gate: G1 + ≤5s presence + per-op confirm | R5.2, R12.3 |
| C-Dstr-3 | High gate: ≥G2 + presence + confirm `[D-6]` | R5.3 |
| C-Dstr-4 | Criticality is history-independent | R5.8, R12.6 |
| C-Dstr-5 | Confirmation single-use + timeout | R5.9 |
| C-Dstr-6 | Attention-drift hold | R12.5 |
| C-Dstr-7 | Structured disclosure required | R5.4 |
| C-Dstr-8 | Terminal-attested state, not Fay-supplied | R5.6, R12.1 |
| C-Dstr-9 | One confirmation = one atomic op (anti-bundle) | R5.5, R12.2 |
| C-Dstr-10 | Confirmation re-binds exact operation | R10.2 |
| C-Dstr-11 | Driver-declared classification is authoritative | R5.6, R12.4 |
| C-Dstr-12 | Default-deny on unknown classification | R2.4, R5.1 |
| C-Dstr-13 | No execute rollback; gate is pre-execution | R5.2 |
| C-Dstr-14 | Confirmation audit + Human-Prime view | R5.7, R12.7 |

## Changelog
- (draft) Initial destructive/irreversible operation confirmation model, H1-PROTO-01.
