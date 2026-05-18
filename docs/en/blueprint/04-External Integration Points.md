## Chapter 4: External Integration Points

This chapter describes the integration points between the CAP protocol and external systems, specifying the responsibility boundaries, interaction directions, and communication protocol requirements for each integration point. The CAP protocol does not operate in isolation — it needs to collaborate with external systems such as iFay_Runtime, terminal operating systems, hardware drivers, and Registration_Authority to jointly manage terminal control authority. Understanding the interface boundaries and interaction methods of these integration points is a prerequisite for system integrators to correctly implement the CAP protocol.

### 4.1 Integration with iFay_Runtime

iFay_Runtime is the runtime environment for Fays (iFay or coFay), responsible for Fay instance lifecycle management and scheduling. From the CAP protocol's perspective, iFay_Runtime is the initiator of control authority requests — when a Fay needs to access terminal resources, iFay_Runtime initiates authorization verification requests to the CAP protocol layer on its behalf.

**Integration Responsibilities:**

- **Initiating Control Authority Requests**: iFay_Runtime submits authorization verification requests to the Protocol_Engine on behalf of the Fay, with requests carrying the Fay's identity identifier, target Terminal_Resource, and authorization credentials (Authorization_Descriptor or Trusted_Ticket)
- **Managing Fay Lifecycle**: iFay_Runtime is responsible for starting, pausing, resuming, and terminating Fay instances. When a Fay instance is terminated, iFay_Runtime notifies the Protocol_Engine to release all active Sessions held by that Fay
- **Maintaining Liveness Detection Channel**: iFay_Runtime is responsible for maintaining the persistent connection between the Fay and the terminal, and sending application-layer heartbeat messages at configured intervals to support the normal operation of the Liveness_Detection mechanism
- **Receiving Session Status Notifications**: The Protocol_Engine notifies iFay_Runtime of Session status changes (successful creation, handover requests, forced termination, etc.), which iFay_Runtime relays to the corresponding Fay instance

**Interaction Direction:** Bidirectional

iFay_Runtime initiates requests to the CAP protocol layer (authorization verification, session release, heartbeat sending), and the CAP protocol layer returns responses and pushes notifications to iFay_Runtime (verification results, session status changes, handover requests).

**Communication Protocol Requirements:**

- Communication between iFay_Runtime and Protocol_Engine uses a message-based request-response pattern, with message formats conforming to the CAP protocol Schema definition
- The liveness detection channel requires persistent connection support to enable application-layer heartbeats and real-time status push
- The communication channel should support TLS encryption to ensure the confidentiality and integrity of authorization credentials and session information during transmission
- The message serialization format should be consistent with the Schema definition file (schema.json) to ensure cross-implementation interoperability

### 4.2 Integration with Terminal Operating Systems

The terminal operating system is the manager of Terminal_Resources, responsible for unified management of software and hardware resources on the terminal. The CAP protocol, through integration with the operating system, translates authorization verification results into actual resource access control — the operating system allows or denies Fay access to specific resources based on instructions from the Protocol_Engine.

**Integration Responsibilities:**

- **Executing Resource Access Control**: The operating system allows or blocks Fay access to Terminal_Resources at the system level based on access control instructions issued by the Protocol_Engine. This includes file system access control, process permission management, device access permissions, etc.
- **Reporting Resource Status**: The operating system reports the current status of Terminal_Resources (available, occupied, unavailable) to the Protocol_Engine, enabling the Protocol_Engine to make accurate resource allocation decisions
- **Isolating Execution Environments**: The operating system provides process-level or sandbox-level isolation for different Fays' operations, ensuring that one Fay's operations do not affect other Fays' or human users' resource access
- **Forwarding Resource Events**: When the status of a Terminal_Resource changes (e.g., hardware disconnected, software crashed), the operating system forwards event notifications to the Protocol_Engine, triggering corresponding Session termination or resource reclamation processes

**Interaction Direction:** Bidirectional

The CAP protocol layer issues access control instructions and resource query requests to the operating system, and the operating system reports resource status and forwards resource events to the CAP protocol layer.

**Communication Protocol Requirements:**

- Communication between the CAP protocol layer and the operating system uses local inter-process communication (IPC) mechanisms, with the specific method depending on the operating system platform (e.g., Unix Domain Socket, Named Pipe, D-Bus, etc.)
- Access control instructions should be executed via synchronous calls to ensure the Protocol_Engine can immediately know the instruction execution result
- Resource event notifications use asynchronous push, with the operating system proactively notifying the Protocol_Engine when resource status changes
- The communication interface should follow the operating system's native security model, ensuring that only authorized Protocol_Engine processes can issue access control instructions

### 4.3 Integration with Hardware Drivers

Hardware drivers are the access interfaces for hardware-type Terminal_Resources (such as cameras, microphones, GPS modules, storage devices, etc.). The CAP protocol, through integration with hardware drivers, achieves fine-grained control authority management over hardware resources. Hardware drivers sit below the operating system in the CAP protocol's integration architecture, providing low-level access capabilities for hardware resources.

**Integration Responsibilities:**

- **Providing Hardware Resource Access Interfaces**: Hardware drivers expose hardware resource capability descriptions and operation interfaces to the CAP protocol layer, enabling the Protocol_Engine to understand the access modes supported by hardware resources (read, write, execute, configure)
- **Executing Hardware-level Access Control**: Hardware drivers execute access permission or denial at the hardware level based on control instructions passed from the Protocol_Engine through the operating system. For example, allowing or prohibiting a Fay from activating the camera or accessing the microphone
- **Reporting Hardware Status**: Hardware drivers report hardware device connection status, operational status, and exception information to upper layers, with this information ultimately reaching the Protocol_Engine for Session management decisions
- **Supporting Exclusive Control of Hardware Resources**: For hardware resources requiring exclusive access (such as a camera's video stream output), hardware drivers ensure at the low level that only one controlling party can have exclusive use at any given time

**Interaction Direction:** Unidirectional — CAP Protocol Layer → Hardware Drivers

The CAP protocol layer issues control instructions to hardware drivers through the operating system. Hardware driver status information is propagated upward through the operating system's device management layer. The CAP protocol layer does not establish direct communication channels with hardware drivers; instead, it interacts indirectly through the operating system as an intermediary. The hardware status reporting path is: Hardware Driver → Operating System → Protocol_Engine, which is part of the operating system integration point (4.2).

**Communication Protocol Requirements:**

- The CAP protocol layer does not communicate directly with hardware drivers; all interactions are completed indirectly through the operating system's device management interfaces (e.g., DeviceIoControl, ioctl, sysfs, etc.)
- Hardware resource capability descriptions should follow a unified resource description format, enabling the Protocol_Engine to manage different types of hardware resources in a consistent manner
- Hardware drivers should support the standard device access control interfaces provided by the operating system, ensuring that CAP protocol access control instructions can take effect at the hardware level
- For highly sensitive hardware resources (such as cameras and microphones), hardware drivers should support hardware-level access locking mechanisms to prevent bypassing operating system-level access control

### 4.4 Integration with Registration_Authority

Registration_Authority is a core component of the CAP protocol's trust infrastructure, responsible for terminal hardware, software, and operating system registration management, as well as distributing Verification_Keys to terminals. The Verification_Key obtained by terminals through Registration_Authority is the trust anchor for offline authorization verification — without a legitimate Verification_Key, the terminal cannot verify the digital signature of Authorization_Descriptors.

**Integration Responsibilities:**

- **Terminal Registration**: Terminal devices (including hardware, operating systems, and client software) complete registration through Registration_Authority, obtaining unique terminal identifiers and initial configuration information
- **Verification_Key Distribution**: Registration_Authority distributes Verification_Keys to registered terminals, which use these keys to verify the digital signatures of Authorization_Descriptors. The key distribution process must be completed through secure channels to prevent key tampering or theft
- **Key Update and Rotation**: When Verification_Keys need to be updated (e.g., key expiration, issuer change), Registration_Authority is responsible for pushing new keys to terminals and coordinating smooth transitions during the key rotation process
- **Registration Status Management**: Registration_Authority maintains terminal registration status, including registered, suspended, and deregistered. A terminal's registration status affects whether it can receive Verification_Key updates and participate in the CAP protocol's authorization verification process

**Interaction Direction:** Unidirectional — Registration_Authority → Terminal

Registration_Authority distributes Verification_Keys and registration information to terminals, which passively receive them. Terminals do not initiate authorization verification requests or report operational status to Registration_Authority — terminal authorization verification is completed entirely locally (using the distributed Verification_Key) without requiring real-time communication with Registration_Authority. Registration requests initiated by terminals to Registration_Authority are a one-time initialization process and do not constitute routine interactions during CAP protocol runtime.

**Communication Protocol Requirements:**

- Communication between Registration_Authority and terminals must be completed through secure channels (e.g., TLS/mTLS) to ensure the confidentiality and integrity of Verification_Keys during transmission
- The key distribution protocol should support both offline pre-provisioning and online update modes: terminals can be pre-provisioned with initial Verification_Keys at the factory, with subsequent key updates received through online channels
- Key update messages should include version numbers and effective times, supporting smooth transition periods between old and new keys to avoid authorization verification interruptions caused by key rotation
- Registration_Authority should provide a key distribution confirmation mechanism to ensure terminals have successfully received and stored new Verification_Keys

### 4.5 Integration Point Interaction Directions and Communication Protocol Overview

The following table summarizes the integration point information between the CAP protocol and the 4 external systems, including interaction directions and communication protocol requirements:

| Integration Point | External System | Interaction Direction | Communication Protocol Requirements |
|-------------------|----------------|----------------------|-------------------------------------|
| 4.1 | iFay_Runtime | Bidirectional | Message-based request-response pattern; persistent connection support (liveness detection); TLS encryption; message format conforming to CAP Schema definition |
| 4.2 | Terminal Operating System | Bidirectional | Local inter-process communication (IPC); synchronous calls + asynchronous event push; conforming to the operating system's native security model |
| 4.3 | Hardware Drivers | Unidirectional (CAP → Hardware Drivers) | Indirect communication through operating system device management interfaces; unified resource description format; hardware-level access locking support |
| 4.4 | Registration_Authority | Unidirectional (RA → Terminal) | TLS/mTLS secure channel; offline pre-provisioning and online update support; key version management and smooth rotation |

**Interaction Direction Descriptions:**

- **Bidirectional**: Request-response or event notification interactions exist in both directions between the CAP protocol layer and the external system, with both parties able to initiate communication
- **Unidirectional**: Information flows in only one direction. In 4.3, the CAP protocol layer issues instructions to hardware drivers through the operating system (no direct communication); in 4.4, Registration_Authority distributes keys and registration information to terminals (terminals passively receive)

**Communication Protocol Design Principles:**

- **Security**: All communication channels involving authorization credential and key transmission must be encrypted to prevent man-in-the-middle attacks and information leakage
- **Platform Adaptability**: Integration with operating systems and hardware drivers uses platform-native interfaces to ensure compatibility across different operating systems
- **Interoperability**: Integration with iFay_Runtime and Registration_Authority uses standardized message formats (based on CAP Schema) to ensure interoperability between different implementations
- **Fault Tolerance**: Communication protocols should support retry and degradation mechanisms so that communication interruptions do not affect the normal operation of CAP protocol core functions (especially offline authorization verification)
