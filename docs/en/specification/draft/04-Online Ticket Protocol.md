# Chapter 4: Online Ticket Protocol

This chapter defines the issuance, query, conversion, and degradation flows of `Trusted_Ticket`. The flow in this chapter corresponds to the design intent in §2.2 of the blueprint.

## 4.1 Applicability Premises

Trusted_Ticket is used only when the terminal can connect to Ticket_Issuer or an online revocation service. When the online service is unavailable, the terminal automatically falls back to offline authorization per §4.5.

The terminal MUST simultaneously support both Authorization_Descriptor (Chapter 3) and Trusted_Ticket. The terminal MUST NOT refuse to accept legitimately signed Trusted_Tickets, even if its policy prefers offline authorization.

## 4.2 Issuance Flow

### 4.2.1 Issuance Request

The authorizer initiates an issuance request to Ticket_Issuer through a CAP-supporting client (e.g., mobile App, web console). The interaction between the client and Ticket_Issuer is outside the scope of this specification, but Ticket_Issuer MUST:

1. Verify that the request originates from an authenticated authorizer
2. Verify that the authorizer is authorized to grant access to the target Fay and resource
3. Apply local policy to validate the authorization scope (e.g., maximum validity, resource whitelist)

### 4.2.2 Issuance Steps

Ticket_Issuer MUST generate Trusted_Ticket following these steps:

1. **Construct Header**: Set `alg`, `typ`, `kid` per §2.4.2
2. **Construct Payload**: Fill in TicketPayload per §2.4.3; all required fields MUST be set
3. **Validate constraints**: Locally validate `exp - nbf ≤ 7 days`
4. **JSON serialization**: Serialize Header and Payload into UTF-8 JSON bytes
5. **Base64URL encoding**: Base64URL-encode the Header and Payload bytes separately
6. **Sign**: Compute the signature input `base64url(header) + "." + base64url(payload)` per RFC 7515
7. **Generate Compact Serialization**: Concatenate as the `header.payload.signature` string
8. **Register record**: Maintain the issued ticket status internally in Ticket_Issuer (for revocation queries)

### 4.2.3 Post-Issuance Delivery

Trusted_Ticket is delivered to the target Fay or its iFay_Runtime via an encrypted channel such as HTTPS. The specific delivery method is outside the scope of this specification.

## 4.3 Terminal Validation Flow

The differences between the terminal validating Trusted_Ticket and validating Authorization_Descriptor are:

1. Trusted_Ticket is fully transmitted to the terminal by iFay_Runtime at request time (not pre-stored)
2. The terminal MAY query Ticket_Issuer's revocation status in real time when online
3. Error codes upon validation failure carry the `E_TICKET_*` prefix

### 4.3.1 Validation Request

The AuthRequest sent by iFay_Runtime carries the complete Trusted_Ticket:

```
AuthRequest (body of ProtocolMessage) {
  required fay_id        : Fay_ID
  required resource_id   : Resource_ID
  required access_mode   : AccessMode
  required credential    : CredentialContent
}

CredentialContent {
  required type : enum["descriptor_ref", "ticket"]
  oneof:
    descriptor_id : string                  // type == "descriptor_ref"
    ticket        : string                  // type == "ticket", JWS Compact string
}
```

### 4.3.2 Validation Steps (Executed in Order)

The terminal MUST execute validation in the following order. Failure at any step immediately returns an error.

#### Step 1: JWS Parsing

The terminal parses the JWS Compact string:

- Split into the three parts `header.payload.signature`
- Base64URL-decode header and payload
- Verify `header.typ == "cap-ticket+jws"`
- Verify that `header.alg` is in the algorithm set permitted by §8

Failure at any step → return `E_TICKET_MALFORMED`

#### Step 2: Signature Validation

The terminal verifies the JWS signature using the Verification_Key corresponding to `header.kid` (per RFC 7515).

- Signature validation failed → return `E_INVALID_SIGNATURE`
- The key corresponding to `kid` is unregistered or revoked → return `E_VERIFICATION_KEY_INVALID`

#### Step 3: Validity Period

Validate `nbf` and `exp` per the rules in Step 3 of §3.3.2.

#### Step 4: Subject, Terminal, Resource, and Mode Match

Match per the rules in Steps 4–6 of §3.3.2. Error codes use the `E_TICKET_*` prefix (see §9).

#### Step 5: Online Revocation Query (Optional)

If terminal policy requires online revocation verification, the terminal MAY initiate a revocation status query to Ticket_Issuer:

```
TicketRevocationQuery {
  required jti : uuid
}

TicketRevocationResponse {
  required jti      : uuid
  required revoked  : boolean
  optional revoked_at : timestamp
}
```

- Query timeout (default 2 seconds) → the terminal SHOULD reject the request and return `E_REVOCATION_QUERY_TIMEOUT`, but MAY be configured to allow it through (accepting revocation delay risk)
- Query returns `revoked == true` → return `E_TICKET_REVOKED`
- Query failure (e.g., service unreachable) → the terminal MAY decide based on locally cached revocation status; if beyond cache validity, treat as timeout

The terminal SHOULD cache revocation query results, with cache validity not exceeding 5 minutes.

### 4.3.3 After Validation Passes

Processing after validation passes is consistent with §3.3.3, creating a Session and returning AuthResult.

## 4.4 Conversion to Offline Authorization_Descriptor

When `TicketPayload.convertible == true` (default), the terminal can convert the Trusted_Ticket to a local Authorization_Descriptor for offline use after validation passes.

### 4.4.1 Conversion Trigger

Conversion SHOULD be automatically performed in the following situations:

1. Trusted_Ticket validation passed and `convertible == true`
2. Terminal policy permits offline subsequent access
3. `TicketPayload.exp` is sufficiently far from the current time (recommended > 1 hour)

### 4.4.2 Conversion Steps

1. **Field mapping**: Map TicketPayload fields to DescriptorPayload per the §2.4.4 table
2. **Validity period constraint**: The converted `not_after` MUST NOT exceed `min(TicketPayload.exp, conversion time + 7 days)`
3. **Metadata retention**: Store key audit information of the original Trusted_Ticket in metadata:
   - `metadata["origin"] = "converted_from_ticket"`
   - `metadata["origin_jti"] = TicketPayload.jti`
   - `metadata["origin_iss"] = TicketPayload.iss`
   - `metadata["origin_kid"] = TicketHeader.kid`
4. **Local signing**: The terminal re-signs the converted payload using its locally stored key (see §4.4.3)
5. **Local storage**: Encrypt and store the converted Authorization_Descriptor per §3.2.3

### 4.4.3 Terminal Local Signing Keys

To support the conversion flow, the terminal MUST hold a pair of local signing keys:

- Private key: Stored in the terminal's secure storage, used only for the local conversion flow
- Public key: Registered as a Verification_Key of type `source == "pre-installed"` in its own key store

The `signature.key_id` of the converted Authorization_Descriptor MUST identify this local key, and `payload.issuer_id` MUST be set to `"local-conversion:" + terminal_id`.

When validating a locally converted Authorization_Descriptor, the terminal verifies the signature using the local public key. This design ensures:

- The converted credential can be independently validated offline, without depending on the original Ticket_Issuer
- The conversion trust chain is anchored to the terminal's own key security

### 4.4.4 Conversion Limitations

A locally converted Authorization_Descriptor MUST NOT:

1. Be trusted by other terminals (its signature key_id is local)
2. Be migrated for cross-terminal use
3. Be revoked through Descriptor_Issuer's revocation mechanism (must be deleted locally)

The terminal SHOULD proactively delete locally converted Authorization_Descriptors in the following situations:

1. Receipt of a revocation notification for the original Trusted_Ticket (correlated by jti)
2. The credential has expired for more than 24 hours

## 4.5 Online-to-Offline Degradation

### 4.5.1 Degradation Trigger

The terminal continuously monitors online service availability. When any of the following conditions are met, degradation is triggered:

1. The connection with Ticket_Issuer is unreachable for more than 30 seconds
2. Revocation query requests time out 3 consecutive times
3. The network stack reports global network unavailability

### 4.5.2 Degradation Behavior

After degradation, the terminal handles things per the following rules:

1. Refuse to accept new Trusted_Tickets (unless configured for offline acceptance) — to avoid being unable to verify revocation status online
2. Tickets already converted to local Authorization_Descriptors continue to be used per §3 rules
3. Authorization validation requests directly using Authorization_Descriptor are not affected by degradation
4. The terminal MAY hint to iFay_Runtime that it is currently in offline mode, allowing Fay to adjust behavior policies

### 4.5.3 Recovery

After the terminal detects online service recovery, it MUST:

1. Prioritize synchronizing the revocation list, applying any revocation statements potentially missed during the offline period to local state
2. Re-check active Sessions; for Sessions using locally converted Authorization_Descriptor whose original ticket has been revoked, forcibly terminate them per Chapter 5 rules
3. Restore normal Trusted_Ticket acceptance policy

The recovery process is transparent to iFay_Runtime, but the terminal MAY notify mode changes via `SessionStateChanged`.

## 4.6 Priority in Dual-Credential Scenarios

When a Fay simultaneously holds both a Trusted_Ticket and an Authorization_Descriptor for the same resource, the iFay_Runtime selection policy:

| Network State | Recommended Choice | Reason |
|---------|---------|------|
| Online and Ticket_Issuer reachable | Trusted_Ticket | Stronger timeliness, real-time revocation possible |
| Offline | Authorization_Descriptor or converted local credential | Trusted_Ticket cannot verify revocation online |

iFay_Runtime MAY provide a fallback credential in AuthRequest, allowing the terminal to try a backup credential when validation of the primary credential fails. This specification does not mandate a specific format for the fallback mechanism, but SHOULD be implemented through two independent AuthRequests to avoid protocol complexity.

## 4.7 Semantic Consistency Between Trusted_Ticket and Authorization_Descriptor

When iFay_Runtime uses both credential types, the terminal MUST guarantee:

1. **Validation result consistency**: The same authorization scope produces consistent pass/reject judgments under both credentials
2. **Error code semantic consistency**: Equivalent failure reasons (e.g., validity period, signature, revocation) use semantically corresponding error codes (see Chapter 9)
3. **Session creation consistency**: There is no difference in lifecycle management of Sessions created via the two credentials
