## Architecture Overview Diagram

The following diagram illustrates the overall architectural position of the CAP protocol within the iFay ecosystem and its interaction relationships with external systems.

```mermaid
graph TB
    subgraph iFay_System["iFay Ecosystem"]
        IFAY["iFay / coFay<br/>(Intelligent Agents)"]
        RT["iFay_Runtime<br/>(Fay Runtime Environment)"]
    end

    subgraph CAP_Protocol["CAP Protocol Layer"]
        AD["Authorization_Descriptor<br/>(Offline Authorization - Core Mechanism)"]
        TT["Trusted_Ticket<br/>(Online Tickets - Supplementary Mechanism)"]
        SM["Session Management"]
        HP["Handover_Policy<br/>(Control Authority Handover Policy)"]
        RAM["Resource_Access_Mode<br/>(Resource Access Mode)"]
    end

    subgraph Terminal["Terminal Side"]
        OS["Operating System"]
        HW["Hardware Drivers"]
        SW["Client Software"]
    end

    subgraph Trust["Trust Infrastructure"]
        RA["Registration_Authority<br/>(Registration Authority)"]
        DI["Descriptor_Issuer<br/>(Authorization Issuer)"]
    end

    RT -->|Initiates Control Authority Requests| CAP_Protocol
    IFAY -->|Carries Authorization Credentials| CAP_Protocol
    CAP_Protocol -->|Authorization Verification and Access Control| Terminal
    CAP_Protocol -->|Session Lifecycle Management| SM
    CAP_Protocol -->|Multi-party Control Authority Coordination| HP
    CAP_Protocol -->|Resource Access Tiering| RAM
    RA -->|Distributes Verification_Keys| Terminal
    DI -->|Issues Authorization_Descriptors| IFAY
    OS -->|Reports Resource Status| CAP_Protocol
    RT <-->|Session Status Notifications and Heartbeats| CAP_Protocol
```

**Diagram Description:**

- **iFay Ecosystem**: iFay/coFay (intelligent agents) interact with the CAP protocol layer through iFay_Runtime. iFay_Runtime is responsible for initiating control authority requests on behalf of Fays and maintaining the heartbeat channel required for liveness detection
- **CAP Protocol Layer**: The core processing layer of the protocol, containing five core capabilities — Authorization_Descriptor (offline authorization), Trusted_Ticket (online tickets), Session Management, Handover_Policy (control authority handover policy), and Resource_Access_Mode (resource access mode)
- **Terminal Side**: Includes the operating system, hardware drivers, and client software, serving as the ultimate executor of CAP protocol authorization verification results. The operating system is responsible for executing resource access control and reporting resource status to the protocol layer; hardware drivers receive control instructions indirectly through the operating system
- **Trust Infrastructure**: Registration_Authority distributes Verification_Keys to terminals to support offline authorization verification; Descriptor_Issuer, delegated by the authorizer, issues Authorization_Descriptors to Fays
