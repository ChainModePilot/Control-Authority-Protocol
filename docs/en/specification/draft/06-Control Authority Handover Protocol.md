# Chapter 6: Control Authority Handover Protocol

This chapter defines the protocol flow of `Handover_Policy` (control authority handover policy), including the semantics of three policy types, atomicity guarantees of handover, timeout rollback, and notification mechanisms. This chapter corresponds to the design intent in §2.4 of the blueprint.

## 6.1 Applicable Scenarios

Control authority handover occurs in scenarios where multiple controllers need to use the same `Terminal_Resource` sequentially. This specification defines two core handover types:

1. **Fay-to-Fay**: One Fay transfers its held Session control authority to another Fay
2. **Fay-to-Human**: A Fay returns control authority to a human user, or a human user delegates control authority to a Fay

This specification primarily describes Fay-to-Fay. Fay-to-Human handover reuses the same flow, modeling the "human user" as an entity that interacts directly with the terminal OS through a special interface.

## 6.2 Handover Initiation

### 6.2.1 Trigger Conditions

A handover is triggered by one of the following scenarios:

1. **New Fay actively requests resource**: Fay-B requests access to a resource occupied by Fay-A via AuthRequest
2. **Current Fay actively yields**: Fay-A actively requests via `SessionTransferRequest` to transfer control authority to a designated Fay-B
3. **Management trigger**: A human user or management interface requests transfer of control authority

### 6.2.2 Handover Request Message

```
SessionTransferRequest (body of ProtocolMessage) {
  required source_session_id : Session_ID                  // Session currently holding control authority
  required target_fay_id     : Fay_ID                      // Target taker
  required target_credential : CredentialContent           // Authorization credential of the target
  required handover_reason   : string
  optional handover_metadata : map<string, string>
}
```

After receiving the handover request, the terminal initiates the handover evaluation flow.

## 6.3 Handover Evaluation

The terminal evaluates whether to permit the handover following these steps:

### 6.3.1 Step 1: Source Session Validation

- Verify that the Session corresponding to `source_session_id` exists and is in `active` state
- Verify that the initiator has the right to request this handover (e.g., from the Session's iFay_Runtime or an entity with management authority)

Failure → return `E_HANDOVER_INVALID_SOURCE`

### 6.3.2 Step 2: Target Authorization Validation

- Validate `target_credential` for legitimacy (full validation per Chapter 3/4 rules)
- Validate that `target_fay_id` is authorized for the resource in the original Session's `access_mode`

Failure → return `E_HANDOVER_INVALID_TARGET`

### 6.3.3 Step 3: Policy Evaluation

The terminal applies `Handover_Policy` per §6.4 to decide whether to permit the handover.

Failure → return `E_HANDOVER_REJECTED_BY_POLICY`

### 6.3.4 Step 4: Atomic Pre-Occupancy

After policy evaluation passes, the terminal immediately switches the original Session to `handover_pending`, entering the pre-occupancy state:

- The resource does not accept other control authority requests until the handover is complete
- The original Session can still maintain heartbeat (retaining heartbeat requirements equivalent to active) but cannot initiate new resource operations

## 6.4 Handover Policy Types

`Handover_Policy` is configured at Resource_ID granularity; each resource MAY adopt a different policy. This specification defines three policy types; all terminals MUST implement at least the §6.4.1 priority rule script.

### 6.4.1 Priority Rule Script

The terminal calculates priority scores for the source Session and the target Fay according to a predefined rule script; the higher score wins control authority.

**Configuration structure**:

```
PriorityPolicyConfig {
  required policy_type : "priority_script"
  required script_id   : string                      // Script identifier pre-configured on the terminal
  optional parameters  : map<string, string>
}
```

**Evaluation flow**:

1. The terminal loads the rule script corresponding to `script_id`
2. Input: source Session info, target Fay identifier, target credential grants, current time, Resource_ID metadata
3. Output: source priority score `S_source`, target priority score `S_target`
4. Decision: `S_target > S_source` → handover permitted; otherwise rejected

The specific language of the rule script (e.g., CEL, Lua subset, JSON Logic) is chosen by the implementation, but MUST:

- Be deterministic (the same input produces the same output)
- Have no access to the network or file system
- Have execution time limited to within 100 ms

### 6.4.2 AI Model Real-Time Decision

The terminal calls a pre-integrated AI model to make a real-time decision based on the current context.

**Configuration structure**:

```
AIPolicyConfig {
  required policy_type   : "ai_model"
  required model_endpoint : string                    // Model inference endpoint accessible to the terminal
  optional context_fields : array<string>
  required decision_timeout_ms : uint32 (default 500)
}
```

**Evaluation flow**:

1. The terminal constructs a decision request, including source/target info and context specified by `context_fields`
2. Calls the AI model inference endpoint
3. The model returns a decision: `allow` / `reject` / `require_human`
4. On `require_human`, fall back to §6.4.3 human user decision

The terminal MUST:

- Set a hard timeout (default 500 ms); on timeout, treat as `reject`
- Cache recent decision results to avoid high-frequency oscillation (cache validity ≤ 5 seconds)
- Fall back to the priority rule script when the AI model is unavailable

### 6.4.3 Human User Decision

The terminal requests a decision from a human user through the user interface.

**Configuration structure**:

```
HumanPolicyConfig {
  required policy_type     : "human_decision"
  required ui_channel      : enum["system_dialog", "companion_app", "external_panel"]
  required decision_timeout_seconds : uint32 (default 30)
  required default_action  : enum["allow", "reject"]    // Default action on timeout
}
```

**Evaluation flow**:

1. The terminal presents the handover request details to the human user via `ui_channel`
2. Wait for user input: allow / reject
3. On timeout → handle per `default_action`

Human decision mode MUST:

- Clearly display in the UI: source Fay identifier, target Fay identifier, resource identifier, access mode, handover reason
- NOT display sensitive credential details (e.g., signature, key ID) in the UI
- Recommend setting the default action to `reject` (conservative security)

## 6.5 Atomicity Guarantee

Handover MUST satisfy atomicity: **at any moment, a single Resource_ID has at most one active controller**.

### 6.5.1 Atomic Sequence

The terminal MUST execute handover in the following order, with each step completed within a critical section:

```
[T0] Enter handover_pending: Source Session state switches; resource enters pre-occupancy
[T1] Terminate source Session: source_session_id state active → terminating
                               OS access control revocation triggered
[T2] Resource reclamation complete: source state terminating → terminated
                                    Resource is in "no active Session" state at this point
[T3] Create target Session: Create new Session for target Fay per §5.2 flow
                            State enters creating
[T4] Deliver OS access control: Enable resource access for the target Fay
[T5] Target Session switches to active: Handover complete
```

External observers between [T2] and [T3] observe the resource state as "no active Session" and **never** observe two Sessions active simultaneously.

### 6.5.2 Rollback on Intermediate Step Failure

If any step in [T0]–[T2] fails:

- The source Session MUST roll back to `active`
- Resource reclamation pre-occupancy is canceled
- Return `E_HANDOVER_FAILED_AT_RELEASE`

If [T3]–[T4] fails (source has terminated but target cannot be created):

- The terminal MUST immediately clean up resources (resource enters "no active Session" state)
- Notify the original iFay_Runtime (source Session has terminated) and the target iFay_Runtime (takeover failed)
- Return `E_HANDOVER_FAILED_AT_ACQUIRE`
- **Resource state**: Remain idle, with subsequent requests competing for it. **MUST NOT** automatically restore the source Session

### 6.5.3 Serialization of Concurrent Handover Requests

Multiple concurrent handover requests for the same resource MUST be serialized:

- During the resource being in `handover_pending`, new handover requests are queued or rejected
- Implementations MAY choose rejection (`E_HANDOVER_IN_PROGRESS`) or queueing (maximum queue length 8)
- Queued requests are processed in FIFO order; each request enters evaluation only after the preceding handover completes (success or failure)

## 6.6 Timeout Handling

Each handover MUST set a hard timeout. Timeout thresholds are determined by policy type:

| Policy Type | Default Timeout | Range |
|---------|---------|------|
| Priority rule script | 1 second | 0.5–2 seconds |
| AI model real-time decision | 1 second | 0.5–3 seconds |
| Human user decision | 30 seconds | 5–120 seconds |

### 6.6.1 Timeout Rollback Principle

If a handover does not complete within the timeout (i.e., does not reach [T5]), the terminal MUST:

1. Abort the handover step currently in progress
2. If in [T0]–[T2] (source not fully terminated): Roll back to the original Session active state
3. If in [T3]–[T4] (source has terminated): Handle per §6.5.2 (do not restore source; resource is set to idle)
4. Send `HandoverFailedNotification` to the relevant parties

### 6.6.2 Timeout Notification

```
HandoverFailedNotification (body of ProtocolMessage) {
  required handover_id   : uuid
  required source_session_id : Session_ID
  required target_fay_id : Fay_ID
  required reason        : enum["timeout", "policy_rejected", "release_failed", "acquire_failed"]
  optional details       : map<string, string>
}
```

## 6.7 Retry Policy

The retry policy after handover failure is determined by the initiator. This specification does not mandate a retry mechanism but defines the boundaries of retries:

- Retry interval for the same handover request SHOULD be ≥ 1 second
- The number of retries for the same (source_session, target_fay) pair SHOULD be ≤ 3
- Retries MUST use a new handover_id

The terminal MAY reject overly frequent retry requests and return `E_HANDOVER_RETRY_LIMIT`.

## 6.8 Fay-to-Human Handover

Handovers that return control authority to a human user follow the above flow, but the target party is a human user:

- The `target_fay_id` field uses the special value `"human:" + terminal_id` (representing the human user of the current terminal)
- Target authorization validation is skipped (the human user has full permissions over the resources of their own terminal by default)
- Policy evaluation typically uses §6.4.3 human user decision (confirming takeover intent)
- When creating the target Session, the Session is not bound to any Fay but is held directly by the OS user process

The reverse flow of human user proactive takeover is symmetric: replace the original Fay Session with an OS process held by the human.

## 6.9 Observability of Handover

The terminal MUST provide handover event observability to the following:

| Receiver | Event Received |
|--------|-----------|
| Source iFay_Runtime | `SessionStateChanged` (active → handover_pending → terminating → terminated) |
| Target iFay_Runtime | `AuthResult` (success response for new Session creation) or `HandoverFailedNotification` |
| Terminal audit log | Complete handover record (including handover_id, source, target, policy decision, final result) |

## 6.10 Handover Message Sequence

Complete message sequence for a successful handover:

```
[Source Runtime]      [Target Runtime]      [Terminal]
       │                     │                  │
       │── SessionTransferRequest ─────────────→│
       │                     │                  │── Evaluate policy
       │                     │                  │── source: active → handover_pending
       │←─ SessionStateChanged(handover_pending)│
       │                     │                  │── Terminate source
       │←─ SessionStateChanged(terminating)─────│
       │←─ SessionStateChanged(terminated)──────│
       │                     │                  │── Create target
       │                     │←─ AuthResult ────│
       │                     │                  │── target: creating → active
       │                     │←─ SessionStateChanged(active)
```

Message sequence for failure rollback:

```
[Source Runtime]      [Target Runtime]      [Terminal]
       │                     │                  │
       │── SessionTransferRequest ─────────────→│
       │                     │                  │── Evaluate policy
       │                     │                  │── source: active → handover_pending
       │←─ SessionStateChanged(handover_pending)│
       │                     │                  │── Timeout or failure
       │                     │                  │── source: handover_pending → active
       │←─ SessionStateChanged(active)──────────│
       │←─ HandoverFailedNotification ──────────│
       │                     │←─ HandoverFailedNotification
```
