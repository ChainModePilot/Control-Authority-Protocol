# Chapter 7: Resource Access Mode

This chapter defines the semantics of `Resource_Access_Mode`, the read-write lock matrix, the application of constraint conditions, and its relationship with Session permissions. This chapter corresponds to the design intent in §2.5 of the blueprint.

## 7.1 Mode Definitions

The CAP protocol defines 4 access modes:

```
AccessMode = enum["read", "write", "execute", "configure"]
```

### 7.1.1 read

**Semantics**: Read-only access to Terminal_Resource without modifying resource state.

Typical operations:

- Reading sensor real-time data (temperature, humidity, location)
- Viewing media files (photos, videos)
- Querying device state attributes (battery level, connection state)

**Concurrency**: Shareable. Multiple Sessions MAY hold read mode access to the same resource simultaneously.

### 7.1.2 write

**Semantics**: Data writing or state modification to Terminal_Resource.

Typical operations:

- Writing files to storage devices
- Modifying device operating parameters (volume, brightness, temperature setting)
- Sending content input to application windows

**Concurrency**: Exclusive. At any moment, a single Session at most holds write mode for the same resource.

### 7.1.3 execute

**Semantics**: Triggering operation instructions on Terminal_Resource.

Typical operations:

- Launching application programs
- Triggering hardware operation instructions (taking photos, takeoff, printing)
- Calling RPC interfaces

**Concurrency**: Exclusive.

The distinction between `write` and `execute`: write modifies data state; execute triggers actions (even if both ultimately change state). The terminal's access control granularity may not strictly distinguish between the two; in such cases, implementations MAY treat execute as a subclass of write (holding write permission automatically grants execute capability). However, the default semantics of this specification is that the two are independent — holding execute permission does **NOT** automatically grant write permission, and vice versa.

### 7.1.4 configure

**Semantics**: System-level configuration changes to Terminal_Resource.

Typical operations:

- Modifying device hardware parameters (camera resolution, network SSID)
- Adjusting security policies (firewall rules, pairing settings)
- Changing resource access policies (e.g., `Handover_Policy` configuration)

**Concurrency**: Exclusive and high-privilege. configure typically requires higher-level authorization to exercise (see §7.4.3).

## 7.2 Read-Write Lock Matrix

The read-write lock matrix defines the compatibility of different modes for the same resource. The compatibility of **new requests with existing occupancy** is as follows:

| Existing Occupancy \ New Request | read | write | execute | configure |
|------------------|:----:|:-----:|:-------:|:---------:|
| None             |  ✓   |   ✓   |    ✓    |     ✓     |
| read (≥1)        |  ✓   |   ✗   |    ✗    |     ✗     |
| write            |  ✗   |   ✗   |    ✗    |     ✗     |
| execute          |  ✗   |   ✗   |    ✗    |     ✗     |
| configure        |  ✗   |   ✗   |    ✗    |     ✗     |

`✓`: Compatible; the new request is allowed to immediately create a Session.
`✗`: Incompatible; the new request is rejected (return `E_RESOURCE_BUSY`).

### 7.2.1 Matrix Semantic Notes

- **read mutual compatibility**: Multiple read-only Sessions can coexist concurrently
- **write/execute/configure pairwise incompatibility**: When any exclusive mode occupies the resource, other exclusive mode requests are rejected
- **read and exclusive modes are mutually exclusive**: When read occupies, exclusive requests are forbidden; when an exclusive mode occupies, read requests are forbidden

### 7.2.2 Terminal Handling

When the terminal handles AuthRequest, it MUST:

1. Check compatibility per the matrix after validation passes and before the Session enters `creating`
2. Return `E_RESOURCE_BUSY` directly when incompatible; **do NOT** automatically queue and wait
3. iFay_Runtime may choose to retry or initiate a handover request (see Chapter 6)

## 7.3 Relationship with Session Permissions

### 7.3.1 Permission List

Each Session carries a `granted_modes` list at creation (see §2.5). The list contents come from `Grant.modes` matched in `Authorization_Descriptor.grants` or `Trusted_Ticket.grants`.

- When Fay operates on the resource through this Session, it can only use modes contained in `granted_modes`
- The `access_mode` specified by iFay_Runtime in the AuthRequest MUST be a subset of the Grant authorization list

### 7.3.2 Least Privilege Principle

iFay_Runtime SHOULD only request the minimum privilege mode needed for the current task in AuthRequest. For example, when only data reading is needed, request `read` rather than `read` + `write`.

The issuer SHOULD authorize per the least privilege principle within Authorization_Descriptor, avoiding unnecessary privilege relaxation.

### 7.3.3 No Hierarchical Inclusion Between Modes

In CAP v1, access modes are mutually independent, with **no** hierarchical inclusion relationship:

- Holding `configure` MUST NOT automatically grant `read` / `write` / `execute` capabilities
- Holding `write` MUST NOT automatically grant `read` capability
- Each access mode MUST be independently authorized

Implementations MUST NOT automatically expand the authorization scope.

### 7.3.4 Permissions Cannot Be Elevated

After a Session is created, its `granted_modes` MUST NOT be elevated within the lifecycle. If Fay needs higher-privilege modes:

1. iFay_Runtime proactively releases the current Session
2. Initiates a new AuthRequest using a credential that has the required modes
3. The terminal creates a new Session per the complete flow

Implementations MUST NOT provide any "permission upgrade" runtime interface.

## 7.4 Constraint Application

`Grant.constraints` are additional restriction conditions applied by the issuer to authorization. This specification defines the following standard constraint keys, which implementations MUST support. MAY support custom constraints (the semantics of custom constraints are negotiated between issuers and terminals, but MUST be declared in the schema documentation).

### 7.4.1 Time Window Constraint

```
constraints["time_window"] = "08:00-22:00"           // Valid daily 8:00-22:00
constraints["time_window_tz"] = "Asia/Shanghai"     // Time zone, default UTC
```

Validation timing: At Step 6 of §3.3.2, check whether each Grant satisfies constraints. If the current time is not within the window → Grant does not match.

### 7.4.2 Frequency Limit Constraint

```
constraints["max_calls_per_hour"] = "60"
constraints["max_calls_per_day"]  = "1000"
```

The terminal SHOULD maintain call counts for each (descriptor_id, fay_id, resource_id) combination. When the limit is exceeded → Grant does not match (return `E_RATE_LIMIT_EXCEEDED`).

### 7.4.3 High-Sensitivity Mode Constraint

```
constraints["require_human_confirmation"] = "true"
```

When `access_mode == "configure"` or other scenarios satisfying this constraint, the terminal MUST request a one-time confirmation from the human user before Session creation. The confirmation UI shares the same channel with §6.4.3.

### 7.4.4 Geofence Constraint

```
constraints["geo_fence"] = "geohash:wx4g0bm"        // Valid only within the specified geographic range
constraints["geo_fence_radius_m"] = "1000"
```

The terminal's location source SHOULD use calibrated GPS or network positioning. When location is unavailable → Grant does not match (conservative rejection).

### 7.4.5 Handling of Unrecognized Constraints

When the terminal encounters unrecognized constraints keys, it MUST:

- Default to rejecting the Grant (conservative principle)
- Return `E_UNSUPPORTED_CONSTRAINT` and include the unrecognized key name

When using custom constraints, issuers SHOULD inform the terminal of the constraint semantics through other channels to avoid authorization failure.

## 7.5 Mapping of Modes to OS Access Control

The terminal enforces access_mode execution through the operating system's access control interface. This section defines the recommended mapping from modes to OS controls. Specific implementations may differ across platforms but MUST maintain semantic equivalence.

| Resource Type | read | write | execute | configure |
|---------|------|-------|---------|-----------|
| Files/directories | File system read permission | File system write permission | Execute permission | Metadata modification permission |
| Device files | Device read | Device write | ioctl/control interface | Device configuration registers |
| Application processes | State query | Input injection | Launch/call | Modify launch parameters |
| Network resources | Receive data | Send data | Establish connection | Modify routing/firewall |
| Camera | Read frames | Not applicable | Capture/recording control | Adjust resolution/frame rate |

Platform-specific access control mechanisms (e.g., SELinux, AppArmor, macOS Sandbox, Android Permission) are chosen and integrated by the terminal implementation.

## 7.6 Mode Change Constraints

The same (Session, Resource) MUST maintain its initial access_mode unchanged within the lifecycle. Mode changes are realized through new Sessions:

1. Release the current Session
2. Specify the new access_mode when creating a new Session
3. The terminal re-evaluates resource occupancy compatibility per the matrix

The terminal MUST NOT provide any "mode switching" runtime interface.

## 7.7 Observability of Resource Access

The terminal SHOULD record key information for each resource access (for audit purposes):

- session_id
- fay_id
- resource_id
- access_mode
- Operation type (e.g., read size, write byte count)
- Timestamp

The specific audit log format is outside the scope of this specification (see blueprint §3.2 exclusion item "Audit log standardized format").

## 7.8 Boundaries of Mode Semantics

### 7.8.1 read mode does not prevent the resource from being modified concurrently

read mode only authorizes "read" permission; it does not guarantee that the resource will not be modified by other controllers during the read process:

- Multiple read Sessions do not interfere with each other, but if the resource is in "no active Session" state and is directly operated by a human user (bypassing the CAP protocol), read Sessions may still read changing data
- read mode does not provide transactional consistency guarantees

### 7.8.2 Atomicity of write mode

write mode only provides the protocol-level guarantee of "at most one write Session at any moment". The atomicity of specific write operations (e.g., partial write failure) depends on the underlying semantics of the resource and is outside the scope of the CAP protocol.

### 7.8.3 Side Effects of execute mode

The operations triggered by execute mode MAY change the resource state. Side effects triggered by Fay through execute are the responsibility of that Fay. The CAP protocol does not provide rollback capability for execute operations.

## 7.9 Complete Decision Flow

The terminal's complete decision flow for handling resource access:

```
[AuthRequest arrives]
        │
        ▼
  ┌─────────────────────┐
  │ §3 / §4 credential validation │
  └──────┬──────────────┘
         │ Pass
         ▼
  ┌─────────────────────┐
  │ §7.4 constraint evaluation │
  │ (constraints)       │
  └──────┬──────────────┘
         │ Pass
         ▼
  ┌─────────────────────┐
  │ §7.2 read-write lock matrix check │
  └──────┬──────────────┘
         │ Compatible
         ▼
  ┌─────────────────────┐
  │ §5.2 Create Session │
  └──────┬──────────────┘
         │
         ▼
  ┌─────────────────────┐
  │ §7.5 OS access control │
  │     delivery        │
  └──────┬──────────────┘
         │
         ▼
  [Return AuthResult granted]
```

Failure at any step returns the corresponding error code and does not enter subsequent steps.
