## Chapter 2: Core Capability Matrix

This chapter provides a strategic-level description of the five core capabilities of the CAP protocol, covering offline authorization, online tickets, session management, control authority handover policies, and resource access modes. Each capability is elaborated from the dimensions of design motivation, lifecycle, key mechanisms, and policy configuration, providing top-level guidance for the subsequent development of protocol technical specifications.

### 2.1 Offline Authorization (Authorization_Descriptor)

#### Design Motivation

Authorization_Descriptor is the core authorization mechanism of the CAP protocol. Its design motivation stems from a fundamental judgment: **a terminal should not be completely stripped of a Fay's takeover rights when offline, because the Fay may have been previously authorized by its human host**.

In real-world scenarios, terminal devices frequently find themselves in states where the network is unavailable or unstable — airplane mode, underground spaces, remote areas, network failures, etc. If authorization verification relied entirely on online services, the moment the network is interrupted, a Fay would immediately lose all control over terminal resources, even if the human host (Natural_Person) or official post (Official_Post) had previously granted explicit authorization. For example, a wildlife photographer's iFay is helping operate a drone for aerial shots when the phone signal is lost after flying into a mountain valley — without an offline authorization mechanism, the iFay would immediately lose control of the drone, potentially causing it to crash. Such a design would make a Fay's availability severely dependent on network status, contradicting the iFay ecosystem's vision of "intelligent agents seamlessly taking over terminals."

Authorization_Descriptor persists the authorization relationship locally on the terminal in the form of an encrypted file, enabling the terminal to independently complete authorization verification while offline. This file contains the scope of resources, permission types, and validity period for which the Fay is authorized. The terminal-side Descriptor_Validator can verify its legitimacy and validity without requiring network connectivity.

#### Lifecycle Overview

The lifecycle of an Authorization_Descriptor consists of the following 5 phases:

1. **Issuance**: The Descriptor_Issuer generates and issues an Authorization_Descriptor after obtaining explicit authorization from the authorizer (Natural_Person or Official_Post). The issuance process includes determining the authorization scope, setting the validity period, generating digital signatures, and other steps. Upon completion of issuance, the Authorization_Descriptor is delivered to the corresponding Fay entity. For example, user Zhang San authorizes their iFay through the authorization management interface to use the laptop's camera and microphone for the next 30 days, and the Descriptor_Issuer generates an Authorization_Descriptor containing these permissions and delivers it to the iFay.

2. **Local Storage**: The Fay submits the obtained Authorization_Descriptor to the target terminal, which stores it in encrypted form in a local secure storage area. Local storage ensures the terminal can access authorization information while offline, forming the physical foundation of the offline authorization mechanism. For example, the iFay submits the authorization file to Zhang San's laptop, which encrypts and stores it in the local security chip.

3. **Validation**: When a Fay requests access to terminal resources, the terminal-side Descriptor_Validator performs validation on the locally stored Authorization_Descriptor. Validation includes: legitimacy of the digital signature (verified using the Verification_Key), whether the validity period has expired, whether the authorization scope covers the requested resources and operation types, and whether it has been revoked. For example, when the iFay requests to activate the camera, the terminal checks whether the authorization file's signature is legitimate, whether it is within the 30-day validity period, and whether it includes camera usage permission.

4. **Revocation**: When the authorizer decides to withdraw authorization, a revocation declaration is issued to invalidate the corresponding Authorization_Descriptor. Revocation declarations are distributed to terminals through multiple channels (see "Revocation Declaration Distribution Strategy" below). In subsequent validations, the terminal will reject Authorization_Descriptors that have been revoked. For example, Zhang San discovers the iFay's behavior does not meet expectations and immediately revokes its camera usage authorization; the terminal will reject the iFay's camera access request at the next validation.

5. **Renewal**: When an Authorization_Descriptor is about to expire or the authorization scope needs adjustment, the Descriptor_Issuer issues a new Authorization_Descriptor to replace the old version. The renewal process is essentially a new issuance; the old version automatically becomes invalid after the new version takes effect. For example, as the 30-day validity period is about to expire, Zhang San decides to renew and additionally authorize the iFay to use storage devices; the Descriptor_Issuer issues a new authorization file to replace the old version.

#### Trust Chain Model

The trustworthiness of offline authorization relies on a complete trust chain. The trust propagation path from the authorizer to the terminal-side validator is as follows:

```
Authorizer (Natural_Person / Official_Post)
    │
    │ Authorization Delegation
    ▼
Descriptor_Issuer (Authorization Descriptor Issuer)
    │
    │ Issues Authorization_Descriptor (with digital signature)
    ▼
Fay (iFay / coFay)
    │
    │ Submits Authorization_Descriptor
    ▼
Terminal-side Descriptor_Validator
    │
    │ Verifies signature using Verification_Key
    ▼
Validation Result (Approved / Rejected)
```

Key links in the trust chain:

- **Authorizer → Descriptor_Issuer**: The authorizer delegates issuance authority to the Descriptor_Issuer through a secure authorization delegation mechanism, ensuring that only authorized issuers can generate legitimate Authorization_Descriptors
- **Descriptor_Issuer → Fay**: The Descriptor_Issuer digitally signs the Authorization_Descriptor using its private key, and the Fay receives the signed authorization file
- **Fay → Descriptor_Validator**: The Fay submits the Authorization_Descriptor to the terminal, and the terminal-side Descriptor_Validator verifies the signature's legitimacy using the Verification_Key distributed by the Registration_Authority
- **Registration_Authority → Terminal**: The Registration_Authority, as the trust infrastructure, is responsible for distributing Verification_Keys to terminals, ensuring that terminals can verify the signature source of Authorization_Descriptors

#### Revocation Declaration Distribution Strategy

In offline scenarios, timely distribution of revocation declarations is a core challenge. Terminals need to obtain the latest revocation information as much as possible without being able to query online revocation services in real time. The CAP protocol adopts the following distribution strategies:

- **Local Revocation List**: The terminal maintains a local revocation list recording the identifiers of known revoked Authorization_Descriptors. Each time the terminal connects to the network, it automatically synchronizes the latest revocation list from the revocation service
- **Proactive Synchronization When Connected**: When the terminal transitions from offline to online status, it prioritizes synchronizing the revocation list to ensure local revocation information is as up-to-date as possible
- **Validity Period Fallback**: Authorization_Descriptors carry their own validity period. Even if a revocation declaration fails to arrive in time, expired Authorization_Descriptors will automatically become invalid, limiting the maximum impact window of revocation delays
- **Effective at Next Validation**: The terminal checks the local revocation list during each Authorization_Descriptor validation. Once a revocation declaration reaches the terminal, subsequent validation requests will immediately reject the revoked authorization

### 2.2 Online Tickets (Trusted_Ticket)

#### Positioning and Use Cases

Trusted_Ticket is the online authorization supplementary mechanism of the CAP protocol in connected environments. Its core positioning is: **providing real-time authorization issuance and revocation status query capabilities in connected environments**, compensating for the shortcomings of offline authorization in terms of timeliness and revocation response speed.

Typical use cases for Trusted_Ticket include:

- **Temporary Authorization**: When a human host needs to temporarily grant a Fay access to a specific resource, a Trusted_Ticket can be instantly issued through online services without going through the full Authorization_Descriptor issuance process. For example, a user temporarily needs their iFay to help print a document; with a one-tap authorization on their phone, the online service instantly issues a temporary ticket limited to printer use, without going through the full offline authorization issuance process
- **Real-time Revocation Verification**: In connected states, the terminal can query the revocation status of authorizations in real time through the Trusted_Ticket mechanism, obtaining more timely revocation information than the local revocation list. For example, a company administrator has just revoked a coFay's control over conference room equipment; the connected terminal can immediately learn of this revocation through the online ticket mechanism, rather than waiting for the next synchronization of the local revocation list
- **Dynamic Permission Adjustment**: In connected environments, the authorization scope can be dynamically adjusted through Trusted_Tickets without re-issuing an Authorization_Descriptor. For example, an iFay originally only had read permission for files; the user temporarily upgrades its permission to write on their phone, taking effect immediately through an online ticket

#### Relationship with Authorization_Descriptor

The relationship between Trusted_Ticket and Authorization_Descriptor is supplementary rather than substitutive:

- **Authorization_Descriptor is the Core**: Offline authorization is always the foundational mechanism of the CAP protocol. Trusted_Tickets cannot exist independently outside the Authorization_Descriptor system
- **Online Issuance Can Be Converted to Offline Format**: Trusted_Tickets issued online can be converted to the local Authorization_Descriptor format for offline use. This means that authorization obtained through Trusted_Tickets will not immediately become invalid due to network interruption
- **Priority Relationship**: When the terminal holds both a Trusted_Ticket and an Authorization_Descriptor simultaneously, the real-time authorization information provided by the Trusted_Ticket takes priority, as it offers stronger timeliness

#### Online-to-Offline Degradation Strategy

When online services become unavailable, the CAP protocol executes a smooth degradation strategy:

1. **Detect Online Service Status**: The terminal continuously monitors the connection status with the online authorization service
2. **Automatic Fallback**: When online services are unavailable, the terminal automatically falls back to local Authorization_Descriptor verification without interrupting the Fay's resource access
3. **Trusted_Ticket Conversion**: Trusted_Tickets issued during the connected period have already been converted to the local Authorization_Descriptor format for storage, ensuring they remain usable after degradation
4. **Post-recovery Synchronization**: When online services become available again, the terminal automatically resumes using the Trusted_Ticket mechanism and synchronizes any revocation information that may have been missed during the offline period

The degradation process is transparent to the Fay — the Fay does not need to be aware of whether the terminal is currently using online tickets or offline authorization. The authorization verification results remain consistent across both modes.

### 2.3 Session Management (Session)

#### Lifecycle

A Session (control session) is the complete lifecycle from authorization verification approval to access termination, consisting of the following 3 phases:

1. **Creation**: When a Fay's authorization verification passes, the Protocol_Engine creates a Session instance for it. At creation time, the Session records the associated Fay identifier, target Terminal_Resource, authorized permission list, and creation time. Upon successful creation, the Fay gains control over the target resource. For example, an iFay requests to use the laptop's browser; after authorization verification passes, the system creates a control session bound to the browser, and the iFay can operate the browser from that moment on.

2. **Active**: After creation, the Session enters the active state, during which the Fay can perform operations on the bound Terminal_Resource within the scope of authorized permissions. During the active period, the Liveness_Detection mechanism continuously monitors the session's liveness status, ensuring the Fay is still actively using the session. For example, the iFay is querying flight information through the browser for the user, continuously sending heartbeats during this time to indicate it is still actively in use.

3. **Termination**: A Session can be terminated through the following methods:
   - **Active Release**: The Fay actively releases the Session after completing operations, returning control of the Terminal_Resource. For example, the iFay actively releases browser control after completing the flight query
   - **Timeout Termination**: Liveness_Detection detects that the Fay session is no longer active (heartbeat timeout), automatically terminating the Session and reclaiming resources. For example, the iFay stops sending heartbeats due to a runtime crash, and the terminal automatically reclaims browser control after the timeout
   - **Revocation Termination**: When authorization is revoked, associated active Sessions are forcibly terminated. For example, the user discovers the iFay is browsing inappropriate content and immediately revokes authorization, forcibly terminating the browser session

After Session termination, the Fay's control over the Terminal_Resource is immediately released, and the resource returns to a state where it can be acquired by other controlling parties.

#### Binding Relationship with Terminal_Resource

A strict binding relationship exists between Sessions and Terminal_Resources: **one active Session corresponds to control over one Terminal_Resource**.

The core rules of this binding relationship:

- **One-to-One Binding**: Each active Session is bound to a specific Terminal_Resource, and the Fay performs operations on that resource through the Session. For example, an iFay obtains a Session bound to the "front camera" — it can only operate the front camera through this Session and cannot use it to operate the microphone
- **Exclusive Control**: For operation modes requiring exclusive access (write, execute, configure), the same Terminal_Resource can have at most one active exclusive Session at any given time. For example, when iFay-A is writing data to an SD card in write mode, iFay-B cannot simultaneously obtain a write Session for the SD card
- **Concurrent Read Access**: For the read mode, the same Terminal_Resource can have multiple active read-only Sessions simultaneously. For example, iFay-A and iFay-B can simultaneously read location data from the GPS module in read mode
- **Lifecycle Coupling**: When a Terminal_Resource becomes unavailable (e.g., hardware disconnected), all active Sessions bound to that resource will be terminated. For example, after a user unplugs a USB camera, all Sessions bound to that camera are automatically terminated

#### Liveness Detection Mechanism

Liveness_Detection detects whether a Fay session is still active through a combination of persistent connections and application-layer heartbeats. Its design intent is to promptly reclaim resources occupied by expired sessions.

The working mechanism of liveness detection:

- **Persistent Connection Maintenance**: The Fay and the terminal maintain a communication channel through a persistent connection, with the connection status serving as the baseline signal for liveness detection
- **Application-Layer Heartbeat**: On top of the persistent connection, the Fay periodically sends application-layer heartbeat messages indicating it is still actively using the current Session. The heartbeat interval and timeout threshold are configurable
- **Dual Determination**: A Session is only determined to have expired when both the persistent connection is disconnected and the application-layer heartbeat has timed out. This dual determination mechanism prevents false positives caused by brief network fluctuations
- **Resource Reclamation**: When a Session is determined to have expired, the Protocol_Engine automatically terminates the Session and releases the Terminal_Resource it occupied, making the resource available for other controlling parties

### 2.4 Control Authority Handover Policy (Handover_Policy)

#### Core Scenarios

Control authority handover occurs in scenarios where multiple controlling parties need to use the same Terminal_Resource in succession. The CAP protocol defines two types of core handover scenarios:

- **Control Authority Transfer Between Fays**: One Fay transfers its control over a Terminal_Resource to another Fay. For example, after an iFay responsible for data collection completes its task, it hands over camera control to an iFay responsible for video calls; or in a smart factory, after a coFay responsible for quality inspection finishes photographing products, it hands over industrial camera control to a coFay responsible for packaging
- **Control Authority Transfer Between Fays and Human Users**: A Fay returns control to a human user, or a human user delegates control to a Fay. For example, a drone is being autonomously piloted by an iFay when the pilot notices abnormal weather and needs to take manual control — the iFay immediately yields flight control; after the weather improves, the pilot returns control to the iFay to continue autonomous flight. Another example: a user is manually editing a document and needs the iFay to help with formatting, temporarily handing over document editor control to the iFay; after formatting is complete, the iFay returns control

#### Policy-based Configuration Capabilities

Handover_Policy provides policy-based configuration capabilities, supporting the following 3 policy types:

1. **Priority Rule Script**: Determines the priority of control authority handover through predefined rule scripts. The rule scripts calculate priority scores based on factors such as the Fay's role, task urgency, and resource type, with the controlling party having the highest priority gaining resource control. This policy is suitable for scenarios where handover rules are clear and can be predefined. For example, in a medical scenario, the iFay responsible for emergency alerts always has higher priority than the iFay responsible for routine data collection; when both simultaneously request the communication module, the emergency alert iFay automatically gains control.

2. **AI Model Real-time Decision**: An AI model determines control authority allocation in real time based on the current context. The AI model makes decisions by comprehensively considering factors such as the task status of multiple controlling parties, urgency of resource usage, and historical handover patterns. This policy is suitable for dynamic scenarios where handover rules are complex and difficult to cover with static rules. For example, in a smart home, an iFay responsible for security monitoring and an iFay responsible for video calls simultaneously request the camera; the AI model determines priority in real time based on contextual information such as whether there is a current security threat and whether the call is urgent.

3. **Human User Decision**: The decision authority for control authority handover is given to the human user. When multiple controlling parties compete for the same resource, the terminal presents a selection interface to the human user, who decides which party to grant control to. This policy is suitable for highly sensitive resources or scenarios requiring human judgment. For example, two iFays simultaneously request to use the user's banking app; the terminal pops up a prompt for the user to decide which one to authorize, since financial operations require human final approval.

The three policy types can be independently configured at the Terminal_Resource granularity — different resources on the same terminal can adopt different handover policies.

#### Atomicity Guarantee

The control authority handover process must satisfy atomicity requirements: **at any given moment, a Terminal_Resource has at most one active controlling party**.

Implementation principles for the atomicity guarantee:

- **Release Before Acquire**: During the handover process, the original controlling party's Session is terminated first, and then the new controlling party's Session is created, ensuring that two controlling parties never simultaneously hold control over the same resource. For example, when flight control of a drone is handed over from iFay-A to iFay-B, iFay-A's control session must be terminated first before iFay-B's control session can be created — there will never be a situation where two iFays simultaneously control the drone's flight
- **Indivisible**: The handover operation is executed as a whole — either it succeeds completely (original party releases + new party acquires) or it fails completely (rolls back to the pre-handover state)
- **No Intermediate State Exposure**: External observers see a consistent resource control state at any given moment and will not observe intermediate states during the handover process

#### Timeout Handling Principles

The control authority handover process may fail to complete within the expected time due to various reasons (network latency, unresponsive controlling party, etc.). The CAP protocol's principle for handling handover timeouts is: **handovers that fail to complete within the timeout should roll back to the pre-handover state, keeping the original controlling party's session unchanged**.

Specific handling rules:

- **Configurable Timeout Threshold**: Each handover policy can configure an independent timeout threshold to accommodate the timeliness requirements of different scenarios
- **Rollback Protection**: After a timeout is triggered, the handover operation automatically rolls back, and the original controlling party's Session remains in an active state unaffected. For example, iFay-A is controlling the camera and attempts to hand over control to iFay-B, but iFay-B fails to respond within the specified time due to network latency; the handover automatically rolls back, and iFay-A continues to maintain control of the camera
- **Notification Mechanism**: After a timeout rollback, the Protocol_Engine sends a handover failure notification to the relevant controlling parties, explaining the timeout reason
- **Retry Strategy**: After a handover failure, the policy configuration determines whether to automatically retry or wait for manual intervention

### 2.5 Resource Access Mode (Resource_Access_Mode)

#### Tiered Model

Resource_Access_Mode categorizes resource access into 4 modes by operation type, forming a permission tier from low to high:

1. **read**: Read-only access to a Terminal_Resource without modifying the resource state. Examples include reading real-time data from a temperature sensor, viewing photos in the user's album, and obtaining the current location from the GPS module. The read mode is shareable, allowing multiple Fays to simultaneously access the same resource in read mode — for instance, multiple iFays can simultaneously read data from the same temperature sensor without interfering with each other.

2. **write**: Data writing or state modification of a Terminal_Resource. Examples include saving captured photos to a storage device, modifying the temperature setting of a smart home device, and writing content to a document. The write mode is exclusive — only one controlling party is allowed to access the resource in write mode at any given time — for instance, two iFays cannot simultaneously write data to the same file, as this would cause data corruption.

3. **execute**: Executing operational commands on a Terminal_Resource. Examples include launching a navigation app on a phone, triggering a drone's takeoff command, and executing a print job on a printer. The execute mode is exclusive, ensuring that the execution of operational commands is not interfered with by other controlling parties — for instance, when one iFay is controlling a drone to execute a landing procedure, another iFay cannot simultaneously send a takeoff command.

4. **configure**: System-level configuration changes to a Terminal_Resource. Examples include modifying a camera's resolution and frame rate parameters, adjusting network firewall rules, and changing Bluetooth module pairing settings. The configure mode is exclusive and high-privilege, typically requiring a higher level of authorization to execute — for instance, modifying a router's security policy requires a higher authorization level than simply reading network status.

#### Read-Write Lock Essence

The concurrency control of Resource_Access_Mode is essentially a Read-Write Lock model:

- **Read operations allow concurrent access by multiple Fays**: Multiple Fays can simultaneously hold read mode Sessions for the same Terminal_Resource without interfering with each other. This is analogous to a Shared Lock in the read-write lock model. For example, three iFays can simultaneously read temperature and humidity data from the same weather sensor without blocking each other
- **Write, execute, and configure operations require exclusive control**: When any Fay accesses a resource in write, execute, or configure mode, other Fays cannot simultaneously access that resource in any write-type mode. This is analogous to an Exclusive Lock in the read-write lock model. For example, when one iFay is writing a video file to an SD card, another iFay cannot simultaneously write photos to the same SD card
- **Read-write mutual exclusion**: When an active write, execute, or configure Session exists on a resource, new read requests will also be blocked until the exclusive Session is released. Conversely, when active read Sessions exist on a resource, write, execute, and configure requests will wait until all read Sessions have ended. For example, when an iFay is modifying a configuration file (write mode), another iFay that wants to read the file must also wait for the write to complete, to avoid reading an inconsistent intermediate state

This read-write lock model maximizes the concurrent performance of read operations while ensuring data consistency and operational safety.

#### Relationship with Session Permissions

Resource_Access_Mode is closely linked to Session authorization permissions: **the Session's authorized permission list determines the operation types a Fay can perform**.

The specific relationship is as follows:

- **Permission List Constraint**: Each Session carries an authorized permission list at creation time, derived from the permission scope defined in the Authorization_Descriptor or Trusted_Ticket. The Fay can only access resources in modes included in the permission list
- **Principle of Least Privilege**: The Session's permission list should follow the principle of least privilege, containing only the minimum permission modes required for the Fay to complete its current task
- **No Permission Escalation**: After a Session is created, its permission list cannot be escalated during the session lifecycle. If a Fay requires a higher-privilege access mode, it must create a new Session through a new authorization verification
- **No Hierarchical Permission Inclusion**: Higher-privilege modes do not automatically include the capabilities of lower-privilege modes. For example, holding configure permission does not automatically grant read permission — each access mode requires independent authorization
