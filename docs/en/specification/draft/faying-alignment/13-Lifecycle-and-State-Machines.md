# CAP Final §10 — Lifecycle & State Machines (Normative draft)

> **Status**: Drafted clause for H1-PROTO-01, Wave 2 (uplift of current Ch.5.1 Session FSM + **new Faying-Mandate coupling**). Defines the Session finite state machine and the cross-state coupling between the bound Faying Mandate's state (Active / Degraded / Suspended / Revoked) and the CAP Session lifecycle. Modeled on `Faying-Protocol-1.0` §8 (lifecycle) + §4.3 (state coupling).
> **Style**: hard constraints `C-Life-<n>`, `Trace: R<n>.<m>.<k>` to `02`. Open PO decisions are marked `[依赖 D-n]` (see `08`).

---

## §10.1 Session state machine (reuse of Ch.5.1)

```
SessionState = enum["creating","active","handover_pending","terminating","terminated"]
```

| State | Description |
|---|---|
| `creating` | Authorization validation passed; resources / OS access being set up |
| `active` | Fay may operate within the authorized scope (subject to §7/§8 gates) |
| `handover_pending` | Participating in a control-authority handover (§9); outcome undetermined |
| `terminating` | Being terminated; resources being reclaimed; new requests rejected |
| `terminated` | Fully terminated; `session_id` enters the historical record |

**Allowed transitions** (the CAP terminal SHALL NOT perform any transition not listed — Default-Deny on the state-machine layer):

| From | To | Trigger |
|---|---|---|
| `creating` | `active` | resource initialization complete |
| `creating` | `terminating` | initialization failed (e.g., resource busy) |
| `active` | `handover_pending` | handover request received for this Session |
| `active` | `terminating` | proactive release / liveness timeout / revocation / resource unavailable / **Mandate state change (§10.3)** |
| `handover_pending` | `terminating` | handover succeeded (source MUST terminate) or handover timeout |
| `handover_pending` | `active` | handover rejected by target (rollback) |
| `terminating` | `terminated` | resources released; persistence complete |

- **C-Life-1 (atomic transitions)**: each transition SHALL be performed in a critical section (no observable intermediate state), and the transition + its resource operation (OS access-control delivery) SHALL be a single transaction (all-or-rollback). *(Reuse of Ch.5.1.4.)*
  - **Trace**: `R6.1`.
- **C-Life-2 (no auto-restore)**: a `terminated` Session SHALL NOT be auto-restored; a new `AuthRequest` is required.
  - **Trace**: `R7.4`.

## §10.2 Liveness & zombie reclamation (reuse of Ch.5.4)

- **C-Life-3 (dual determination)**: the CAP terminal SHALL determine a Session failed only when **both** the persistent connection is broken beyond the heartbeat-timeout threshold **and** `now − last_heartbeat_at` exceeds it; on failure it SHALL terminate the Session, revoke OS access, and release the resource.
  - **Trace**: `R7.1`, `R7.2`.

## §10.3 Faying-Mandate state ↔ Session lifecycle coupling (new)

The bound Faying Mandate has its own state (consumed from Faying; CAP does not own it). CAP couples each Mandate transition to a Session consequence. The mapping mirrors Faying §4.3 and lands the downgrade-monotonicity principle at the terminal.

| Bound Mandate state | CAP Session consequence | Mechanism | Trace |
|---|---|---|---|
| **Active** | Sessions operate normally subject to §7/§8 gates | normal path | R4.1 |
| **Degraded** (presence stale for the grade) | **tighten**: reject pending `high`/`critical` ops; suspend Sessions whose criticality is no longer permitted at the degraded effective grade; never loosen | C-Grade-3 (`04`) | R4.2, R7.3 |
| **Suspended** (incl. G0 in effect) | **terminate** all of that Fay's Sessions on the terminal; refuse new requests; lock to `human` | C-G0-1 (`04`) | R4.3 |
| **Revoked** (any RevokeReason) | **terminate** every Session referencing the Mandate; cascade same-chain | C-Bind-4 (`04`) | R4.5, R8.3 |

- **C-Life-4 (Mandate-driven termination/tightening)**: the CAP terminal SHALL apply the consequence above for each observed bound-Mandate transition, at a priority no lower than ordinary session events; a `Revoked` or `Suspended` transition SHALL preempt queued lower-priority events.
  - **Trace**: `R4.5`, `R7.3`, `R8.3`.
- **C-Life-5 (downgrade monotonicity)**: a presence-loss / `Degraded` transition SHALL only tighten (suspend / reject), never raise the permitted criticality or granted modes of a live Session; a higher grade/mode requires a fresh `AuthRequest` under a fresh Mandate (no runtime escalation).
  - **Trace**: `R4.2`, `R4.4`.

> **`[依赖 D-1]`** — *how* the CAP terminal observes the bound Mandate's state (live query vs cached-with-staleness vs derivation cascade) depends on the binding model (`08` D-1). The coupling consequences above hold under any model; only the observation mechanism and the staleness budget for "Degraded" detection are frozen once D-1 is resolved. Do not assume a mechanism here.

## §10.4 Credential lifecycle interaction (reuse of Ch.5.7)

| Credential event | Session impact |
|---|---|
| Credential `not_after` reached | referencing Session terminates (`credential_expired`) |
| Credential revoked (statement reaches terminal) | referencing Session terminates (`credential_revoked`) |
| `Verification_Key` revoked (§6.4) | Sessions whose credentials were issued under that key terminate |
| Credential renewed | Session unaffected (still validated against the original `credential_ref`) |

- **C-Life-6 (revocation reachability ≥ establishment)**: a path that could establish a Session SHALL have an at-least-as-reachable path to revoke it; on revocation reaching the terminal, subsequent validations of the revoked credential SHALL be rejected immediately and referencing Sessions terminated.
  - **Trace**: `R8.3`, `R8.4`.

---

## §10.5 Summary of hard constraints (this chapter)

| ID | One-line | Trace |
|---|---|---|
| C-Life-1 | Atomic state transitions (single transaction) | R6.1 |
| C-Life-2 | No auto-restore of terminated Sessions | R7.4 |
| C-Life-3 | Dual-determination liveness reclamation | R7.1, R7.2 |
| C-Life-4 | Mandate-driven termination/tightening, preemptive | R4.5, R7.3, R8.3 |
| C-Life-5 | Downgrade monotonicity (tighten-only) | R4.2, R4.4 |
| C-Life-6 | Revocation reachability ≥ establishment | R8.3, R8.4 |

## Open dependencies in this chapter
- `[依赖 D-1]` §10.3 — the mechanism by which the CAP terminal observes bound-Mandate state transitions (and the staleness budget for "Degraded" detection).

## Changelog
- (draft) Initial Lifecycle & State Machines chapter incl. Mandate-state coupling, H1-PROTO-01 Wave 2.
