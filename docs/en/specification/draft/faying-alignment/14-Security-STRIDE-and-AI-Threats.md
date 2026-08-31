# CAP Final §12 — Security Considerations (Normative draft)

> **Status**: Drafted clause for H1-PROTO-01, Wave 2 (**uplift of current Ch.10 from non-normative to normative; closes G-3**). This chapter makes CAP's threat model a first-class protocol-layer commitment: STRIDE baseline (§12.1), the known-risk register (§12.2), the **complete AI-era adversarial threat catalog CT1–CT8** (§12.3), deployment recommendations (§12.4), and the limitations + security-evolution schedule incl. PQC (§12.5). Modeled on `Faying-Protocol-1.0` §9 / §9.10.
> **Normative shift**: unlike current Ch.10 ("this chapter is non-normative"), §12.1 and §12.3 carry hard constraints `C-Sec-<n>` with `Trace: R<n>.<m>.<k>`. The most dangerous protocol in the system no longer has its threat model as advisory text.
> Each CT threat is anchored to a **protocol structure + error code + correctness property + requirement** (the Faying §9.10 discipline). Properties cite the `P-CAP-<n>` series in `07`. Open PO decisions are marked `[依赖 D-n]` (see `08`).

---

## §12.0 Trust model (reuse of Ch.10.1, restated normative scope)

**Trusted**: `Registration_Authority` (trust anchor), pre-installed RA anchor keys, terminal secure storage. **Not trusted**: Fay instances, `iFay_Runtime`, network channels, other terminal processes. The **bound Faying Mandate's accountability chain** is consumed as the root of accountability (§2.2). CAP protects against network attackers, rogue Fays, credential stealers, and a compromised `Descriptor_Issuer` (via key revocation); it does **not** defend a terminal under full physical control or a fully-compromised Fay process holding its own keys (§12.5).

## §12.1 STRIDE threat model (normative)

| Category | CAP threat | Protocol-level mitigation | Anchors |
|---|---|---|---|
| **S**poofing | Forged descriptor / impersonated Fay submits `AuthRequest` | Signature verification + `subject_fay_id` binding + bound Faying Mandate accountability root | §6.3, §2.2; `E_INVALID_SIGNATURE`, `E_MANDATE_INVALID`; R1.1, R10.1 |
| **T**ampering | MITM alters `AuthRequest` mode/operation to a more destructive one | TLS 1.3 + signed descriptor + **per-operation confirmation re-binds the exact operation** (§8.3 C-Dstr-10) | §8.3; `E_PROTOCOL`; R10.2 |
| **R**epudiation | Human Prime / Fay denies authorizing a destructive action | Confirmation audit record + Faying attestation chain dual-anchor | §8.5 C-Dstr-14; R5.7, R8.2 |
| **I**nfo Disclosure | Error details leak key IDs / resource topology | Error responses expose no internal detail (§11.5 C-Err-1) | §11.5; R10.3 |
| **D**oS | Session/connection/storage exhaustion by a rogue Fay | Session quotas + rate limits + heartbeat reclamation | §10.2, Ch.5.8; `E_SESSION_LIMIT_EXCEEDED`, `E_RATE_LIMIT_EXCEEDED`; R10.4 |
| **E**levation | Fay escalates mode/grade mid-session, or batches a critical op under a benign confirmation | No-escalation (§7.6 C-Grade-5) + grade floor (§7.2) + anti-opaque-bundle (§8.3 C-Dstr-9) | §7, §8; `E_GRADE_INSUFFICIENT`, `E_OPAQUE_ACTION_BUNDLE`; R4.4, R5.5 |

- **C-Sec-1 (STRIDE mitigations are normative)**: The CAP terminal SHALL implement the mitigations above; each is realized by the cited constraint in §6–§11. A conformance claim (§13) SHALL exercise the corresponding negative scenarios.
  - **Trace**: `R10.1`, `R10.2`, `R10.3`, `R10.4`.

## §12.2 Known-risk register (uplift of Ch.10.2; mitigations normative where cited)

| Risk | Mitigation (cited constraints) | Residual risk |
|---|---|---|
| **Credential leakage** | `subject_fay_id` binding (validation step); encrypted credential storage; short-lived tickets (≤7 d) + live revocation | A fully-compromised Fay process (holding its own keys) can still abuse credentials — out of scope (§12.5) |
| **Revocation delay** | Online live query; offline validity ceilings (≤90 d offline / ≤7 d ticket); revocation reachability ≥ establishment (§10.4 C-Life-6) | Long-offline terminals may honor a revoked credential until the ceiling — the offline-availability trade-off (R3) |
| **Replay** | `message_id` seen-cache (§5.2 C-Msg-4); `Heartbeat.sequence_number` monotonicity; TLS session isolation | In-session replay if TLS itself is broken — a TLS-layer threat |
| **Clock attack** | Trusted clock source; clock-drift detection; re-sync prompt after long offline | A physical attacker controlling the clock — physical security (§12.5) |
| **Issuer compromise** | Emergency `Verification_Key` revocation (§6.4); session kill on key revocation (C-Crypto-8) | Credentials issued before key revocation may be abused in the revocation window |
| **Resource exhaustion** | Session quotas (Ch.5.8); storage ceiling; heartbeat reclamation; per-`fay_id` rate limiting | Colluding legitimately-issued credentials — bounded by per-Fay quota policy |
| **Handover race** | `handover_pending` pre-occupancy (§9); deterministic idle on failure; handover timeout | Extreme post-release failure leaves the resource idle (no auto-restore) — a deliberate safety cost (R6.2) |
| **Side channel** | No internal detail in errors (§11.5); comparable timing across failure branches (§11.5 C-Err-4) | Full side-channel elimination is impractical at the protocol layer |

## §12.3 AI-era adversarial threat catalog CT1–CT8 (normative, complete)

> **Scope statement**: non-exhaustive. The Protocol Committee MAY add CT9, CT10, … under §1.6 backward compatibility, each with at least one protocol-level defense + EARS clause + anchor. **No exemption** from CT1–CT7 hard constraints is allowed under the pretext of an unenumerated threat.
>
> **Purpose**: elevate adversarial threats targeting the *terminal control-authority layer* in the AI-Agent era from business-side fallback to protocol-level hard constraint. CAP's defining hazard is that an **authorized-but-misaligned Fay drives physical hardware**; the catalog below specializes Faying's T1–T7 for that domain.

### §12.3.1 CT index table

| CT | Threat | Protocol-level defense focus | Anchor | Property | Requirement |
|---|---|---|---|---|---|
| **CT1** | Perception Spoofing of Terminal State | confirmation disclosure on terminal-attested state, not Fay-supplied state | §8.3 C-Dstr-8 | P-CAP-3 | R12.1 |
| **CT2** | Opaque Hardware-Command Bundle | one-confirmation-one-atomic-operation; reject bundled critical ops | §8.3 C-Dstr-9 | P-CAP-8 | R12.2 |
| **CT3** | Simulated-Operation Confusion (GUI-grounding / computer-use) | high-criticality GUI/`execute` ⇒ ≥G1 + fresh presence + explicit confirm, no automation exemption | §7.2, §8.2 | P-CAP-3 | R12.3 |
| **CT4** | Screen-Content / External-Knowledge Prompt Injection | Fay-originated criticality is untrusted; driver-declared prevails; unknown ⇒ critical | §8.4 C-Dstr-11/12 | P-CAP-1 | R12.4 |
| **CT5** | Presence-Replay / Attention-Drift Exploit | per-grade fresh presence + attention-drift hold for critical ops | §7.4, §8.2 C-Dstr-6 | P-CAP-2 | R12.5 |
| **CT6** | Cultivated-Trust Manipulation | criticality gate is history-independent | §8.2 C-Dstr-4 | P-CAP-3 | R12.6 |
| **CT7** | Confirmation / Audit Flooding | priority-bucketed channel + rate limit; critical bucket never dropped | §11 C-Err-3 | P-CAP-1 | R12.7 |
| **CT8** | Reserved extension slot | Committee addition process | §11, §1.6 | — | R12.8 *(reserved)* |

### §12.3.2 CT1 Perception Spoofing of Terminal State

**Threat**: a Fay or attacker feeds the terminal a falsified resource state (e.g., reports "altitude 0 m" for an airborne drone, "door already open", "tank empty") so a human confirms a destructive operation under a wrong premise.

**Protocol-level defense**:
1. **Terminal-attested state only** — the `terminal_attested_state` in a `high`/`critical` disclosure SHALL originate from the terminal/driver, not the requesting Fay (§8.3 C-Dstr-8); the CAP terminal SHALL NOT confirm a destructive operation on Fay-asserted state.
2. **Attested at the indirect driver boundary** — terminal state crosses the §2.5 hardware-driver (indirect) interface, not the Fay channel.
3. **No Window-A confirmation** — a `high`/`critical` confirmation SHALL NOT be granted while presence is within its window but liveness is in doubt; converge pessimistically (deny).

**Anchors**: §8.3 C-Dstr-8; `E_PROTOCOL` (fields-missing disclosure); **P-CAP-3**; **R12.1**.

### §12.3.3 CT2 Opaque Hardware-Command Bundle

**Threat**: multiple sub-commands are packed into one `execute` so a critical action (e.g., `motor.arm`) hides under a benign confirmation (e.g., "run pre-flight check").

**Protocol-level defense**:
1. **One confirmation = one atomic operation** — the CAP terminal SHALL bind each human confirmation to exactly one atomic operation; a single `execute`/confirmation covering a bundle that includes any `high`/`critical` operation SHALL be rejected (`E_OPAQUE_ACTION_BUNDLE`) (§8.3 C-Dstr-9).
2. **Exact re-binding** — the confirmation SHALL re-bind `(operation_id, resource_id, mode, parameters_digest)`; a divergent executed operation SHALL be rejected (§8.3 C-Dstr-10).

**Anchors**: §8.3 C-Dstr-9/10; `E_OPAQUE_ACTION_BUNDLE`; **P-CAP-8**; **R12.2**.

### §12.3.4 CT3 Simulated-Operation Confusion

**Threat**: a Fay using GUI-grounding / computer-use injects actions the user did not intend; CAP cannot observe "intent", only the operation reaching the gate. The risk is treating an automation-driven destructive op as routine because "the agent was just clicking".

**Protocol-level defense**:
1. **No automation exemption** — a `high`/`critical` operation requested via a GUI-grounding / `execute` path SHALL apply the §8.2 gate (≥G1 + fresh presence + explicit per-operation confirmation for `critical`) with no exemption for "automation convenience" (§7.2, R12.3).
2. **Criticality dominates grade** — a high grade SHALL NOT exempt a `critical` operation from confirmation (§7.3 C-Grade-4).

**`[依赖 D-6]`** — the `high`-class confirmation policy (G2 vs G1 floor; confirm-always vs G1-exempt) governs whether GUI-driven `high` ops always require confirmation; this CT's `critical`-path defense is fixed regardless.

**Anchors**: §7.2, §8.2; `E_GRADE_INSUFFICIENT`, `E_HUMAN_CONFIRMATION_REQUIRED`; **P-CAP-3**; **R12.3**.

### §12.3.5 CT4 Screen-Content / External-Knowledge Prompt Injection

**Threat**: malicious on-screen content or fetched external knowledge steers the Fay into requesting destructive operations, or into *claiming* an operation is low-criticality.

**Protocol-level defense**:
1. **Fay-originated criticality is untrusted** — the CAP terminal SHALL treat any Fay-supplied criticality claim as untrusted; the **driver-declared** classification (Resource Capability Descriptor) prevails (§8.4 C-Dstr-11).
2. **Default-deny on unknown** — an operation with no declared criticality SHALL be classified `critical` and gated accordingly (§8.4 C-Dstr-12; R2.4).
3. **Raise-only** — a Fay or issuer MAY request stricter treatment but SHALL NOT lower the declared criticality.

**`[依赖 D-5]`** — the authoritative classification source and its trust anchor (driver manifest vs CAP registry vs issuer-declared) is an open PO decision; this CT's defense assumes the *trust rule* (driver-declared prevails; unknown ⇒ critical) holds under whichever source is chosen.

**Anchors**: §8.4 C-Dstr-11/12; `E_CAPABILITY_UNDECLARED`; **P-CAP-1**; **R12.4**.

### §12.3.6 CT5 Presence-Replay / Attention-Drift Exploit

**Threat**: reuse of a stale presence proof, or slipping a critical operation into a human-inattention window.

**Protocol-level defense**:
1. **Per-grade fresh presence** — the CAP terminal SHALL require a presence proof no staler than the grade's `max_staleness_ms` (G1 ≤ 5 s), rejecting stale proofs (`E_PRESENCE_STALE`) and tightening (§7.4 C-Grade-2/3).
2. **Attention-drift hold** — a `high`/`critical` operation arriving while the Human-Prime view channel has been inactive beyond the attention-drift threshold SHALL be held pending fresh confirmation, not passed (§8.2 C-Dstr-6).
3. **Single-use confirmation + timeout** — a per-operation confirmation is valid for one operation and expires at the confirmation timeout (§8.2 C-Dstr-5).

**Anchors**: §7.4, §8.2 C-Dstr-5/6; `E_PRESENCE_STALE`, `E_HUMAN_CONFIRMATION_TIMEOUT`; **P-CAP-2**; **R12.5**.

### §12.3.7 CT6 Cultivated-Trust Manipulation

**Threat**: a long benign history is used to lower scrutiny before an off-baseline destructive operation.

**Protocol-level defense**:
1. **History-independent gate** — the CAP terminal SHALL NOT lower any requirement of the §8.2 confirmation gate based on the Fay's prior behavior, reputation, or session age (§8.2 C-Dstr-4).
2. **Off-baseline critical op → full gate** — an off-baseline `critical` operation receives the full §8 gate with no reputation discount.

**Anchors**: §8.2 C-Dstr-4; `E_HUMAN_CONFIRMATION_REQUIRED`; **P-CAP-3**; **R12.6**.

### §12.3.8 CT7 Confirmation / Audit Flooding

**Threat**: flooding the Human-Prime view channel with low-priority confirmations/audits to bury a critical one (e.g., to hide an `E_MANDATE_REVOKED` or a critical confirmation).

**Protocol-level defense**:
1. **Priority-bucketed channel** — the Human-Prime view channel SHALL bucket events into at least `critical / high / normal / low`; security-critical errors and `critical`-confirmation events default to the `critical` bucket (§11 C-Err-3).
2. **Rate limit + critical penetration** — confirmation pushes per (HP, Fay) pair are rate-limited, but `critical`-bucket events SHALL NOT be dropped under rate limiting (§11 C-Err-3).

**Anchors**: §11 C-Err-3, §8.5 C-Dstr-14; (audit/HP-view consequence columns); **P-CAP-1**; **R12.7**.

### §12.3.9 CT8 Reserved extension slot

- Enumerated as a non-exhaustive set; the Protocol Committee MAY add CT9+ via the §1.6 process.
- Each new threat SHALL attach at least one protocol-level defense (EARS clause + anchor to a protocol section).
- Within a valid credential's lifetime, no exemption from CT1–CT7 hard constraints is allowed under the pretext of an unenumerated threat.

**Anchors**: §1.6, §11; **R12.8** *(reserved — added to the EARS baseline together with the first CT9+ threat; `02` currently defines R12.1–R12.7)*.

- **C-Sec-2 (CT defenses are normative + history-independent)**: The CAP terminal SHALL implement the CT1–CT7 defenses above; none SHALL be weakened by reputation, automation convenience, or "unenumerated threat" pretext.
  - **Trace**: `R12.1`–`R12.7`, `R5.8`.

## §12.4 Deployment security recommendations (reuse of Ch.10.3)

- **Key management**: HSM/TPM/Secure Enclave for private keys; rotation ≤90 d (terminal) / ≤365 d (issuer); leakage response procedures; hardware-PIN/multi-person approval for critical signing.
- **Monitoring & auditing**: log all validation failures with alerting; monitor abnormal revocation frequency (issuer-compromise signal); tamper-evident independent audit storage; comprehensively audit `configure`-mode and all `high`/`critical` operations.
- **Terminal hardening**: OS mandatory access control (SELinux/AppArmor); sandbox `iFay_Runtime`; minimize services; patch regularly.
- **Network segmentation**: isolate `Registration_Authority`; controlled `Descriptor_Issuer` environments; mTLS for critical links.

## §12.5 Protocol limitations & security evolution (uplift of Ch.10.4–10.5)

**Explicitly not addressed by this protocol**: (1) Fay integrity (assumed guaranteed by code signing / runtime integrity measurement); (2) physical security of a fully-controlled terminal; (3) audit-log canonical format (v1 exclusion); (4) cross-terminal identity federation.

**Security-evolution schedule**:
1. **PQC** — the negotiation slot (ML-DSA-65 / TLS-1.3-PQ) is **reserved now** (§6.2 C-Crypto-4); promotion of a hybrid suite from OPTIONAL to MUST tracks NIST PQC standardization. *(This replaces the Ch.10.5 "v2+ deferral" with a concrete reservation.)*
2. Distributed revocation consensus; audit-log canonical format; deeper zero-trust integration; a formal protocol security proof (the correctness properties `P-CAP-*` in `07` are the first step).

---

## §12.6 Summary of hard constraints (this chapter)

| ID | One-line | Trace |
|---|---|---|
| C-Sec-1 | STRIDE mitigations are normative + tested | R10.1–R10.4 |
| C-Sec-2 | CT1–CT7 defenses normative + history-independent | R12.1–R12.7, R5.8 |

## Open dependencies in this chapter
- `[依赖 D-5]` §12.3.5 (CT4) — authoritative criticality classification source & trust anchor.
- `[依赖 D-6]` §12.3.4 (CT3) — `high`-class confirmation policy governs GUI-driven `high` ops.

## Changelog
- (draft) Initial normative Security chapter: STRIDE + complete CT1–CT8 catalog, H1-PROTO-01 Wave 2.
