## Chapter 1: Protocol Positioning and Boundaries

### 1.1 CAP's Role in the iFay Ecosystem

Control Authority Protocol (CAP) assumes a single and well-defined core responsibility within the iFay ecosystem: **verifying whether a Fay (iFay or coFay) has been authorized by its Human Prime (Natural_Person) or official post (Official_Post) to legitimately access terminal resources (Terminal_Resource)**.

Specifically, the CAP protocol is responsible for the following:

- **Authorization Verification**: When a Fay requests access to software or hardware resources on a terminal, the CAP protocol verifies whether the authorization credentials it carries (Authorization_Descriptor or Trusted_Ticket) are legitimate, valid, and have not been revoked. For example, when a user's iFay wants to open the phone camera to take a photo, the terminal must first verify that the iFay has indeed been authorized by the user to use the camera
- **Session Management**: Establishing control sessions (Sessions) for Fays that pass authorization verification, and managing the complete lifecycle of those sessions. For example, after an iFay is authorized and begins operating a browser, the entire process from opening the browser to closing it constitutes a complete control session
- **Control Authority Coordination**: Coordinating the allocation and handover of terminal resource control authority among multiple Fays or between Fays and human users. For example, a manually controlled drone now needs to be handed over to a Fay for autonomous flight; or two iFays need to use the same printer in succession
- **Resource Access Tiering**: Managing resource access in tiers by operation type (read, write, execute, configure). For example, an iFay reading temperature sensor data only requires read permission, while modifying the air conditioner's temperature setting requires write permission
- **Liveness Detection**: Monitoring the liveness status of active sessions and promptly reclaiming resources occupied by expired sessions. For example, after an iFay loses connectivity due to a network interruption, the terminal detects a heartbeat timeout and automatically releases the camera control held by that iFay, preventing resources from being locked by "zombie sessions" indefinitely

The CAP protocol is the critical bridge connecting intelligent agents to terminal resources in the iFay ecosystem — it does not concern itself with who a Fay is or what it can do, only whether a Fay is permitted to do something.

### 1.2 Responsibilities Outside CAP's Scope

To ensure clear responsibility boundaries, the CAP protocol explicitly does not handle the following:

1. **Fay Identity Creation and Management**: Registration of Fay entities, identity identifier assignment, identity attribute maintenance, and other tasks are handled by the identity management subsystem within the iFay ecosystem. CAP only consumes identity information for authorization verification and does not participate in the identity creation and management process. For example, "what this iFay is called and who it belongs to" is determined by the identity management system; CAP only cares about "whether this iFay is allowed to use the camera"
2. **Fay Intelligent Reasoning Capabilities**: How a Fay understands user intent, plans operational steps, and performs intelligent reasoning belongs to the Fay's own intelligence layer and is unrelated to the CAP protocol. CAP makes no assumptions or requirements about a Fay's intelligence level. For example, an iFay decides to first open a browser and then search for flight tickets — this decision-making process is unrelated to CAP; CAP only performs authorization verification when the iFay actually requests to open the browser
3. **Specific Business Logic of Terminal Resources**: How software on the terminal responds to operational commands and how hardware executes specific functions — these business logic implementations are the responsibility of the terminal resources themselves. CAP is only responsible for authorization verification and access control, and does not intervene in the specific business processing of resources. For example, how a printer formats pages or which ink cartridge to use for printing — these are the printer's own business logic; CAP only verifies whether the iFay has the right to use the printer
4. **Network Communication Protocol Implementation**: CAP defines the logical flow and message formats for authorization verification, but does not prescribe the specific implementation of underlying network transport protocols. For example, whether the terminal communicates with iFay_Runtime via WebSocket or gRPC is not restricted by CAP
5. **Internal Security Mechanisms of Terminal Operating Systems**: The operating system's own user permission management, sandbox isolation, process security, and other mechanisms are outside CAP's jurisdiction. CAP collaborates through integration interfaces with the operating system. For example, Android's own application sandbox mechanism is managed by Android; CAP issues access control instructions through the interfaces provided by Android

### 1.3 Relationships with Other Subprojects

The CAP protocol has well-defined interaction relationships with the following subprojects within the iFay ecosystem:

| Subproject | Relationship Description | Interaction Boundary |
|------------|------------------------|---------------------|
| **iFay_Runtime** | CAP's primary caller. iFay_Runtime is responsible for Fay instance lifecycle management and scheduling. When a Fay needs to access terminal resources, iFay_Runtime initiates control authority requests on its behalf | iFay_Runtime → CAP: Initiates authorization verification requests; CAP → iFay_Runtime: Returns verification results and session information |
| **Registration_Authority** | CAP's trust infrastructure dependency. Registration_Authority is responsible for terminal hardware, software, and operating system registration, and distributes Verification_Keys to terminals | Registration_Authority → Terminal: Distributes Verification_Keys; Terminals use Verification_Keys to verify Authorization_Descriptor signatures |
| **Descriptor_Issuer** | CAP's authorization credential source. Descriptor_Issuer, authorized by a Natural_Person or Official_Post, is responsible for generating and issuing Authorization_Descriptors | Descriptor_Issuer → Fay: Issues Authorization_Descriptors; Fay carries Authorization_Descriptors to initiate access requests to terminals |
| **Identity Management Subsystem** | CAP's identity information source. CAP references Fay identity identifiers during authorization verification but does not participate in identity creation and management | Identity Management → CAP: Provides Fay identity identifier information (unidirectional dependency) |

The design principle of the CAP protocol is to maintain the cohesion of its own responsibilities: only perform authorization verification and control authority management, leaving identity management, intelligent reasoning, resource business logic, and other responsibilities to other subprojects in the ecosystem.

### 1.4 Applicable Scenarios

The core applicable scenario for the CAP protocol is: **an iFay takes over its Human Prime's terminal, using the terminal's software and invoking the terminal's hardware just as a human would**.

In this scenario, the Human Prime (Natural_Person) grants partial or full control of their terminal device to their iFay, which operates the terminal's client software (such as browsers, email clients, office software) and hardware devices (such as cameras, microphones, storage devices) on their behalf. The CAP protocol ensures that during this process:

- **Authorization Legitimacy**: The terminal can verify that the iFay has indeed been authorized by the Human Prime, rather than being an unauthorized illegal access. For example, if Zhang San's iFay attempts to operate Li Si's laptop, the terminal will deny access due to the lack of authorization credentials issued by Li Si
- **Offline Availability**: Even when the terminal is offline, as long as the iFay holds a valid Authorization_Descriptor, it can still legitimately access authorized resources. For example, when a user enables airplane mode on a flight, their iFay can still continue operating office software on the laptop using the pre-stored offline authorization file
- **Multi-party Coordination**: When multiple Fays or Fays and human users simultaneously need to access the same terminal resource, the CAP protocol provides control authority handover and resource access mode management capabilities. For example, a user is taking photos with their phone when the iFay also needs to use the camera for document scanning — the CAP protocol coordinates the usage order between the two
- **Security and Controllability**: All authorization verification and resource access operations can be audited, and authorization can be revoked at any time. For example, if a user discovers abnormal behavior from their iFay, they can immediately revoke its authorization for all terminal resources, and the iFay's active sessions will be forcibly terminated

The same protocol framework also applies to coFay scenarios — collaborative Fay entities, authorized by an Official_Post, take over organizational terminal devices to perform collaborative tasks.

### 1.5 Core Design Principles

The CAP protocol follows the core design principle of **offline-first, online-supplementary**.

**Offline authorization (Authorization_Descriptor) is the core mechanism.** Terminal devices should not be completely stripped of a Fay's takeover rights due to network unavailability. If a Fay has previously been authorized by its Human Prime, this authorization relationship should be stored locally on the terminal in the form of an encrypted file, enabling the terminal to independently complete authorization verification while offline. Authorization_Descriptor is the concrete implementation of this concept — it is an encrypted authorization descriptor file stored locally on the terminal, containing the scope of resources, permission types, and validity period for which the Fay is authorized. The terminal can complete verification through the local Descriptor_Validator without requiring network connectivity.

**Online tickets (Trusted_Ticket) are the supplementary mechanism.** In connected environments, Trusted_Tickets provide real-time authorization issuance and revocation status query capabilities, compensating for the shortcomings of offline authorization in terms of timeliness and revocation response speed. When online services are available, the terminal can obtain the latest authorization status through Trusted_Tickets; when online services are unavailable, the terminal automatically falls back to local Authorization_Descriptor verification, ensuring uninterrupted service.

The core considerations of this design principle are:

- **Availability First**: Network interruptions should not be a reason for a Fay to be unable to work; offline authorization ensures basic availability. For example, when a user's phone signal is interrupted in the subway, the iFay can still continue operating offline apps on the phone using the local authorization file
- **Enhanced Security**: Online tickets provide stronger real-time security guarantees when connected, including instant revocation and dynamic permission adjustments
- **Graceful Degradation**: The transition from online to offline is smooth and does not cause sudden service interruption. Trusted_Tickets issued online can be converted to local Authorization_Descriptor format for offline use. For example, a user authorizes their iFay to use the camera via an online ticket while on Wi-Fi; when the user moves out of Wi-Fi coverage, the authorization automatically degrades to offline mode and continues to be effective
