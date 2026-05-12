# CAP Architecture Blueprint

Control Authority Protocol (CAP) defines how terminal devices verify that a Fay (iFay or coFay) has been authorized by its human host to legitimately access terminal resources. The protocol uses an offline Authorization_Descriptor as its core mechanism, supplemented by an online Trusted_Ticket, covering authorization verification, session management, control authority handover, resource access modes, and liveness detection. It provides a standardized control authority framework for intelligent agents in the iFay ecosystem to securely take over terminal software and hardware.

## Glossary

- **CAP (Control Authority Protocol)**: Control Authority Protocol, defining how terminal devices verify whether a Fay has been authorized to legitimately access terminal resources
- **iFay**: Independent Fay entity, an intelligent agent attached to a Natural_Person
- **coFay**: Collaborative Fay entity, a collaborative intelligent agent attached to an Official_Post
- **Natural_Person**: A natural person, the human individual to which an iFay is attached, and the ultimate source of authorization
- **Official_Post**: An official post, the organizational position to which a coFay is attached, and the ultimate source of authorization
- **iFay_Runtime**: iFay runtime environment, responsible for Fay instance lifecycle management, scheduling, and initiating control authority requests
- **Authorization_Descriptor**: Authorization descriptor file, an encrypted file stored locally on the terminal that describes the scope of resources, permission types, and validity period for which a Fay is authorized, serving as the core mechanism for offline authorization
- **Trusted_Ticket**: Trusted ticket, an online credential issued by a Ticket_Issuer in connected scenarios, serving as a supplementary mechanism to offline authorization
- **Terminal_Resource**: Terminal resource, hardware devices or client software on the terminal that can be accessed and operated
- **Descriptor_Issuer**: Authorization descriptor issuer, a trusted entity authorized by a Natural_Person or Official_Post to generate and issue Authorization_Descriptors
- **Descriptor_Validator**: Descriptor validator, a component on the terminal side responsible for verifying the legitimacy and validity of Authorization_Descriptors
- **Registration_Authority**: Registration authority, a trusted entity responsible for managing terminal hardware, software, and operating system registration and distributing Verification_Keys
- **Verification_Key**: Verification key, a key obtained by the terminal through registration, used to verify the digital signature of Authorization_Descriptors
- **Protocol_Engine**: Protocol engine, the system component that executes the core logic of the Control Authority Protocol
- **Session**: Control session, the complete lifecycle from authorization verification approval to access termination
- **Handover_Policy**: Control authority handover policy, defining the decision rules for control authority transfer between multiple Fays or between Fays and humans
- **Resource_Access_Mode**: Resource access mode, a mechanism for tiered management of resource access by operation type (read-write lock model)
- **Liveness_Detection**: Liveness detection, detecting whether a Fay session is still active through a combination of persistent connections and application-layer heartbeats
- **Capability_Matrix**: Capability matrix, a structured description of CAP protocol core capabilities in the blueprint
- **Audit_Logger**: Audit logger, a component responsible for recording all authorization verification and resource access operations
