## Vue d'ensemble de l'architecture

Le diagramme suivant présente la position architecturale globale du protocole CAP dans l'écosystème iFay, ainsi que les relations d'interaction avec les systèmes externes.

```mermaid
graph TB
    subgraph iFay_System["Écosystème iFay"]
        IFAY["iFay / coFay<br/>(Agents intelligents)"]
        RT["iFay_Runtime<br/>(Environnement d'exécution Fay)"]
    end

    subgraph CAP_Protocol["Couche du protocole CAP"]
        AD["Authorization_Descriptor<br/>(Autorisation hors ligne - Mécanisme principal)"]
        TT["Trusted_Ticket<br/>(Ticket en ligne - Mécanisme complémentaire)"]
        SM["Session Management<br/>(Gestion des sessions)"]
        HP["Handover_Policy<br/>(Politique de transfert de contrôle)"]
        RAM["Resource_Access_Mode<br/>(Mode d'accès aux ressources)"]
    end

    subgraph Terminal["Côté terminal"]
        OS["Système d'exploitation"]
        HW["Pilotes matériels"]
        SW["Logiciels clients"]
    end

    subgraph Trust["Infrastructure de confiance"]
        RA["Registration_Authority<br/>(Autorité d'enregistrement)"]
        DI["Descriptor_Issuer<br/>(Émetteur d'autorisations)"]
    end

    RT -->|Initie les demandes de contrôle| CAP_Protocol
    IFAY -->|Présente les justificatifs d'autorisation| CAP_Protocol
    CAP_Protocol -->|Vérification d'autorisation et contrôle d'accès| Terminal
    CAP_Protocol -->|Gestion du cycle de vie des sessions| SM
    CAP_Protocol -->|Coordination du contrôle multipartite| HP
    CAP_Protocol -->|Hiérarchisation de l'accès aux ressources| RAM
    RA -->|Distribue les Verification_Keys| Terminal
    DI -->|Émet les Authorization_Descriptors| IFAY
    OS -->|Rapporte l'état des ressources| CAP_Protocol
    RT <-->|Notifications d'état de session et battements de cœur| CAP_Protocol
```

**Légende du diagramme :**

- **Écosystème iFay** : iFay/coFay (agents intelligents) interagissent avec la couche du protocole CAP via iFay_Runtime. iFay_Runtime est responsable d'initier les demandes de contrôle au nom du Fay et de maintenir le canal de battements de cœur nécessaire à la détection de vivacité
- **Couche du protocole CAP** : Couche de traitement principale du protocole, comprenant cinq capacités principales — Authorization_Descriptor (autorisation hors ligne), Trusted_Ticket (ticket en ligne), Session Management (gestion des sessions), Handover_Policy (politique de transfert de contrôle) et Resource_Access_Mode (mode d'accès aux ressources)
- **Côté terminal** : Comprend le système d'exploitation, les pilotes matériels et les logiciels clients, constituant l'exécutant final des résultats de vérification d'autorisation du protocole CAP. Le système d'exploitation est responsable de l'exécution du contrôle d'accès aux ressources et du rapport de l'état des ressources à la couche du protocole, les pilotes matériels recevant indirectement les instructions de contrôle via le système d'exploitation
- **Infrastructure de confiance** : La Registration_Authority distribue les Verification_Keys aux terminaux pour soutenir la vérification d'autorisation hors ligne, le Descriptor_Issuer émet les Authorization_Descriptors aux Fays sur délégation de l'autorisant
