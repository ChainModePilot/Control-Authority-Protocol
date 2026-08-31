# CAP Final §13–§15 — Conformance & Verification (Normative draft + plan)

> **Status**: Drafted clause for H1-PROTO-01, Wave 2. Three chapters: **§13 Conformance** (normative; uplift of Ch.0.5 + Ch.9.8 + a new Faying-RP addendum), **§14 Conformance & Test-Vector Plan** (spec-body framing; detailed execution in `07`), **§15 Reference-Implementation Plan** (spec-body framing; detailed execution in `07`). Together they close G-6 on the spec side. Modeled on `Faying-Protocol-1.0` §10/§EH conformance framing + the Faying `vectors/` + `conformance/` + `reference-impls/` asset pattern.
> **Wave boundary**: §14/§15 are *plans*, not authored vectors/CDDL. Per the H1-PROTO-01 boundary, the actual CDDL, vectors, and implementations are authored only after the spec is frozen and PO decisions D-1…D-11 resolved. The authoritative step-by-step plan is maintained as a separate internal follow-up plan.
> **Style**: hard constraints `C-Conf-<n>`, `Trace: R<n>.<m>.<k>` to `02`. Open PO decisions marked `[依赖 D-n]` (see `08`).

---

## §13 Conformance (normative)

### §13.1 Conformance levels (reuse of Ch.0.5)

An implementation MUST satisfy at least the **Terminal** level to claim CAP/1.0 compliance.

| Level | Applies to | Must (summary) |
|---|---|---|
| **Terminal** | `Descriptor_Validator` + `Protocol_Engine` + session manager (the CAP terminal) | full descriptor validation; Session lifecycle + Liveness; read-write-lock; reject non-passing requests with standardized codes; maintain + sync revocation; **the §7 grade gate + §8 confirmation gate** |
| **Issuer** | `Descriptor_Issuer` / `Ticket_Issuer` | issue legitimate credentials per §4 field constraints; sign per §6; maintain status + revocation; ticket→descriptor conversion |
| **Runtime** | `iFay_Runtime` | submit `AuthRequest` per §5; maintain persistent connection + heartbeats; forward `SessionStateChanged`; release Sessions on Fay termination; **relay `ConfirmationRequest` to the Human-Prime view and return `ConfirmationResponse`** |

### §13.2 Faying-RP conformance addendum (new — closes part of G-1)

A new conformance facet asserting that an implementation **correctly consumes** the Faying authorization context.

- **C-Conf-1 (Faying-RP correctness)**: an implementation claiming **Faying-RP conformance** SHALL correctly consume the bound Mandate's grade, presence freshness, and revocation status, and SHALL enforce: the grade floor (§7.2 C-Grade-1), presence-staleness tightening (§7.4 C-Grade-3), G0 emergency reclamation (§7.5 C-G0-1), Mandate-revoked session termination (§10.3 C-Life-4), and the §8 criticality gate. It SHALL NOT issue, mint, or mutate a Mandate.
  - **Trace**: `R1.1`, `R4.1`, `R4.3`, `R4.5`, `R5.2`.

> **`[依赖 D-1]`** — exactly *what* "correctly consume" requires on the wire (live `mandate_ref` resolution vs cached-with-staleness vs derivation cascade) depends on the binding model (`08` D-1). The Faying-RP conformance test dimension is specified once D-1 is resolved.

### §13.3 Conformance-claim structure (reuse of Ch.9.8)

```
ConformanceClaim {
  required protocol_version  : "CAP/MAJOR.MINOR"
  required levels            : array<ConformanceLevel> (1..3)   // terminal | issuer | runtime
  optional faying_rp         : bool                              // claims §13.2
  optional optional_features : array<string>                    // e.g. "aggregated_heartbeat", "ai_model_handover_policy"
  optional vendor_extensions : array<string>
}
```

- **C-Conf-2 (claim honesty)**: an implementation SHALL NOT claim a level it does not fully satisfy; the conformance suite (§14) is the verification instrument but SHALL NOT be relied on as the sole correctness guarantee.
  - **Trace**: `R9.3`.

## §14 Conformance & Test-Vector Plan (spec-body framing; execution in `07`)

> The vector + conformance asset tree mirrors Faying (`vectors/` + `conformance/`). **Authoring sequence, generators, validators, and `index.json` are detailed in `07` §3–§4.** This chapter fixes only the *framework* the Final spec commits to.

### §14.1 Vector categories (positive + negative)

| Category | Asserts | Gates |
|---|---|---|
| `validation/` | 7-step descriptor/ticket validation outcomes | Terminal / Issuer |
| `grade-alignment/` | `Grant.min_grade` vs effective grade; presence tightening | Terminal / Faying-RP |
| `destructive-confirm/` | criticality matrix + confirmation gate (**the safety core; build first**) | Terminal / Faying-RP |
| `handover-atomicity/` | [T0–T5] success + rollback; never two active controllers | Terminal |
| `revocation/` | Mandate/descriptor revoked → session kill; G0 override | Terminal / Faying-RP |
| `encoding-equivalence/` | CBOR ↔ JSON semantic + signature-input equality | all |

- **C-Conf-3 (negative vectors are mandatory)**: the Final conformance suite SHALL include negative vectors for every error code in §11 and every CT1–CT7 attack scenario; the `destructive-confirm/` negatives are the highest-value tests and SHALL be authored first.
  - **Trace**: `R2.1`, `R5.2`, `R12.1`–`R12.7`.

### §14.2 Conformance dimensions & cross-impl matrix

- Dimensions `{validation, grade, destructive, encoding, handover, revocation}` × levels `{terminal, issuer, runtime, Faying-RP}` → `matrix.yaml` (`07` C1).
- An `attacks/` suite replays STRIDE + CT1–CT7 negative scenarios (`07` C3).
- A `cross-impl/` harness proves vectors issued by one implementation verify in the other (`07` C4); a CI gate (H1-GOV-03) requires green vectors + conformance + cross-impl to merge (`07` C5).
- **`[依赖 D-8]`** — whether the vector index carries a numeric companion error-code column.
- **`[依赖 D-10]`** — the authoritative schema path (`schemas/cap-1.0.cddl` vs the iAccord scaffold).

## §15 Reference-Implementation Plan (spec-body framing; execution in `07`)

- **C-Conf-4 (≥2 independent implementations)**: interoperability SHALL be demonstrated by at least two independent implementations sharing the §14 vectors, with no shared protocol library between them (the Faying template: Rust + TypeScript).
  - **Trace**: `R9.1`, `R9.3`.
- Each implementation covers: descriptor validation, the grade gate (§7), the destructive-confirm gate (§8), the session FSM (§10), and error mapping (§11), validated against all `vectors/`. Steps R1/R2/R3 are detailed in `07` §5.
- **`[依赖 D-11]`** — reference-implementation languages (recommended Rust + TypeScript, FayGer/SDK-aligned).

---

## §13–§15 Summary of hard constraints

| ID | One-line | Trace |
|---|---|---|
| C-Conf-1 | Faying-RP correctness (consume, never issue, a Mandate) | R1.1, R4.1, R4.3, R4.5, R5.2 |
| C-Conf-2 | Honest conformance claims | R9.3 |
| C-Conf-3 | Negative vectors mandatory; destructive-confirm first | R2.1, R5.2, R12.1–R12.7 |
| C-Conf-4 | ≥2 independent implementations share vectors | R9.1, R9.3 |

## Open dependencies in this chapter
- `[依赖 D-1]` §13.2 — what Faying-RP "correctly consume" requires on the wire.
- `[依赖 D-8]` §14.2 — numeric companion error codes in the vector index.
- `[依赖 D-10]` §14.2 — authoritative schema path.
- `[依赖 D-11]` §15 — reference-implementation languages.

## Changelog
- (draft) Initial Conformance chapter + Test-Vector/Reference-Impl plan framing, H1-PROTO-01 Wave 2.
