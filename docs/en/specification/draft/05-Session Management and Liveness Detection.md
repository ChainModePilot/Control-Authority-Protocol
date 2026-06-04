# Chapter 5: Session Management and Liveness Detection

This chapter defines the state machine, lifecycle events, and binding rules of `Session` with `Terminal_Resource`, as well as the `Liveness_Detection` mechanism. This chapter corresponds to the design intent in §2.3 of the blueprint.

## 5.1 Session State Machine

Each Session passes through multiple states of a finite state machine within its lifecycle.

### 5.1.1 State Definitions

```
SessionState = enum[
  "creating",
  "active",
  "handover_pending",
  "terminating",
  "terminated"
]
```

| State | Description |
|------|------|
| `creating` | Authorization validation passed; Session is initializing (e.g., allocating resources, setting up OS access control) |
| `active` | Session is in active state; Fay can perform operations within the authorized scope |
| `handover_pending` | Session is participating in a control authority handover (see Chapter 6); the result is undetermined |
| `terminating` | Session is being terminated; resources are being reclaimed; new requests are rejected |
| `terminated` | Session is fully terminated; session_id enters the historical record |

### 5.1.2 State Transition Diagram

```
              ┌─────────────┐
              │   (start)   │
              └──────┬──────┘
                     │ Authorization validation passed
                     ▼
              ┌─────────────┐
              │  creating   │
              └──────┬──────┘
                     │ Resource initialization complete
                     ▼
              ┌─────────────┐  Handover initiated  ┌──────────────────┐
              │   active    │─────────────────────→│ handover_pending │
              └──────┬──────┘                      └──────┬───────────┘
                     │  ↑                                 │
              Terminate│  │Handover timeout rollback     │Handover complete
                     ▼  │                                 ▼
              ┌─────────────┐                       ┌─────────────┐
              │ terminating │←──────────────────────┤ (handed off)│
              └──────┬──────┘                       └─────────────┘
                     │ Resource reclamation complete
                     ▼
              ┌─────────────┐
              │ terminated  │
              └─────────────┘
```

### 5.1.3 State Transition Rules

The terminal MUST update Session state only via the transitions allowed in the table below:

| Current State | Target State | Trigger Condition |
|---------|---------|---------|
| `creating` | `active` | Resource initialization complete |
| `creating` | `terminating` | Initialization failed (e.g., resource occupied by another Session) |
| `active` | `handover_pending` | Received a handover request for this Session |
| `active` | `terminating` | Proactive release, heartbeat timeout, revocation termination, resource unavailable |
| `handover_pending` | `terminating` | Handover succeeded (the original Session MUST terminate) or handover timeout cancellation |
| `handover_pending` | `active` | Handover rejected by the new controller (rolled back to original state) |
| `terminating` | `terminated` | Resources fully released; state persistence complete |

Implementations MUST NOT perform state transitions not listed.

### 5.1.4 Atomicity of State Transitions

Each state transition MUST satisfy the following atomicity requirements:

1. State read, judgment, and write MUST be completed within a critical section; external observers do not see intermediate states
2. State transitions and the corresponding resource operations (e.g., OS access control delivery) MUST be executed as a single transaction; either all succeed or all roll back

## 5.2 Session Creation

### 5.2.1 Creation Trigger

Session is created in the following scenarios:

1. iFay_Runtime initiates an AuthRequest and authorization validation passes (see §3.3, §4.3)
2. After successful control authority handover, a new Session is created for the new controller (see Chapter 6)

### 5.2.2 Creation Steps

The terminal MUST create a Session following these steps:

1. **Resource occupancy pre-check**: Check whether the target `resource_id` can be occupied under `access_mode` (see §5.3 and the read-write lock rules in Chapter 7)
2. **Generate Session_ID**: Allocate a new UUID v7
3. **Set initial state**: `state = "creating"`
4. **Construct Session object**: Fill in the fields defined in §2.5
5. **Deliver OS access control**: Inform the operating system through the §1.3.2 interface to allow the Fay process to access the resource in the specified mode
6. **State transition**: `state: creating → active`; record `created_at` and `last_heartbeat_at`
7. **Return AuthResult**: Return `session_id` to iFay_Runtime via AuthResult

### 5.2.3 Creation Failure Handling

If any step fails:

1. Allocated resources (e.g., OS access control entries) MUST be rolled back
2. Session state transitions to `terminating` and then immediately to `terminated`
3. Return the corresponding error code (e.g., `E_RESOURCE_BUSY`, `E_OS_INTEGRATION_FAILED`)

## 5.3 Binding of Session to Terminal_Resource

### 5.3.1 One-to-One Binding

Each active Session MUST be bound to exactly one `Resource_ID`. The terminal maintains the resource occupancy relationship through (resource_id, access_mode).

### 5.3.2 Exclusivity and Sharing Rules

See §7.2 read-write lock matrix in Chapter 7 for details. This section gives the simplified rules at the session level:

| Resource Current Active Session | New Session Request | Handling |
|---------------------|----------------|------|
| None | Any mode | Creation allowed |
| ≥1 read | read | Creation allowed |
| ≥1 read | write/execute/configure | Reject (`E_RESOURCE_BUSY`) |
| 1 write/execute/configure | Any mode | Reject (`E_RESOURCE_BUSY`) |

### 5.3.3 Cascading Termination on Resource Unavailability

When the resource corresponding to `Resource_ID` becomes unavailable (e.g., hardware disconnect, software process crash, operating system reports resource disappearance):

1. The terminal MUST immediately switch all Sessions bound to that resource to `terminating`
2. The terminal MUST push `SessionStateChanged` notifications to all affected iFay_Runtimes, with `reason` as `"resource_unavailable"`
3. After the resource recovers, terminated Sessions MUST NOT be automatically restored; iFay_Runtime needs to re-initiate AuthRequest to create a new Session

## 5.4 Liveness_Detection

Liveness detection determines whether a Session is still active through two independent signals: **persistent connection maintenance** and **application-layer heartbeat**.

### 5.4.1 Persistent Connection Maintenance

Between iFay_Runtime and the terminal, MUST maintain a persistent connection channel (e.g., WebSocket, HTTP/2 stream, gRPC stream).

The terminal MUST monitor the state of this persistent connection:

- Connection established: Persistent connection signal valid
- Connection broken: Persistent connection signal invalid (including TCP RST, timeout without response, etc.)

A persistent connection break MAY be the result of temporary network jitter, so MUST NOT be used alone as the basis for terminating a Session (see §5.4.3 dual determination).

### 5.4.2 Application-Layer Heartbeat

iFay_Runtime MUST periodically send Heartbeat messages over the persistent connection for each active Session:

```
Heartbeat (body of ProtocolMessage) {
  required session_id      : Session_ID
  required sequence_number : uint64
}

HeartbeatAck (body of ProtocolMessage) {
  required session_id      : Session_ID
  required sequence_number : uint64
}
```

| Parameter | Default | Range |
|------|--------|------|
| Heartbeat interval | 10 seconds | 1–60 seconds |
| Heartbeat timeout threshold | 30 seconds | Heartbeat interval × 2 to heartbeat interval × 6 |

`sequence_number` is monotonically increasing within each Session, starting from 0.

The terminal MUST:

1. Immediately return HeartbeatAck after receiving Heartbeat (`sequence_number` matches the request)
2. Update the Session's `last_heartbeat_at`
3. Reject Heartbeats with `sequence_number` less than the recorded maximum (replay prevention)

### 5.4.3 Dual Determination

The terminal MUST determine Session failure only when the following two conditions are met **simultaneously**:

1. The persistent connection has been broken for longer than the heartbeat timeout threshold
2. The duration since `last_heartbeat_at` exceeds the heartbeat timeout threshold

The design intent of this dual determination is:

- During short-term jitter of the persistent connection, application-layer heartbeats may still be normal (should not be misjudged as failed)
- When application-layer heartbeats stop but the persistent connection remains (e.g., Fay stuck), the heartbeat timeout is needed for detection
- Both conditions occurring simultaneously is a high-confidence failure signal

### 5.4.4 Handling After Failure

After determining failure, the terminal MUST:

1. Switch Session state: `active → terminating`
2. Release the occupied Terminal_Resource
3. Revoke the Fay's access to the resource through the OS interface
4. While waiting for persistent connection or heartbeat recovery, all requests for that Session return `E_SESSION_TERMINATED`
5. The terminal MAY push one `SessionStateChanged` notification to iFay_Runtime after persistent connection recovery, but the Session is no longer recoverable at this point

### 5.4.5 Optional Heartbeat Modes

Implementations MAY support the following heartbeat optimization modes (with explicit support declaration required):

1. **Aggregated heartbeat**: iFay_Runtime reports multiple Sessions in a single Heartbeat message (applicable to scenarios where a runtime manages a large number of Sessions)
2. **Reduced-frequency heartbeat**: When a Session has not initiated resource access for a long time, the heartbeat interval can be expanded up to 60 seconds

The specific format of aggregated heartbeats, as an optional extension, is defined as optional fields in schema.json.

## 5.5 Session Termination

### 5.5.1 Termination Trigger Conditions

| Trigger Reason | reason Field Value | Trigger |
|---------|---------------|--------|
| Proactive release | `"client_release"` | iFay_Runtime |
| Heartbeat timeout | `"liveness_timeout"` | Terminal Liveness_Detection |
| Revocation termination | `"credential_revoked"` | Terminal revocation list update |
| Resource unavailable | `"resource_unavailable"` | Terminal OS integration layer |
| Credential expired | `"credential_expired"` | Terminal periodic check |
| Handover complete | `"handed_over"` | Terminal handover flow |
| Forced termination | `"forced"` | Terminal management interface (e.g., admin operations) |

### 5.5.2 Termination Steps

The terminal MUST execute termination following these steps:

1. **State transition**: `{active|handover_pending} → terminating`
2. **OS resource reclamation**: Revoke the Fay process's resource access through the §1.3.2 interface
3. **Release concurrency control**: Remove (resource_id, access_mode) from the occupancy list (see Chapter 7 read-write lock update)
4. **Notify iFay_Runtime**: Send a `SessionStateChanged` message with state `terminated`, including reason
5. **State transition**: `terminating → terminated`
6. **Persistence**: MAY write the Session's historical record to the audit log

### 5.5.3 Idempotency of Termination

The SessionRelease message initiated by iFay_Runtime MUST be idempotent:

- If the Session is already in `terminating` or `terminated`, repeated release MUST return success (not treated as an error)
- If the Session does not exist (already reclaimed), return `E_SESSION_NOT_FOUND`

### 5.5.4 Observability of Termination

The terminal MUST notify Session termination to at least the following:

1. The iFay_Runtime associated with the Session (via `SessionStateChanged`)
2. The terminal's local resource management subsystem (for updating availability state)

The terminal MAY choose whether to notify:

- Other iFay_Runtimes waiting for that resource (via the §6 handover notification mechanism)
- Audit log (depending on the audit policy of the terminal implementation)

## 5.6 State Change Notification

```
SessionStateChanged (body of ProtocolMessage) {
  required session_id : Session_ID
  required new_state  : SessionState
  optional reason     : string
  optional details    : map<string, string>
}
```

The terminal MUST send SessionStateChanged to iFay_Runtime when the following state transitions occur:

1. `creating → active`: Session creation succeeded (may also be communicated directly via AuthResult; either)
2. `active → handover_pending`: Session enters the handover flow
3. `handover_pending → active`: Handover canceled or rejected
4. `* → terminating` and `terminating → terminated`: Beginning and end of the termination process

## 5.7 Relationship Between Session and Credential Lifecycle

Session lifecycle is independent of credential lifecycle, but the terminal MUST monitor credential state changes:

| Credential Event | Session Impact |
|---------|-------------|
| Credential `not_after` reached | Associated Session immediately terminates with `credential_expired` |
| Credential revoked (revocation statement reaches terminal) | Associated Session immediately terminates with `credential_revoked` |
| Credential re-issued with same ID (theoretically impossible, descriptor_id is non-reusable) | Not applicable |
| Credential overwritten by new version (renewal flow) | Session not affected (still validated using the original credential_ref) |

## 5.8 Session Quantity Limits

The terminal SHOULD implement Session quantity limits to prevent resource exhaustion:

| Dimension | Default Limit | Notes |
|------|---------|------|
| Total active Sessions on a single terminal | 1024 | Adjustable by terminal policy |
| Active Sessions for a single Fay | 64 | Prevents a single Fay from monopolizing system resources |
| Active read Sessions for a single Resource_ID | 32 | Prevents abuse of read sharing |

When a limit is exceeded, the terminal MUST return `E_SESSION_LIMIT_EXCEEDED`.
