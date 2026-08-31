# CAP Final §7 — Authorization & Control Model (Normative draft, Faying-aligned)

> **Status**: Drafted clause for H1-PROTO-01 (closes gaps G-1, G-8). This is the chapter that makes CAP a first-class Faying citizen: it defines the composite control decision, binds CAP to a live Faying Mandate, maps Faying grades G0–G4 onto terminal control authority, and lands the Emergency-Override and downgrade-monotonicity behaviors at the terminal.
> **Style**: hard constraints carry IDs `C-<area>-<n>` and `Trace: R<n>.<m>.<k>` to the requirements baseline (`02`). Mirrors `Faying-Protocol-1.0` §2.3 / §7.
> **Authority**: CAP **consumes** the Faying grade and presence semantics (Faying owns them per the authoritative-source-map); this chapter specifies only how the CAP terminal *enforces* them on terminal resources.

---

## 7.1 The composite control decision

The CAP terminal grants control on a single `AuthRequest` only when **every** predicate below holds, evaluated in order; failure at any predicate is a Default-Deny rejection with the corresponding error code (§11). This is the CAP analog of Faying's "four-element verification gate".

```
GRANT  ⟺  P1 credential-valid          (§9.1 7-step validation)
        ∧ P2 mandate-bound-and-live     (C-Bind-1)
        ∧ P3 grade-sufficient           (C-Grade-1)
        ∧ P4 presence-fresh-for-grade   (C-Grade-2)
        ∧ P5 rwlock-compatible          (§7.5 matrix)
        ∧ P6 constraints-satisfied      (§9 constraints)
        ∧ P7 operation-criticality-gate (§8 / C-Grade-4)
```

- **C-Stack-1 (every operation passes the CAP gate)**: The CAP terminal SHALL NOT permit any Fay-originated operation on a `Terminal_Resource` to bypass predicates P1–P7; any operation reaching the OS access-control layer without a passing CAP decision SHALL be treated as a rogue action and rejected.
  - **Trace**: `R2.1`, `R2.3`.

## 7.2 Mandate binding (CAP as a Faying Relying Party)

- **C-Bind-1 (bound, live Mandate required)**: WHEN evaluating P2, the CAP terminal SHALL require a verifiable binding (`mandate_ref`) to a Faying Mandate that is currently `Active` (or `Disconnected` under a valid endorsement, per Faying §7.5); IF absent, expired, or unverifiable, THEN reject with `E_MANDATE_MISSING` / `E_MANDATE_INVALID`.
  - **Trace**: `R1.2`, `R1.3`.
- **C-Bind-2 (single accountability endpoint)**: The CAP terminal SHALL confirm the bound Mandate's attestation chain resolves to exactly one Human Prime / Official_Post; IF multiple or none, THEN reject (`E_MANDATE_INVALID`) and write to audit.
  - **Trace**: `R1.1`, `R1.3`.
- **C-Bind-3 (scope convergence)**: The `granted_modes` the CAP terminal derives from a matched `Grant` SHALL be a subset of the bound Mandate's scope; the CAP terminal SHALL NOT grant any mode/operation outside the Mandate scope even if the local credential would allow it.
  - **Trace**: `R13.4`.
- **C-Bind-4 (revocation enforcement)**: WHEN the bound Mandate becomes `Revoked` (any RevokeReason), the CAP terminal SHALL terminate every Session referencing it and reject new requests bound to it (`E_MANDATE_REVOKED`).
  - **Trace**: `R4.5`, `R8.3`.

> **`[需人类决策 D-1]` — the exact binding model.** Three candidate models; this draft is written to be correct under any of them, but the data-model fields and conformance vectors depend on the choice:
>
> | Model | How `mandate_ref` binds | Pros | Cons |
> |---|---|---|---|
> | **A. Derivation** — the `Authorization_Descriptor` is *issued under* a Faying Mandate (the Mandate's scope includes terminal-resource grants; the descriptor is a projection signed within the attestation chain) | Single source of truth; revocation cascades naturally | Couples issuance tightly; offline issuance flow must carry chain |
> | **B. Reference + live check** (recommended) — descriptors stay CAP-native but each `AuthRequest` carries a `mandate_ref` the CAP terminal validates against Faying (live or cached-with-staleness) | Clean separation; reuses existing CAP issuance; offline-friendly with cached attestation | Two artifacts to keep consistent |
> | **C. Embedded grade only** — the descriptor embeds `faying_grade` + presence requirement but no live Mandate check | Simplest offline | Weakest accountability; grade can go stale; not symmetric-revocable |
>
> Recommendation: **B**, with **A** as the issuance ideal for high-criticality resources, and **C explicitly disallowed** for any `high`/`critical` operation. Requires cross-repo coordination with `faying-protocol` and `fayger` (multi-repo-coordination: a CAP `openspec/change` + linked Faying/FayGer follow-ups).

## 7.3 Mapping Faying G0–G4 → CAP control authority

CAP does not invent grades; it states, per grade, the **maximum** access modes and operation criticalities a Fay may exercise on terminal resources. This is the default mapping; a `Grant.min_grade` or a Resource Capability Descriptor MAY require a stricter grade (never looser).

| Faying grade | Presence basis (Faying App. B) | Max access modes (default) | Max operation criticality (default) | Typical CAP use |
|---|---|---|---|---|
| **G4 Autonomous** | heartbeat ≤ 30 min | `read`; `execute`/`write` only for `routine` | `routine` | Sensor reads, log tailing, idempotent low-risk automation |
| **G3 Loose** | intent-ticket at key steps | `read`, `write`, `execute` | `sensitive` | Email/calendar actions, reversible writes |
| **G2 Moderate** | ≤ 60 s before sensitive subtask | `read`, `write`, `execute`, `configure` (reversible) | `high` (with confirmation per §8) | Booking, form submission, hard-to-reverse but recoverable ops |
| **G1 Strict** | ≤ 5 s continuous | all modes incl. high-privilege `configure` | `critical` (with per-operation confirmation per §8) | Irreversible / safety / life-critical actuation |
| **G0 Emergency Override** | N/A (HP-only) | — (no Fay operations) | — | HP reclaims control; all Fay sessions suspended |

- **C-Grade-1 (grade floor)**: The CAP terminal SHALL reject an `AuthRequest` whose matched `Grant.min_grade`, or whose requested operation's required grade per the mapping above and §8, is stricter than the bound Mandate's effective grade (`E_GRADE_INSUFFICIENT`).
  - **Trace**: `R4.1`.
- **C-Grade-2 (presence freshness per grade)**: The CAP terminal SHALL require a presence proof no staler than the bound grade's `max_staleness_ms`; IF stale, THEN reject the operation (`E_PRESENCE_STALE`) and apply C-Grade-3.
  - **Trace**: `R4.2`, `R12.5`.
- **C-Grade-4 (criticality dominates grade)**: The grade mapping above is a *ceiling*, not a license; the operation-criticality gate in §8 SHALL additionally apply. A high grade SHALL NOT exempt a `critical` operation from per-operation confirmation.
  - **Trace**: `R5.2`, `R5.8`.

> **`[需人类决策 D-2]` — the default mode×grade×criticality matrix above.** The cells are a defensible starting point but are a policy choice with safety consequences (e.g., "may G2 ever perform a `critical` op with confirmation, or is `critical` strictly G1-only?"). This draft fixes **`critical` ⇒ G1-only** (see §8, R5.2) and leaves the `high`-class grade floor (G2 vs G1) as `[需人类决策 D-6]`.

## 7.4 Downgrade monotonicity at the terminal ("without presence, stricter")

Mirrors Faying §7.3. Presence loss narrows the human's view, so CAP MUST tighten, never loosen.

- **C-Grade-3 (presence-loss tightening)**: WHEN the bound Mandate's presence becomes stale for the current grade, the CAP terminal SHALL, for each active Session of that Fay: (a) immediately reject any pending `high`/`critical` operation; (b) suspend the Session if its access mode/criticality is no longer permitted at the degraded effective grade; (c) SHALL NOT raise the permitted criticality or modes under any circumstance.
  - **Trace**: `R4.2`, `R7.3`.
- **C-Grade-5 (no runtime escalation)**: The CAP terminal SHALL NOT provide any interface that elevates a live Session's effective grade or `granted_modes`; a higher grade/mode requires a new `AuthRequest` bound to a fresh Mandate.
  - **Trace**: `R4.4`.

## 7.5 Emergency Override (G0) at the terminal

G0 is orthogonal to G1–G4 (Faying §7): it is not "a stricter tier" but the channel by which the Human Prime instantly reclaims control. CAP is the terminal-side enforcement endpoint.

- **C-G0-1 (immediate reclamation)**: WHEN the CAP terminal receives a verified Faying G0 Emergency Override targeting a Fay (directly or via the Revocation Registry push), it SHALL, atomically and at the highest priority over any queued event: (a) terminate all of that Fay's active Sessions on the terminal (reason `emergency_override`); (b) revoke their OS access; (c) refuse new `AuthRequest`s from that Fay until a fresh Mandate is bootstrapped; (d) where the controlled resource supports a defined safe state, request the driver to enter it; (e) lock control to `human`.
  - **Trace**: `R4.3`.
- **C-G0-2 (HP-only, non-forgeable)**: The CAP terminal SHALL accept a G0 only when it carries a verifiable Human-Prime witness (or an authority/regulator signature per Faying RevokeReason); a G0 purportedly self-initiated by a Fay / runtime SHALL be rejected (`E_PROTOCOL`) and audited.
  - **Trace**: `R4.3`, `R10.1`.
- **C-G0-3 (no endorsement shelter)**: WHILE a G0 is in effect for a Fay, the CAP terminal SHALL NOT honor any offline-endorsement autonomy for that Fay (parity with Faying's G0 ⟂ endorsement mutual exclusion).
  - **Trace**: `R4.3`.

> **`[需人类决策 D-3]` — the safe-state obligation (C-G0-1(d)).** Whether CAP *mandates* a driver "safe state" on G0 (e.g., drone → hover/return-to-home, infusion pump → hold-at-current-rate, vehicle → controlled-stop) or merely *terminates control* is safety-critical and resource-class-specific. This draft requires "request safe state **where the resource supports a defined safe state**"; the catalog of safe states and whether absence-of-safe-state blocks G0 takeover requires human/domain review.

## 7.6 Interaction with revocation, sessions, and handover

- **C-Map-1 (Mandate-state ↔ Session-state coupling)**: The CAP terminal SHALL terminate or suspend Sessions in response to bound-Mandate transitions as follows: `Revoked` → terminate (C-Bind-4); `Suspended` (incl. G0) → terminate (C-G0-1); presence-`stale`/`Degraded` → tighten (C-Grade-3).
  - **Trace**: `R4.5`, `R7.3`.
- **C-Map-2 (handover inherits the gate)**: WHEN control is handed to a target Fay or to `human`, the CAP terminal SHALL re-evaluate P1–P7 for the target; a handover SHALL NOT transfer a grade/criticality the target's bound Mandate does not satisfy, and a Fay-to-Human handover of `high`/`critical` control SHALL apply the §8 confirmation gate.
  - **Trace**: `R6.3`, `R4.1`.

---

## 7.7 Summary of hard constraints (this chapter)

| ID | One-line | Trace |
|---|---|---|
| C-Stack-1 | No operation bypasses the CAP gate | R2.1, R2.3 |
| C-Bind-1 | Bound, live Mandate required | R1.2, R1.3 |
| C-Bind-2 | Single accountability endpoint | R1.1, R1.3 |
| C-Bind-3 | Granted modes ⊆ Mandate scope | R13.4 |
| C-Bind-4 | Mandate revoked → kill sessions | R4.5, R8.3 |
| C-Grade-1 | Grade floor enforced | R4.1 |
| C-Grade-2 | Presence freshness per grade | R4.2, R12.5 |
| C-Grade-3 | Presence-loss tightening (monotonic) | R4.2, R7.3 |
| C-Grade-4 | Criticality dominates grade | R5.2, R5.8 |
| C-Grade-5 | No runtime escalation | R4.4 |
| C-G0-1 | G0 → immediate reclamation | R4.3 |
| C-G0-2 | G0 is HP-only, non-forgeable | R4.3, R10.1 |
| C-G0-3 | No endorsement shelter under G0 | R4.3 |
| C-Map-1 | Mandate-state ↔ session-state coupling | R4.5, R7.3 |
| C-Map-2 | Handover inherits the gate | R6.3, R4.1 |

## Changelog
- (draft) Initial Faying-aligned control model, H1-PROTO-01.
