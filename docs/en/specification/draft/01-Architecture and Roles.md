# Chapter 1: Architecture and Roles

This chapter defines the roles in the CAP protocol, the trust relationships between them, and the interface contracts with external systems. The content of this chapter provides the shared terminology base and architectural context for subsequent chapters.

## 1.1 Protocol Roles

The CAP protocol involves the following normative roles. Each role takes on clear responsibilities in protocol interactions and interacts with other roles through restricted interfaces.

### 1.1.1 Authorizer Role

**Natural_Person** and **Official_Post** are the ultimate sources of authorization. Authorizers do not directly participate in protocol message interactions — authorizers delegate issuance authority to `Descriptor_Issuer` through an authorization delegation mechanism.

Authorizers MUST prove their identity through a secure identity verification mechanism. The specific verification mechanism for authorizer identity is outside the scope of this specification, but such a mechanism MUST satisfy the following requirements:

- Non-repudiation: Authorizers cannot repudiate the authorization decisions they have made
- Anti-forgery: Third parties cannot forge an authorizer's authorization decision

### 1.1.2 Issuer Role

**Descriptor_Issuer** is responsible for generating and issuing `Authorization_Descriptor`. **Ticket_Issuer** is responsible for issuing `Trusted_Ticket`. A single entity MAY take on both issuer roles simultaneously.

Issuers MUST:

1. Issue credentials only after receiving explicit authorization from a legitimate authorizer
2. Apply digital signatures to credentials using the signature algorithms defined in Chapter 8
3. Maintain status records of issued credentials (at minimum including issuance time, validity period, and current status)
4. Promptly update revocation status and publish revocation statements upon receiving revocation requests

Issuers MUST NOT:

1. Issue credentials without obtaining authorization
2. Issue credentials beyond the authorizer's authorization scope
3. Reuse identifiers of revoked credentials

### 1.1.3 Holder Role

**Fay** (including `iFay` and `coFay`) is the holder of credentials and the requester of resource access. Fay participates in protocol interactions indirectly through `iFay_Runtime` — Fay itself does not directly send protocol messages.

Fay MUST complete all protocol interactions with the terminal through its associated `iFay_Runtime`. Credentials held by Fay MUST be submitted to the terminal through iFay_Runtime.

### 1.1.4 Runtime Role

**iFay_Runtime** is the runtime environment for Fay instances and takes on the sending and receiving of protocol messages. A single iFay_Runtime instance MAY manage multiple Fay instances.

iFay_Runtime MUST satisfy the runtime conformance level requirements defined in Section 0.5.3.

### 1.1.5 Terminal Role

The **terminal** (Terminal) is the holder of `Terminal_Resource` and the executor of access control. The terminal integrates the following normative components:

- **Protocol_Engine**: System component executing the core logic of the CAP protocol
- **Descriptor_Validator**: Component responsible for verifying the legitimacy and validity of `Authorization_Descriptor`
- **Session manager**: Maintains the Session state machine and Liveness_Detection mechanism
- **Local revocation list**: Stores known revoked credential identifiers

The terminal MUST satisfy the terminal conformance level requirements defined in Section 0.5.1.

### 1.1.6 Trust Infrastructure Role

**Registration_Authority** is responsible for terminal registration management and the distribution of `Verification_Key`. Registration_Authority is the root anchor of the CAP protocol's trust chain.

Registration_Authority MUST:

1. Distribute Verification_Key only to terminals that have passed identity verification
2. Complete key distribution through secure channels (see Chapter 8)
3. Provide key update and rotation mechanisms
4. Maintain authoritative records of terminal registration status

## 1.2 Trust Chain

The trust chain of the CAP protocol is propagated from the authorizer to the terminal's `Descriptor_Validator`, composed of two independent but cooperating trust paths.

### 1.2.1 Authorization Trust Path

The authorization trust path determines **whether a particular Fay is authorized to access a particular resource**:

```
Authorizer (Natural_Person / Official_Post)
    │
    │ ① Authorization delegation (with non-repudiable evidence)
    ▼
Descriptor_Issuer
    │
    │ ② Issues Authorization_Descriptor (with digital signature)
    ▼
Fay
    │
    │ ③ Submits Authorization_Descriptor
    ▼
Descriptor_Validator
```

The terminal MUST verify the authenticity of each link on the authorization trust path:

- The authenticity of step ① is verified by Descriptor_Issuer at issuance time; the terminal does not directly participate
- The authenticity of step ② is verified by the terminal through verifying Descriptor_Issuer's digital signature (depending on the key trust path)
- The authenticity of step ③ is verified by the terminal through verifying the binding relationship between Authorization_Descriptor and the submitting Fay's identifier

### 1.2.2 Key Trust Path

The key trust path determines **whether the terminal trusts a particular Descriptor_Issuer's signature**:

```
Registration_Authority (trust anchor)
    │
    │ ① Distributes Verification_Key (via secure channel)
    ▼
Terminal local key store
    │
    │ ② Provides Verification_Key to Descriptor_Validator
    ▼
Descriptor_Validator
    │
    │ ③ Verifies Authorization_Descriptor signature using Verification_Key
    ▼
Validation pass / reject
```

The terminal MUST:

1. Trust only Verification_Keys distributed by Registration_Authority
2. Securely store Verification_Keys (see Chapter 8 for key storage requirements)
3. Reject signature verification using keys not distributed by Registration_Authority

## 1.3 External Interface Contracts

The CAP protocol layer interacts with 4 external systems through clearly defined interfaces. This section defines the normative contracts of these interfaces.

### 1.3.1 Interface with iFay_Runtime

The interface between iFay_Runtime and the terminal Protocol_Engine is a **bidirectional message interface**.

**Message direction: iFay_Runtime → Protocol_Engine**

| Message Type | Purpose | Required Fields |
|---------|------|---------|
| `AuthRequest` | Initiates an authorization validation request | fay_id, resource_id, access_mode, credential |
| `SessionRelease` | Proactively releases a session | session_id |
| `Heartbeat` | Application-layer heartbeat | session_id, sequence_number |
| `HandoverResponse` | Responds to a handover request | handover_id, accept |

**Message direction: Protocol_Engine → iFay_Runtime**

| Message Type | Purpose | Required Fields |
|---------|------|---------|
| `AuthResult` | Authorization validation result | request_id, status, session_id (on success), error_code (on failure) |
| `SessionStateChanged` | Session state change notification | session_id, new_state, reason |
| `HandoverRequest` | Handover request notification | handover_id, target_resource_id, deadline |
| `HeartbeatAck` | Heartbeat acknowledgment | session_id, sequence_number |

The interface implementation MUST:

1. Support persistent connections to carry `Heartbeat` and asynchronous push messages
2. Support TLS encrypted transmission
3. Use a serialization format that follows the `schema.json` definition

The interface implementation MAY:

1. Choose a specific transport protocol (WebSocket, gRPC, HTTP/2, etc.)

### 1.3.2 Interface with the Terminal Operating System

The interface between Protocol_Engine and the terminal operating system is a **local bidirectional interface**.

The operating system MUST expose the following capabilities to Protocol_Engine:

1. Resource access control delivery: Protocol_Engine issues instructions of "allow Fay X to access resource Z in mode Y"
2. Resource state query: Protocol_Engine queries the current availability and occupancy state of resources
3. Resource event subscription: Protocol_Engine subscribes to resource state change events (e.g., hardware disconnect, software crash)

The specific implementation of the interface depends on the operating system platform (Unix Domain Socket, Named Pipe, D-Bus, etc.). This specification does not prescribe the specific implementation, but MUST satisfy:

1. Access control instructions are executed synchronously and return execution results
2. Resource events are notified via asynchronous push
3. The interface is protected by the operating system's native security model, callable only by authorized Protocol_Engine processes

### 1.3.3 Interface with Hardware Drivers (Indirect)

Protocol_Engine **MUST NOT** communicate directly with hardware drivers. All hardware control is forwarded through the operating system.

Status reporting path from hardware drivers to Protocol_Engine:

```
Hardware driver → Operating system → Protocol_Engine
```

Hardware drivers SHOULD implement hardware-level access locking mechanisms to ensure that even if operating-system-level access controls are bypassed, the hardware itself can still reject unauthorized access.

### 1.3.4 Interface with Registration_Authority

The interface between Registration_Authority and the terminal is a **unidirectional interface** (RA → terminal).

Registration_Authority MUST push the following messages to the terminal:

| Message Type | Purpose | Required Fields |
|---------|------|---------|
| `VerificationKeyDistribution` | Distributes a new verification key | key_id, key_material, valid_from, issuer_id |
| `VerificationKeyRevocation` | Revokes a verification key | key_id, revocation_time, reason |
| `RegistrationStatusUpdate` | Updates terminal registration status | terminal_id, new_status |

The terminal MUST:

1. Receive messages over a TLS/mTLS secure channel
2. Verify that the signature source of the message is a trusted Registration_Authority
3. Return a confirmation to Registration_Authority after receiving a new key

The terminal does not proactively initiate protocol messages to Registration_Authority (the terminal's initial registration flow does not belong to the normal runtime interactions of the CAP protocol and is outside the scope of this specification).

## 1.4 Role and Conformance Level Mapping

| Implemented Role | Required Conformance Level |
|-----------|---------------------|
| Terminal (including Protocol_Engine and Descriptor_Validator) | Terminal Conformance Level (§0.5.1) |
| Descriptor_Issuer or Ticket_Issuer | Issuer Conformance Level (§0.5.2) |
| iFay_Runtime | Runtime Conformance Level (§0.5.3) |
| Registration_Authority | Outside the conformance scope of this specification (registration flow exceeds CAP protocol runtime) |
