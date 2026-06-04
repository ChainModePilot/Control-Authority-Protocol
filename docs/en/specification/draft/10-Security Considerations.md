# Chapter 10: Security Considerations

This chapter describes the threat model, known risks, and mitigations of the CAP protocol. This chapter is non-normative content (does not use keywords such as MUST/SHOULD/MAY), but the risks and mitigations described in this chapter SHOULD be taken seriously by implementers.

## 10.1 Threat Model

The threat model of the CAP protocol is based on the following assumptions and considerations.

### 10.1.1 Trust Assumptions

The CAP protocol assumes the following entities are trusted:

1. **Registration_Authority**: Trust anchor; its key distribution capability is the foundation of protocol security
2. **Pre-installed trust anchor keys**: RA public keys pre-installed at terminal manufacturing or deployment
3. **Terminal secure storage**: Hardware/software security modules used to store Verification_Keys and local signing keys

The CAP protocol does **not** assume the following entities are trusted:

1. **Fay instances**: Fay may be maliciously programmed or controlled by an attacker
2. **iFay_Runtime**: May contain bugs or be replaced
3. **Network channels**: May be eavesdropped, tampered with, or blocked
4. **Unauthorized application processes**: Other processes on the terminal may attempt to bypass CAP control

### 10.1.2 Attacker Capabilities

Considered attackers with the following capabilities:

| Attacker Type | Capability |
|-----------|------|
| Network attacker | Eavesdrop, tamper, block, replay network messages |
| Rogue Fay | Holds legitimate credentials but exhibits abnormal behavior |
| Credential stealer | Obtains a copy of someone else's credential through other channels |
| Insider threat | Descriptor_Issuer itself compromised |
| Physical attacker | Physical contact with terminal device |

The CAP protocol provides protection against the first 4 types of attackers; defense against physical attackers depends on the terminal's hardware security mechanisms (not addressed at the protocol level).

## 10.2 Known Risks and Mitigations

### 10.2.1 Credential Leakage Risk

**Risk**: After an Authorization_Descriptor file is stolen by an attacker, it may be submitted to the terminal by an entity other than the Fay holding the credential.

**Mitigations**:

- Authorization_Descriptor's `subject_fay_id` is bound to a specific Fay; the terminal verifies subject match at Step 4 of §3.3.2
- The attacker must simultaneously control the target Fay's runtime (i.e., attack Fay itself); credential leakage alone is insufficient to constitute an attack
- Credential storage MUST be encrypted (§3.2.3) to prevent offline extraction
- Trusted_Ticket limits the damage window through short validity periods (max 7 days) and real-time revocation queries

**Residual risk**: If the attacker fully controls the Fay process (including its keys), the credential may still be abused. This protocol cannot solve the problem of "fully compromised Fay" — this is a higher-level Fay integrity protection issue.

### 10.2.2 Revocation Delay Risk

**Risk**: There is a delay window between credential revocation and the revocation statement reaching the terminal, during which the revoked credential can still pass validation.

**Mitigations**:

- When the terminal is continuously online, revocation statements are synchronized within 1 hour by default (§3.4.5)
- When using Trusted_Ticket, the terminal MAY eliminate delay through real-time revocation queries (§4.3.2 Step 5)
- The credential validity ceiling (90 days offline / 7 days online) limits the maximum delay
- Issuers SHOULD use shorter validity periods (e.g., 1 day) in high-sensitivity scenarios

**Residual risk**: Long-term offline terminals may accept revoked credentials for days or longer. This is the inherent trade-off of the offline authorization mechanism — choosing availability over revocation timeliness.

### 10.2.3 Replay Attack Risk

**Risk**: An attacker captures legitimate protocol messages and replays them to the terminal.

**Mitigations**:

- AuthRequest contains `message_id` (UUID v7, time-related); the terminal MAY maintain a short-term seen-ID cache to reject replays
- Heartbeat's `sequence_number` is monotonically increasing; replays are detected (§5.4.2)
- TLS channels provide basic replay protection (each TLS session is independent)
- The revocation statement's `revocation_id` is unique; replays are invalid

**Residual risk**: Within a TLS session, if an attacker has already broken TLS encryption, message replays are possible. This is a TLS-level threat, not within the scope of the CAP protocol.

### 10.2.4 Clock Attack Risk

**Risk**: An attacker bypasses validity period checks by modifying the terminal clock or exposes clock-related vulnerabilities.

**Mitigations**:

- The terminal clock source should be trusted (system clock, NTP synchronization)
- §3.6 defines clock drift detection: validation is rejected upon significant clock jumps
- Terminals offline for too long SHOULD prompt the user to re-synchronize the clock

**Residual risk**: An attacker physically controlling the terminal may be able to manipulate the clock. Physical security is the responsibility of the terminal hardware.

### 10.2.5 Descriptor_Issuer Compromise Risk

**Risk**: Descriptor_Issuer is fully controlled by an attacker, who can issue arbitrary credentials.

**Mitigations**:

- §8.6.2 defines an emergency key revocation mechanism
- Registration_Authority can invalidate all credentials issued by Descriptor_Issuer by revoking its Verification_Key
- The terminal immediately terminates related Sessions upon receiving Verification_Key revocation (§5.5)
- The security of the trust root (Registration_Authority) is critical — its compromise will collapse the entire trust chain

**Residual risk**: Credentials issued before Verification_Key revocation may still be abused. The revocation window is unavoidable.

### 10.2.6 Resource Exhaustion Attack Risk

**Risk**: A malicious Fay exhausts terminal resources (connections, Sessions, storage) through massive requests.

**Mitigations**:

- §5.8 defines Session quantity limits
- §3.2.3 defines credential storage ceiling
- Heartbeat timeout reclaims invalid Sessions and releases resources
- The terminal SHOULD implement rate limiting based on fay_id (in addition to §7.4.2 frequency constraints)

**Residual risk**: Multiple legitimately issued credentials may collude to exhaust resources. Terminal policy should limit the total quota of a single Fay.

### 10.2.7 Handover Race Condition Risk

**Risk**: An attacker exploits intermediate states in the handover flow (e.g., the "no active Session" state at §6.5.1 [T2]) to seize the resource.

**Mitigations**:

- During handover, the resource is in pre-occupancy state (`handover_pending`) and does not accept other Session requests (§6.5.3)
- Even if failure occurs between [T2] and [T3], the resource clearly enters "idle" state, with subsequent requests competing fairly (§6.5.2)
- Handover timeout prevents the resource from remaining in pending state for long

**Residual risk**: In the extreme case of [T3]–[T4] failure, the original Fay's session is unrecoverable — AuthRequest must be re-initiated. This is a necessary cost, avoiding the more complex security analysis brought by "automatic recovery".

### 10.2.8 Degradation Attack Risk

**Risk**: An attacker forces the terminal into degradation mode (§4.5) by blocking the connection between the terminal and Ticket_Issuer, thereby bypassing real-time revocation checks.

**Mitigations**:

- The terminal rejects new Trusted_Ticket acceptance by default in degradation mode
- The validity period of tickets already converted to local Authorization_Descriptor is short (max 7 days)
- The terminal SHOULD hint to iFay_Runtime that it is currently in degradation mode, allowing sensitive operations to actively reject

**Residual risk**: Locally converted credentials during the degradation period may still be used. This is the inherent trade-off of the offline-first design.

### 10.2.9 Side Channel Attack Risk

**Risk**: Inferring sensitive information (e.g., the existence of valid credentials) by observing CAP protocol response time or error code details.

**Mitigations**:

- Error codes do not expose internal details (§9.9)
- Implementations SHOULD maintain similar response times across different validation failure branches
- Do not leak key fingerprints, internal IDs, etc., in error responses

**Residual risk**: Completely eliminating side channel attacks is extremely difficult. This protocol layer provides basic measures; deep defense depends on implementation quality.

## 10.3 Deployment Security Recommendations

Deployers implementing the CAP protocol should consider the following security practices:

### 10.3.1 Key Management

- Use hardware security modules (HSM, TPM, Secure Enclave) for private key storage
- Key rotation period no longer than 90 days (terminal local keys) / 365 days (Issuer keys)
- Establish key leakage emergency response procedures
- Critical signing operations require multi-person approval or hardware PIN

### 10.3.2 Monitoring and Auditing

- Log all authorization validation failure events with alerts
- Monitor abnormal revocation request frequencies (may indicate Descriptor_Issuer compromise)
- Audit logs use independent persistent storage with anti-tampering
- Comprehensively audit all operations on high-sensitivity resources (configure mode, `require_human_confirmation` constraint)

### 10.3.3 Terminal Hardening

- Enable OS mandatory access control (SELinux, AppArmor)
- Use sandboxes to isolate iFay_Runtime from other processes
- Disable unnecessary system services to reduce attack surface
- Regularly update system patches

### 10.3.4 Network Segmentation

- Isolate Registration_Authority deployment networks from public networks
- Deploy Descriptor_Issuer in controlled environments
- Restrict communication from terminal to Issuer to necessary ports/protocols
- Implement mTLS (mutual authentication) to protect critical communications

## 10.4 Protocol Limitations

This specification explicitly does not address the following security issues:

1. **Fay integrity**: The CAP protocol assumes that the integrity of the Fay instance's own code and behavior is guaranteed by other mechanisms (e.g., code signing, runtime integrity measurement)
2. **Physical security**: When the terminal is fully controlled by a physical attacker, the software level cannot provide protection
3. **Audit log format**: v1 does not standardize the audit log format (blueprint §3.2 exclusion item)
4. **Cross-terminal identity federation**: v1 does not support cross-terminal unified identity management; each terminal independently trusts Registration_Authority

## 10.5 Security Evolution in Subsequent Versions

Security enhancements planned for introduction in future versions (v2+):

1. Distributed revocation consensus (blueprint §3.2 exclusion item)
2. Audit log standardized format (blueprint §3.2 exclusion item)
3. Quantum-resistant cryptographic algorithms (tracking NIST PQC standards)
4. Deeper integration of zero-trust principles
5. Formal protocol security proof
