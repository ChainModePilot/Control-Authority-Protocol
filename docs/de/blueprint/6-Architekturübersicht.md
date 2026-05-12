## Architekturübersicht

Das folgende Diagramm zeigt die Gesamtarchitekturposition des CAP-Protokolls im iFay-Ökosystem sowie die Interaktionsbeziehungen mit externen Systemen.

```mermaid
graph TB
    subgraph iFay_System["iFay-Ökosystem"]
        IFAY["iFay / coFay<br/>(Intelligente Agenten)"]
        RT["iFay_Runtime<br/>(Fay-Laufzeitumgebung)"]
    end

    subgraph CAP_Protocol["CAP-Protokollschicht"]
        AD["Authorization_Descriptor<br/>(Offline-Autorisierung – Kernmechanismus)"]
        TT["Trusted_Ticket<br/>(Online-Ticket – Ergänzungsmechanismus)"]
        SM["Session Management<br/>(Sitzungsverwaltung)"]
        HP["Handover_Policy<br/>(Kontrollübergabestrategie)"]
        RAM["Resource_Access_Mode<br/>(Ressourcenzugriffsmodus)"]
    end

    subgraph Terminal["Endgeräteseite"]
        OS["Betriebssystem"]
        HW["Hardwaretreiber"]
        SW["Client-Software"]
    end

    subgraph Trust["Vertrauensinfrastruktur"]
        RA["Registration_Authority<br/>(Registrierungsstelle)"]
        DI["Descriptor_Issuer<br/>(Autorisierungsaussteller)"]
    end

    RT -->|Initiiert Kontrollanfrage| CAP_Protocol
    IFAY -->|Trägt Autorisierungsnachweis| CAP_Protocol
    CAP_Protocol -->|Autorisierungsüberprüfung und Zugriffskontrolle| Terminal
    CAP_Protocol -->|Sitzungslebenszyklusverwaltung| SM
    CAP_Protocol -->|Mehrseitige Kontrollkoordination| HP
    CAP_Protocol -->|Abgestufte Ressourcenzugriffskontrolle| RAM
    RA -->|Verteilt Verification_Key| Terminal
    DI -->|Stellt Authorization_Descriptor aus| IFAY
    OS -->|Meldet Ressourcenstatus| CAP_Protocol
    RT <-->|Sitzungsstatusbenachrichtigung und Heartbeat| CAP_Protocol
```

**Diagrammerläuterung:**

- **iFay-Ökosystem**: iFay/coFay (intelligente Agenten) interagieren über iFay_Runtime mit der CAP-Protokollschicht. iFay_Runtime ist dafür verantwortlich, stellvertretend für den Fay Kontrollanfragen zu initiieren und den für die Aktivitätserkennung erforderlichen Heartbeat-Kanal aufrechtzuerhalten
- **CAP-Protokollschicht**: Die Kernverarbeitungsschicht des Protokolls, die fünf Kernfähigkeiten umfasst – Authorization_Descriptor (Offline-Autorisierung), Trusted_Ticket (Online-Ticket), Session Management (Sitzungsverwaltung), Handover_Policy (Kontrollübergabestrategie) und Resource_Access_Mode (Ressourcenzugriffsmodus)
- **Endgeräteseite**: Umfasst Betriebssystem, Hardwaretreiber und Client-Software und ist die endgültige Ausführungsinstanz der Autorisierungsüberprüfungsergebnisse des CAP-Protokolls. Das Betriebssystem ist für die Durchführung der Ressourcenzugriffskontrolle und die Meldung des Ressourcenstatus an die Protokollschicht verantwortlich. Hardwaretreiber empfangen Kontrollanweisungen indirekt über das Betriebssystem
- **Vertrauensinfrastruktur**: Die Registration_Authority verteilt Verification_Keys an Endgeräte zur Unterstützung der Offline-Autorisierungsüberprüfung, und der Descriptor_Issuer stellt im Auftrag des Autorisierenden Authorization_Descriptors an Fays aus
