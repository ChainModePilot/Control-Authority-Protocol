## Chapter 3: Version Evolution Roadmap

This chapter defines the version evolution roadmap for the CAP protocol, specifying the functional scope and exclusions for v1, establishing version numbering rules and compatibility strategies, and describing the release process from draft to official version. The version roadmap provides clear path guidance for iterative development and community collaboration.

### 3.1 v1 Version Functional Scope

CAP protocol v1 focuses on establishing a complete foundational framework for terminal control authority management, encompassing the following 6 core capabilities:

1. **Offline Authorization (Authorization_Descriptor)**: The core mechanism of v1. Implements the complete lifecycle management of Authorization_Descriptors, including issuance, local storage, validation, revocation, and renewal. The terminal can independently complete authorization verification through locally stored Authorization_Descriptors while offline, ensuring the basic availability of Fays in environments without network connectivity. v1 will implement the complete trust chain model and local revocation list mechanism.

2. **Online Tickets (Trusted_Ticket)**: The supplementary mechanism of v1. Implements real-time authorization issuance and revocation status query capabilities in connected environments, supports conversion from Trusted_Tickets to the local Authorization_Descriptor format, and implements the smooth online-to-offline degradation strategy. v1 ensures the collaborative operation between online tickets and offline authorization.

3. **Session Management (Session lifecycle)**: Implements complete lifecycle management of control sessions, covering the three phases of creation, active, and termination. v1 supports the one-to-one binding relationship between Sessions and Terminal_Resources, and implements three session termination methods: active release, timeout termination, and revocation termination.

4. **Control Authority Handover Policy (Handover_Policy)**: Implements control authority transfer capabilities between multiple controlling parties, covering handover scenarios between Fays and between Fays and human users. v1 supports three policy types (priority rule script, AI model real-time decision, human user decision), guarantees the atomicity of the handover process, and implements the timeout rollback mechanism.

5. **Liveness Detection (Liveness_Detection)**: Implements a session liveness detection mechanism based on the combination of persistent connections and application-layer heartbeats. v1 supports dual determination logic (both persistent connection disconnection and heartbeat timeout must be satisfied simultaneously), configurable heartbeat intervals and timeout thresholds, and automatic termination and resource reclamation of expired sessions.

6. **Resource Access Mode (Resource_Access_Mode)**: Implements tiered resource access management by operation type, supporting four access modes: read, write, execute, and configure. v1 implements the complete read-write lock model, supporting concurrent multi-party access in read mode and exclusive control in write, execute, and configure modes.

The above 6 core capabilities constitute the complete feature set of v1, providing a foundational control authority framework for intelligent agents in the iFay ecosystem to securely take over terminals.

### 3.2 Features Explicitly Excluded from v1

The following features are explicitly not included in v1 and are marked as candidates for future versions:

| Excluded Feature | Description | Candidate Version |
|-----------------|-------------|-------------------|
| **Cross-terminal Session Migration** | Migrating an active Session from one terminal to another while maintaining session state continuity | v2+ |
| **Multi-terminal Collaborative Authorization** | Authorization state synchronization and collaborative verification mechanisms across multiple terminals | v2+ |
| **Authorization Delegation Chain** | A cascading authorization mechanism where a Fay delegates part of its obtained authorization to other Fays | v2+ |
| **Fine-grained Resource Access Control (ABAC)** | Attribute-based access control policies supporting more complex resource access condition expressions | v2+ |
| **Standardized Audit Log Format** | Unified audit log (Audit_Logger) output format and query interface specification | v2+ |
| **Protocol Message Encrypted Transport Specification** | Standardized definition of encryption methods and key negotiation processes for CAP protocol messages at the network transport layer | v2+ |
| **Distributed Revocation Consensus for Offline Authorization** | A consensus mechanism for multiple terminals to reach agreement on revocation status in offline environments | v3+ |
| **Dynamic Permission Escalation (Within Session)** | Dynamically escalating access permissions within an active Session lifecycle without creating a new Session | v3+ |

v1 follows the principle of "establish the foundation first, then expand capabilities," prioritizing the completeness and stability of the 6 core capabilities and leaving advanced features for subsequent version iterations.

### 3.3 Version Numbering Rules and Compatibility Strategy

#### Version Numbering Rules

The CAP protocol adopts Semantic Versioning format: **`MAJOR.MINOR.PATCH`**

- **MAJOR**: Incremented when the protocol undergoes non-backward-compatible major changes. For example, structural adjustments to core message formats or fundamental changes to the authorization verification process. A MAJOR version change means existing implementations need to be adapted
- **MINOR**: Incremented when the protocol adds backward-compatible features or capabilities. For example, adding a new resource access mode or extending the policy types of Handover_Policy. A MINOR version change does not affect the normal operation of existing implementations
- **PATCH**: Incremented when the protocol makes backward-compatible error corrections or documentation clarifications. For example, correcting ambiguous descriptions in the specification document or supplementing handling instructions for boundary cases

Draft versions use the **`draft`** identifier without a version number. Draft versions make no compatibility guarantees and may undergo breaking changes at any time.

Examples of officially released version numbers: `1.0.0`, `1.1.0`, `1.1.1`, `2.0.0`

#### Compatibility Strategy

The compatibility strategy of the CAP protocol follows these principles:

- **Backward Compatible Within the Same Major Version**: Within the same MAJOR version, newer protocol implementations must be able to process protocol messages from older versions. For example, a terminal implementing v1.2.0 must be able to process Authorization_Descriptors in v1.0.0 format
- **No Compatibility Guarantee Between Major Versions**: Backward compatibility is not guaranteed between different MAJOR versions. When the major version number is incremented, the protocol specification will explicitly list all breaking changes and migration guides
- **Schema Version Aligned with Protocol Version**: Each protocol version corresponds to a Schema version, with Schema files stored in directories named after the protocol version number (e.g., `schema/2025-10-25/`), ensuring clear and traceable version correspondence
- **Minimum Supported Version Declaration**: Protocol implementations may declare their minimum supported protocol version; protocol messages below that version will be rejected

### 3.4 Release Process from Draft to Official Version

The release of the CAP protocol from draft to official version follows this process:

**Phase 1: Draft**

- Protocol specifications and Schema definitions are stored in the `draft/` directory (e.g., `schema/draft/`, `specification/draft/`)
- The draft phase allows frequent breaking changes with no compatibility commitments
- Draft content accepts community feedback and review, with continuous iterative optimization
- Documents and Schemas in the draft phase can be updated at any time without following version number increment rules

**Phase 2: Release Candidate**

- When draft content becomes stable, it enters the release candidate phase
- Release candidate versions use the `-rc.N` suffix (e.g., `1.0.0-rc.1`)
- The release candidate phase only accepts error corrections and documentation clarifications, not new features
- Release candidate versions must pass comprehensive test verification, including Schema validation, multi-language document structural equivalence checks, and property tests

**Phase 3: Official Release**

- After sufficient verification, the release candidate version removes the `-rc.N` suffix to become an official version
- The official version's protocol specifications and Schema definitions are copied from the `draft/` directory to a directory named after the version date (e.g., `schema/2025-10-25/`)
- After official release, the protocol specifications and Schema definitions for that version are no longer modified (immutable); any changes are released through new versions
- At the time of official release, blueprint documents and protocol specifications in all 9 languages must be completed and pass structural equivalence verification

**Phase 4: Maintenance**

- After official release, the version enters the maintenance phase, accepting only PATCH-level error corrections
- When a new MAJOR or MINOR version is released, older versions enter a limited maintenance period and are eventually marked as deprecated
- Documents and Schema definitions for deprecated versions are retained in the repository for historical reference but no longer accept any updates
