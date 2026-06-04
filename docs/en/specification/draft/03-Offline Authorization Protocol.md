# Chapter 3: Offline Authorization Protocol

This chapter defines the complete lifecycle protocol flow of `Authorization_Descriptor`, including the five stages of issuance, local storage, validation, revocation, and renewal. The flow in this chapter corresponds to the design intent in §2.1 of the blueprint.

## 3.1 Issuance Flow

The issuance flow is executed by Descriptor_Issuer after receiving explicit authorization from the authorizer. This specification defines the format constraints of the issuance result but does not prescribe the specific interaction between the authorizer and Descriptor_Issuer (different deployments may adopt different forms such as web forms, mobile App authorization, enterprise authorization management systems, etc.).

### 3.1.1 Issuance Input

At the time of issuance, Descriptor_Issuer MUST have already confirmed the following information:

1. The authorizer's identity (`grantor_id`) has passed Descriptor_Issuer's own identity verification mechanism
2. The authorization scope (target `subject_fay_id`, `terminal_id`, `grants`) is explicitly specified by the authorizer
3. The validity period (`not_before`, `not_after`) is explicitly specified by the authorizer or follows a default policy

### 3.1.2 Issuance Steps

Descriptor_Issuer MUST generate Authorization_Descriptor following these steps:

1. **Generate identifier**: Allocate a new `descriptor_id` (UUID v7); MUST NOT reuse a previously used ID
2. **Construct payload**: Fill in `DescriptorPayload` per §2.3.2; all required fields MUST be set
3. **Validate constraints**: Locally validate constraints such as `not_after - not_before ≤ 90 days` (see §2.3.2)
4. **CBOR serialization**: Serialize the payload using RFC 8949 deterministic encoding
5. **Digital signature**: Sign the payload bytes using the Descriptor_Issuer private key with the algorithm defined in §8
6. **Assemble structure**: Construct the complete `AuthorizationDescriptor`, including version, payload, and signature
7. **Register record**: Register the credential in Descriptor_Issuer's internal state store (for subsequent revocation management)

### 3.1.3 Post-Issuance Delivery

After issuance is complete, Descriptor_Issuer delivers Authorization_Descriptor to the target Fay. The delivery method is outside the scope of this specification, but SHOULD satisfy:

1. Delivery via an encrypted channel to avoid interception of the credential during transmission
2. Delivery to the `iFay_Runtime` to which the Fay belongs, with iFay_Runtime holding it on behalf of the Fay

## 3.2 Local Storage Flow

Fay submits Authorization_Descriptor to the target terminal through iFay_Runtime for local storage.

### 3.2.1 Submission Message

iFay_Runtime sends a `DescriptorSubmit` message to Protocol_Engine:

```
DescriptorSubmit (body of ProtocolMessage) {
  required descriptor : AuthorizationDescriptor
}
```

### 3.2.2 Terminal Processing

After receiving `DescriptorSubmit`, the terminal MUST process it in the following order:

1. **Structural validation**: Verify that `descriptor` conforms to the structure defined in §2.3
2. **Signature validation**: Verify the signature using the Verification_Key corresponding to `key_id` (see §3.3.4)
3. **Deduplication check**: If a credential with the same `descriptor_id` is already stored locally and the newly submitted content matches the stored content byte-for-byte, return success; otherwise reject with a duplicate ID error
4. **Storage**: Encrypt and store the Authorization_Descriptor in the local secure storage area (see §3.2.3)
5. **Return result**: Return a `DescriptorSubmitResult` message to iFay_Runtime, containing success or an error code

### 3.2.3 Storage Requirements

The terminal MUST:

1. Store Authorization_Descriptor encrypted; the encryption key MUST NOT be readable by unauthorized processes
2. The storage medium SHOULD have anti-physical-tampering capability (e.g., secure chip, TEE)
3. The storage capacity ceiling SHOULD be no fewer than 1024 credentials; when the ceiling is exceeded, expired credentials are evicted using LRU policy

The terminal MUST NOT:

1. Store Authorization_Descriptor in plaintext form
2. Modify any field of the descriptor before storage (including metadata)

### 3.2.4 Error Codes

| Error Code | Trigger Condition |
|-------|---------|
| `E_INVALID_STRUCTURE` | Structure does not conform to §2.3 |
| `E_INVALID_SIGNATURE` | Signature validation failed |
| `E_UNKNOWN_ISSUER` | The Verification_Key corresponding to `key_id` is not registered on the terminal |
| `E_DUPLICATE_DESCRIPTOR_ID` | Conflict with stored descriptor_id with mismatching content |
| `E_STORAGE_FULL` | Storage capacity is full and no eviction is possible |
| `E_VALIDITY_OUT_OF_RANGE` | `not_after - not_before` exceeds the limit |

## 3.3 Validation Flow

The validation flow is executed on each resource access request. This section defines the complete steps and decision rules for validation.

### 3.3.1 Validation Trigger

When iFay_Runtime sends an `AuthRequest` requesting resource access, the terminal Protocol_Engine triggers the validation flow:

```
AuthRequest (body of ProtocolMessage) {
  required fay_id        : Fay_ID
  required resource_id   : Resource_ID
  required access_mode   : AccessMode
  required credential    : CredentialReference
}

CredentialReference {
  required type : enum["descriptor", "ticket"]
  required id   : string                      // descriptor_id or jti
}
```

Note: In the offline authorization scenario of this specification, `credential` references an Authorization_Descriptor already stored on the terminal; there is no need to transmit the complete credential in the request.

### 3.3.2 Validation Steps (Executed in Order)

The terminal MUST execute validation in the following order. Failure at any step immediately returns the corresponding error code without proceeding to subsequent steps.

#### Step 1: Credential Existence

The terminal looks up the Authorization_Descriptor corresponding to `credential.id` in local storage.

- Not found → return `E_DESCRIPTOR_NOT_FOUND`

#### Step 2: Revocation Status

The terminal checks the local revocation list to confirm that the `descriptor_id` has not been revoked.

- Revoked → return `E_DESCRIPTOR_REVOKED`

#### Step 3: Validity Period

The terminal checks that the current time is within the `[not_before, not_after]` interval.

- Current time < `not_before` → return `E_DESCRIPTOR_NOT_YET_VALID`
- Current time ≥ `not_after` → return `E_DESCRIPTOR_EXPIRED`

Terminal clock source: MUST use a calibrated system clock. The terminal SHOULD handle validation requests cautiously when not online for extended periods (see §3.6 Clock Drift Handling).

#### Step 4: Subject Match

The terminal verifies that `payload.subject_fay_id == AuthRequest.fay_id`.

- Mismatch → return `E_SUBJECT_MISMATCH`

#### Step 5: Terminal Match

The terminal verifies that `payload.terminal_id == current terminal ID`.

- Mismatch → return `E_TERMINAL_MISMATCH`

#### Step 6: Resource and Mode Match

The terminal iterates over `payload.grants` and looks for a `Grant` satisfying:

1. `Grant.resource_pattern` matches `AuthRequest.resource_id` per §2.3.5 rules
2. `Grant.modes` contains `AuthRequest.access_mode`
3. If `Grant.constraints` is not empty, all constraints are satisfied (see §7.4 for constraint semantics)

- No matching Grant found → return `E_AUTHORIZATION_INSUFFICIENT`

#### Step 7: Signature Validation

The terminal re-validates the payload's signature using the Verification_Key corresponding to `signature.key_id`.

Note: The fast checks of steps 1–6 are completed before signature validation, providing reasonable failure semantics (e.g., reporting "credential revoked" before "signature failed"). However, signature validation MUST have been executed and passed before validation passes and success is returned.

- Signature validation failed → return `E_INVALID_SIGNATURE`
- Signing key revoked or expired → return `E_VERIFICATION_KEY_INVALID`

### 3.3.3 After Validation Passes

After all 7 steps pass, the terminal creates a Session per the flow in Chapter 5 and returns to iFay_Runtime:

```
AuthResult (body of ProtocolMessage, success) {
  required status         : "granted"
  required session_id     : Session_ID
  required granted_modes  : array<AccessMode>
  required session_expires_at : timestamp
}
```

`session_expires_at` SHOULD equal `min(not_after, current time + default maximum session duration)`.

### 3.3.4 Signature Validation Details

Specific steps of signature validation:

1. Extract `descriptor.signature.key_id` and look up the corresponding public key in the terminal's Verification_Key store
2. Verify that the Verification_Key is valid at the current time (`valid_from ≤ current time ≤ valid_until`, if valid_until is set)
3. Re-CBOR-serialize `descriptor.payload` per §2.3.3 rules
4. Verify the signature of the serialized bytes using the Verification_Key and `descriptor.signature.algorithm`
5. After successful validation, MUST cache the result (the same descriptor only requires signature validation once during its lifecycle)

## 3.4 Revocation Flow

### 3.4.1 Revocation Initiation

The authorizer initiates a revocation request through Descriptor_Issuer. The specific interaction form for revocation requests is outside the scope of this specification.

After receiving a revocation request, Descriptor_Issuer MUST:

1. Verify that the revocation request is from the original authorizer or an entity with revocation authority
2. Generate a `RevocationStatement` per §2.8
3. Sign the revocation statement with the same signing key used for the original Authorization_Descriptor
4. Add the revocation statement to the revocation list maintained by Descriptor_Issuer

### 3.4.2 Revocation Distribution

Revocation statements are distributed to terminals through the following methods; the terminal MUST support at least two of them:

1. **Pull-based synchronization**: When online, the terminal proactively pulls incremental revocation lists from Descriptor_Issuer or a revocation service. Synchronization frequency is determined by terminal policy and SHOULD be no less than once per hour (during online periods)
2. **Push-based notification**: Descriptor_Issuer pushes revocation statements to the terminal via Registration_Authority or a dedicated revocation distribution service
3. **In-band embedding**: A digest of the recent revocation list is carried in the Trusted_Ticket metadata, allowing tickets obtained while online to automatically carry revocation information

### 3.4.3 Terminal Revocation List Maintenance

The terminal's local revocation list MUST:

1. Permanently store all unexpired revocation statements
2. After a credential's `not_after` time has passed, MAY delete the corresponding revocation statement (the credential has expired naturally)
3. Validate the signature of each revocation statement and reject revocation statements with invalid signatures

### 3.4.4 Revocation Effective Time

The effective time of a revocation statement is:

```
Effective time = max(time the revocation statement reaches the terminal, RevocationStatement.revoked_at)
```

The terminal MUST ensure that subsequent validation requests after a revocation statement reaches the terminal immediately reject the credential. The terminal MAY proactively check whether all active Sessions reference revoked credentials and, if so, forcibly terminate those Sessions per the rules in Chapter 5.

### 3.4.5 Revocation Delay Window

Due to the inherent delay of offline distribution, the following unavoidable revocation delay windows exist:

| Scenario | Maximum Delay |
|------|---------|
| Terminal continuously online | One synchronization cycle (default ≤ 1 hour) |
| Terminal briefly offline | Next synchronization after reconnection |
| Terminal long-term offline | Maximum delay is the credential's `not_after - revocation time`, but no more than 90 days (the maximum credential validity period) |

Issuers SHOULD limit the maximum revocation delay by setting shorter `not_after` values (e.g., 7 days).

## 3.5 Renewal Flow

Renewal is essentially the issuance of a new Authorization_Descriptor to replace the old version, not a modification of the existing credential.

### 3.5.1 Renewal Policy

The authorizer or an automatic renewal mechanism may trigger renewal in the following scenarios:

1. The credential is approaching `not_after` and the authorization relationship is still valid
2. The authorization scope needs to be adjusted (adding/removing grants)
3. The authorization constraints need to be adjusted (modifying constraints)

### 3.5.2 Renewal Flow

Renewal process:

1. Descriptor_Issuer issues a new Authorization_Descriptor following the §3.1 flow, using a new `descriptor_id`
2. The new credential is submitted to the terminal and stored following the §3.2 flow
3. The old credential MAY be proactively revoked by the issuer (per §3.4) or MAY expire naturally

During the period when both new and old credentials coexist, the terminal's processing priorities are:

- Validation requests MUST preferentially match unexpired credentials
- When multiple unexpired credentials all match, the credential with the most recent `issued_at` is used

### 3.5.3 Disallowed Renewal Behaviors

Implementations MUST NOT:

1. Modify any field of an issued credential (including metadata)
2. Reuse the `descriptor_id` of an old credential
3. Modify the semantics of an old credential while it is still valid (must be done through revocation + new issuance)

## 3.6 Clock Drift Handling

Terminal clocks may drift due to long-term offline operation or hardware issues. This section defines the handling rules under clock drift conditions.

### 3.6.1 Clock Tolerance

The terminal MAY introduce tolerance when validating the validity period:

- For `not_before`: May allow up to 5 minutes of early tolerance (accept `current time ≥ not_before - 5min`)
- For `not_after`: MUST NOT introduce tolerance (expired is expired)

Tolerance is used only to compensate for short-term clock offsets and SHOULD NOT be used to bypass validity period constraints.

### 3.6.2 Clock Trustworthiness

The terminal SHOULD detect the following clock anomalies and take protective measures:

- Significant clock backward (system clock jumps to the past > 1 hour): Reject validation requests until clock synchronization
- Significant clock forward (system clock jumps to the future > 24 hours): Reject validation requests until clock synchronization

## 3.7 Complete Flow Diagram

```
[Descriptor_Issuer]                    [Fay/iFay_Runtime]                   [Terminal/Protocol_Engine]
        │                                       │                                       │
        │── Issue AuthorizationDescriptor ────→│                                       │
        │   (with digital signature)            │                                       │
        │                                       │                                       │
        │                                       │── DescriptorSubmit ─────────────────→│
        │                                       │                                       │── Verify signature + store
        │                                       │←─ DescriptorSubmitResult ──────────── │
        │                                       │                                       │
        │                                       │── AuthRequest (with descriptor_id)──→│
        │                                       │                                       │── 7-step validation
        │                                       │←─ AuthResult (granted) ──────────────│
        │                                       │                                       │── Create Session
        │                                       │                                       │
        │                                       │       【Fay continuously accesses resource】│
        │                                       │                                       │
        │── Revoke RevocationStatement ─────────────────────────────────────────────────→│
        │                                       │                                       │── Add to revocation list
        │                                       │                                       │── Forcibly terminate related Session
```
