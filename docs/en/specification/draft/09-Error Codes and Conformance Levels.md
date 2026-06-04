# Chapter 9: Error Codes and Conformance Levels

This chapter defines the standardized error code table of the CAP protocol and the way implementations declare conformance levels.

## 9.1 Error Code Format

Error codes use the `E_` prefix with uppercase underscore naming. The error code string itself is normative — implementations of this specification MUST return the corresponding error code string per the definitions in this chapter to enable consistent diagnosis across implementations.

Error response message structure:

```
ErrorBody {
  required error_code : string
  required message    : string
  optional details    : map<string, string>
  optional retry_after_ms : uint32
}
```

| Field | Description |
|------|------|
| `error_code` | Standard error code defined in this chapter |
| `message` | Human-readable error description (language not fixed; English or zh-CN recommended) |
| `details` | Additional context for the error (e.g., the specific field violated) |
| `retry_after_ms` | Recommended wait time for client retry; meaningful only for transient errors |

## 9.2 Credential-Related Error Codes

Applies to validation failure scenarios for Authorization_Descriptor and Trusted_Ticket.

| Error Code | HTTP Equivalent | Trigger Condition | Conformance |
|-------|----------|---------|--------|
| `E_DESCRIPTOR_NOT_FOUND` | 404 | The referenced descriptor_id was not found locally on the terminal | Terminal MUST |
| `E_INVALID_STRUCTURE` | 400 | Credential structure does not conform to §2.3 / §2.4 | Terminal/Issuer MUST |
| `E_INVALID_SIGNATURE` | 401 | Signature validation failed | Terminal MUST |
| `E_VERIFICATION_KEY_INVALID` | 401 | The Verification_Key corresponding to signature key_id is unregistered or revoked | Terminal MUST |
| `E_UNKNOWN_ISSUER` | 401 | The credential's issuer_id is not in the terminal trust list | Terminal MUST |
| `E_DESCRIPTOR_REVOKED` | 403 | Authorization_Descriptor has been revoked | Terminal MUST |
| `E_TICKET_REVOKED` | 403 | Trusted_Ticket has been revoked | Terminal MUST |
| `E_DESCRIPTOR_EXPIRED` | 403 | Credential past `not_after` | Terminal MUST |
| `E_DESCRIPTOR_NOT_YET_VALID` | 403 | Current time earlier than `not_before` | Terminal MUST |
| `E_TICKET_MALFORMED` | 400 | JWS parsing failed or typ incorrect | Terminal MUST |
| `E_DUPLICATE_DESCRIPTOR_ID` | 409 | Submitted credential conflicts with stored credential ID with mismatching content | Terminal MUST |
| `E_VALIDITY_OUT_OF_RANGE` | 400 | Credential's `not_after - not_before` exceeds the limit | Terminal/Issuer MUST |
| `E_REVOCATION_QUERY_TIMEOUT` | 504 | Online revocation query timeout | Terminal SHOULD |

## 9.3 Authorization Scope Error Codes

Applies to scenarios where the credential is legitimate but the authorization scope does not match.

| Error Code | HTTP Equivalent | Trigger Condition |
|-------|----------|---------|
| `E_SUBJECT_MISMATCH` | 403 | Credential subject does not match requester fay_id |
| `E_TERMINAL_MISMATCH` | 403 | Credential terminal_id does not match current terminal |
| `E_AUTHORIZATION_INSUFFICIENT` | 403 | No matching Grant for (resource, mode) |
| `E_RATE_LIMIT_EXCEEDED` | 429 | Exceeds the frequency limit in constraints |
| `E_OUT_OF_TIME_WINDOW` | 403 | Current time is outside the constraints time window |
| `E_OUT_OF_GEO_FENCE` | 403 | Terminal location is outside the constraints geofence |
| `E_UNSUPPORTED_CONSTRAINT` | 400 | Credential contains unrecognized constraint key |

## 9.4 Resource and Session Error Codes

Applies to errors related to resource occupancy and Session state.

| Error Code | HTTP Equivalent | Trigger Condition |
|-------|----------|---------|
| `E_RESOURCE_BUSY` | 409 | Resource already occupied in requested mode (violates read-write lock matrix) |
| `E_RESOURCE_UNAVAILABLE` | 503 | Resource currently unavailable (hardware disconnect, software not running, etc.) |
| `E_RESOURCE_NOT_FOUND` | 404 | resource_id does not exist |
| `E_SESSION_NOT_FOUND` | 404 | session_id does not exist |
| `E_SESSION_TERMINATED` | 410 | Session terminated, cannot continue operation |
| `E_SESSION_LIMIT_EXCEEDED` | 429 | Exceeds the Session quantity limit defined in §5.8 |
| `E_OS_INTEGRATION_FAILED` | 500 | OS access control delivery failed |

## 9.5 Handover-Related Error Codes

Applies to the handover flow defined in Chapter 6.

| Error Code | HTTP Equivalent | Trigger Condition |
|-------|----------|---------|
| `E_HANDOVER_INVALID_SOURCE` | 400 | source_session_id does not exist or state does not permit handover |
| `E_HANDOVER_INVALID_TARGET` | 400 | target_credential validation failed |
| `E_HANDOVER_REJECTED_BY_POLICY` | 403 | Policy evaluation rejects handover |
| `E_HANDOVER_FAILED_AT_RELEASE` | 500 | Source Session release failed during handover |
| `E_HANDOVER_FAILED_AT_ACQUIRE` | 500 | Target Session creation failed during handover |
| `E_HANDOVER_TIMEOUT` | 504 | Handover timeout |
| `E_HANDOVER_IN_PROGRESS` | 409 | Resource has a handover in progress; new request rejected |
| `E_HANDOVER_RETRY_LIMIT` | 429 | Retry count exceeds limit |

## 9.6 Protocol and System Error Codes

| Error Code | HTTP Equivalent | Trigger Condition |
|-------|----------|---------|
| `E_PROTOCOL_VERSION_UNSUPPORTED` | 400 | Message version field unsupported by implementation |
| `E_INVALID_MESSAGE` | 400 | Protocol message format does not conform to §2.6 |
| `E_INTERNAL_ERROR` | 500 | Terminal internal error (does not expose details) |
| `E_NOT_IMPLEMENTED` | 501 | Implementation does not provide the optional capability |
| `E_STORAGE_FULL` | 507 | Terminal credential storage full |

## 9.7 Error Code Extension

Implementations MAY define custom error codes, but MUST:

1. Use the `E_VENDOR_<vendor_name>_<code>` naming format (e.g., `E_VENDOR_ACME_QUOTA_EXCEEDED`)
2. Not conflict with standard error codes defined in this chapter
3. Clearly specify semantics in implementation documentation

iFay_Runtime SHOULD treat unrecognized custom error codes as `E_INTERNAL_ERROR`.

## 9.8 Conformance Level Declaration

Implementations MUST declare the conformance levels they satisfy through the following.

### 9.8.1 Conformance Claim Structure

```
ConformanceClaim {
  required protocol_version : string                  // E.g., "1.0.0" or "draft"
  required levels           : array<ConformanceLevel> (len 1..3)
  optional optional_features : array<string>
  optional vendor_extensions : array<string>
}

ConformanceLevel = enum["terminal", "issuer", "runtime"]
```

| Field | Description |
|------|------|
| `protocol_version` | CAP protocol version followed by the implementation |
| `levels` | Conformance levels satisfied by the implementation (see §0.5) |
| `optional_features` | Optional capabilities supported by the implementation (e.g., `aggregated_heartbeat`, `ai_model_handover_policy`) |
| `vendor_extensions` | Vendor extension capabilities supported by the implementation |

### 9.8.2 Declaration Publishing Channels

Implementations SHOULD publish ConformanceClaim through the following channels:

1. Explicit declaration in product documentation
2. Returned through specific query interfaces (defined by the implementation)
3. Declared in SDK package metadata

### 9.8.3 Conformance Testing

The CAP protocol's Conformance Test Suite is maintained by the project as an auxiliary asset, but is outside the normative scope of this specification. Any implementation SHOULD pass the test suite for the corresponding conformance level, but MUST NOT rely on the test suite as the sole guarantee of correctness.

## 9.9 Error Code Usage Recommendations

Implementations SHOULD follow the following error code usage principles:

1. **Prefer the most specific error code**: For example, `E_TICKET_REVOKED` is preferred over `E_TICKET_MALFORMED` (even if both could appear during analysis)
2. **Do not expose internal details through error codes**: Avoid exposing key fingerprints, internal IDs, and other sensitive information in the error message field
3. **Provide diagnostic assistance via the details field**: For example, `details["field"] = "not_after"` indicates which specific field is non-compliant
4. **Use retry_after_ms for transient errors**: For example, `E_RESOURCE_BUSY`, `E_HANDOVER_IN_PROGRESS`

## 9.10 Synchronization of Error Codes with schema.json

`schema/{version}/schema.json` MUST contain the complete error code enumeration definition, kept consistent with this chapter. When the error code list in this chapter is updated, schema.json MUST be synchronously updated.
