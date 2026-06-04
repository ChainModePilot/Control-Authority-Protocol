# Chapitre 1 : Architecture et Rôles

Ce chapitre définit les rôles dans le protocole CAP, les relations de confiance entre eux, et les contrats d'interface avec les systèmes externes. Le contenu de ce chapitre fournit la base terminologique partagée et le contexte architectural pour les chapitres suivants.

## 1.1 Rôles du Protocole

Le protocole CAP implique les rôles normatifs suivants. Chaque rôle assume des responsabilités claires dans les interactions du protocole et interagit avec d'autres rôles via des interfaces restreintes.

### 1.1.1 Rôle d'Autorisateur

**Natural_Person** et **Official_Post** sont les sources ultimes d'autorisation. Les autorisateurs ne participent pas directement aux interactions de messages du protocole — les autorisateurs délèguent l'autorité d'émission à `Descriptor_Issuer` via un mécanisme de délégation d'autorisation.

Les autorisateurs DOIVENT prouver leur identité via un mécanisme sécurisé de vérification d'identité. Le mécanisme spécifique de vérification d'identité de l'autorisateur est hors du cadre de cette spécification, mais un tel mécanisme DOIT satisfaire les exigences suivantes :

- Non-répudiation : Les autorisateurs ne peuvent pas répudier les décisions d'autorisation qu'ils ont prises
- Anti-falsification : Les tiers ne peuvent pas falsifier la décision d'autorisation d'un autorisateur

### 1.1.2 Rôle d'Émetteur

**Descriptor_Issuer** est responsable de générer et émettre `Authorization_Descriptor`. **Ticket_Issuer** est responsable d'émettre `Trusted_Ticket`. Une seule entité PEUT assumer les deux rôles d'émetteur simultanément.

Les émetteurs DOIVENT :

1. Émettre des justificatifs uniquement après avoir reçu une autorisation explicite d'un autorisateur légitime
2. Appliquer des signatures numériques aux justificatifs en utilisant les algorithmes de signature définis au Chapitre 8
3. Maintenir des enregistrements d'état des justificatifs émis (incluant au minimum le temps d'émission, la période de validité et l'état actuel)
4. Mettre à jour rapidement l'état de révocation et publier des déclarations de révocation lors de la réception de demandes de révocation

Les émetteurs NE DOIVENT PAS :

1. Émettre des justificatifs sans obtenir d'autorisation
2. Émettre des justificatifs au-delà de la portée d'autorisation de l'autorisateur
3. Réutiliser des identifiants de justificatifs révoqués

### 1.1.3 Rôle de Détenteur

**Fay** (incluant `iFay` et `coFay`) est le détenteur des justificatifs et le demandeur d'accès aux ressources. Fay participe aux interactions du protocole indirectement via `iFay_Runtime` — Fay lui-même n'envoie pas directement de messages de protocole.

Fay DOIT compléter toutes les interactions du protocole avec le terminal via son `iFay_Runtime` associé. Les justificatifs détenus par Fay DOIVENT être soumis au terminal via iFay_Runtime.

### 1.1.4 Rôle de Runtime

**iFay_Runtime** est l'environnement d'exécution pour les instances Fay et assume l'envoi et la réception de messages de protocole. Une seule instance iFay_Runtime PEUT gérer plusieurs instances Fay.

iFay_Runtime DOIT satisfaire les exigences du niveau de conformité runtime définies à la Section 0.5.3.

### 1.1.5 Rôle de Terminal

Le **terminal** (Terminal) est le détenteur de `Terminal_Resource` et l'exécuteur du contrôle d'accès. Le terminal intègre les composants normatifs suivants :

- **Protocol_Engine** : Composant système exécutant la logique principale du protocole CAP
- **Descriptor_Validator** : Composant responsable de vérifier la légitimité et la validité d'`Authorization_Descriptor`
- **Gestionnaire de Session** : Maintient la machine à états Session et le mécanisme Liveness_Detection
- **Liste de révocation locale** : Stocke les identifiants connus de justificatifs révoqués

Le terminal DOIT satisfaire les exigences du niveau de conformité terminal définies à la Section 0.5.1.

### 1.1.6 Rôle d'Infrastructure de Confiance

**Registration_Authority** est responsable de la gestion de l'enregistrement du terminal et de la distribution de `Verification_Key`. Registration_Authority est l'ancre racine de la chaîne de confiance du protocole CAP.

Registration_Authority DOIT :

1. Distribuer Verification_Key uniquement aux terminaux qui ont passé la vérification d'identité
2. Compléter la distribution de clés via des canaux sécurisés (voir Chapitre 8)
3. Fournir des mécanismes de mise à jour et de rotation des clés
4. Maintenir des enregistrements faisant autorité de l'état d'enregistrement du terminal

## 1.2 Chaîne de Confiance

La chaîne de confiance du protocole CAP est propagée de l'autorisateur au `Descriptor_Validator` du terminal, composée de deux chemins de confiance indépendants mais coopérants.

### 1.2.1 Chemin de Confiance d'Autorisation

Le chemin de confiance d'autorisation détermine **si un Fay particulier est autorisé à accéder à une ressource particulière** :

```
Autorisateur (Natural_Person / Official_Post)
    │
    │ ① Délégation d'autorisation (avec preuve non répudiable)
    ▼
Descriptor_Issuer
    │
    │ ② Émet Authorization_Descriptor (avec signature numérique)
    ▼
Fay
    │
    │ ③ Soumet Authorization_Descriptor
    ▼
Descriptor_Validator
```

Le terminal DOIT vérifier l'authenticité de chaque maillon sur le chemin de confiance d'autorisation :

- L'authenticité de l'étape ① est vérifiée par Descriptor_Issuer au moment de l'émission ; le terminal ne participe pas directement
- L'authenticité de l'étape ② est vérifiée par le terminal en vérifiant la signature numérique de Descriptor_Issuer (dépendant du chemin de confiance des clés)
- L'authenticité de l'étape ③ est vérifiée par le terminal en vérifiant la relation de liaison entre Authorization_Descriptor et l'identifiant du Fay soumetteur

### 1.2.2 Chemin de Confiance des Clés

Le chemin de confiance des clés détermine **si le terminal fait confiance à la signature d'un Descriptor_Issuer particulier** :

```
Registration_Authority (ancre de confiance)
    │
    │ ① Distribue Verification_Key (via canal sécurisé)
    ▼
Stockage local de clés du terminal
    │
    │ ② Fournit Verification_Key au Descriptor_Validator
    ▼
Descriptor_Validator
    │
    │ ③ Vérifie la signature d'Authorization_Descriptor avec Verification_Key
    ▼
Validation passée / rejetée
```

Le terminal DOIT :

1. Faire confiance uniquement aux Verification_Keys distribués par Registration_Authority
2. Stocker en sécurité les Verification_Keys (voir Chapitre 8 pour les exigences de stockage de clés)
3. Rejeter la vérification de signature utilisant des clés non distribuées par Registration_Authority

## 1.3 Contrats d'Interface Externes

La couche du protocole CAP interagit avec 4 systèmes externes via des interfaces clairement définies. Cette section définit les contrats normatifs de ces interfaces.

### 1.3.1 Interface avec iFay_Runtime

L'interface entre iFay_Runtime et le Protocol_Engine du terminal est une **interface de messages bidirectionnelle**.

**Direction du message : iFay_Runtime → Protocol_Engine**

| Type de Message | Objectif | Champs Requis |
|---------|------|---------|
| `AuthRequest` | Initie une requête de validation d'autorisation | fay_id, resource_id, access_mode, credential |
| `SessionRelease` | Libère une session proactivement | session_id |
| `Heartbeat` | Battement de cœur de couche d'application | session_id, sequence_number |
| `HandoverResponse` | Répond à une requête de transfert | handover_id, accept |

**Direction du message : Protocol_Engine → iFay_Runtime**

| Type de Message | Objectif | Champs Requis |
|---------|------|---------|
| `AuthResult` | Résultat de validation d'autorisation | request_id, status, session_id (en succès), error_code (en échec) |
| `SessionStateChanged` | Notification de changement d'état de session | session_id, new_state, reason |
| `HandoverRequest` | Notification de requête de transfert | handover_id, target_resource_id, deadline |
| `HeartbeatAck` | Accusé de réception de battement de cœur | session_id, sequence_number |

L'implémentation de l'interface DOIT :

1. Supporter les connexions persistantes pour transporter `Heartbeat` et les messages push asynchrones
2. Supporter la transmission chiffrée TLS
3. Utiliser un format de sérialisation suivant la définition `schema.json`

L'implémentation de l'interface PEUT :

1. Choisir un protocole de transport spécifique (WebSocket, gRPC, HTTP/2, etc.)

### 1.3.2 Interface avec le Système d'Exploitation du Terminal

L'interface entre Protocol_Engine et le système d'exploitation du terminal est une **interface bidirectionnelle locale**.

Le système d'exploitation DOIT exposer les capacités suivantes à Protocol_Engine :

1. Délivrance du contrôle d'accès aux ressources : Protocol_Engine émet des instructions « autoriser Fay X à accéder à la ressource Z en mode Y »
2. Requête de l'état des ressources : Protocol_Engine interroge la disponibilité actuelle et l'état d'occupation des ressources
3. Abonnement aux événements de ressources : Protocol_Engine s'abonne aux événements de changement d'état des ressources (par exemple, déconnexion matérielle, plantage logiciel)

L'implémentation spécifique de l'interface dépend de la plateforme du système d'exploitation (Unix Domain Socket, Named Pipe, D-Bus, etc.). Cette spécification ne prescrit pas l'implémentation spécifique, mais DOIT satisfaire :

1. Les instructions de contrôle d'accès sont exécutées de manière synchrone et retournent les résultats d'exécution
2. Les événements de ressources sont notifiés via push asynchrone
3. L'interface est protégée par le modèle de sécurité natif du système d'exploitation, appelable uniquement par les processus Protocol_Engine autorisés

### 1.3.3 Interface avec les Pilotes Matériels (Indirecte)

Protocol_Engine **NE DOIT PAS** communiquer directement avec les pilotes matériels. Tout le contrôle matériel est transféré via le système d'exploitation.

Chemin de rapport d'état des pilotes matériels à Protocol_Engine :

```
Pilote matériel → Système d'exploitation → Protocol_Engine
```

Les pilotes matériels DEVRAIENT implémenter des mécanismes de verrouillage d'accès au niveau matériel pour s'assurer que même si les contrôles d'accès au niveau du système d'exploitation sont contournés, le matériel lui-même peut toujours rejeter l'accès non autorisé.

### 1.3.4 Interface avec Registration_Authority

L'interface entre Registration_Authority et le terminal est une **interface unidirectionnelle** (RA → terminal).

Registration_Authority DOIT pousser les messages suivants au terminal :

| Type de Message | Objectif | Champs Requis |
|---------|------|---------|
| `VerificationKeyDistribution` | Distribue une nouvelle clé de vérification | key_id, key_material, valid_from, issuer_id |
| `VerificationKeyRevocation` | Révoque une clé de vérification | key_id, revocation_time, reason |
| `RegistrationStatusUpdate` | Met à jour l'état d'enregistrement du terminal | terminal_id, new_status |

Le terminal DOIT :

1. Recevoir les messages sur un canal sécurisé TLS/mTLS
2. Vérifier que la source de signature du message est une Registration_Authority de confiance
3. Retourner une confirmation à Registration_Authority après réception d'une nouvelle clé

Le terminal n'initie pas proactivement de messages de protocole vers Registration_Authority (le flux d'enregistrement initial du terminal n'appartient pas aux interactions normales d'exécution du protocole CAP et est hors du cadre de cette spécification).

## 1.4 Mappage des Rôles et Niveaux de Conformité

| Rôle Implémenté | Niveau de Conformité Requis |
|-----------|---------------------|
| Terminal (incluant Protocol_Engine et Descriptor_Validator) | Niveau de Conformité Terminal (§0.5.1) |
| Descriptor_Issuer ou Ticket_Issuer | Niveau de Conformité Émetteur (§0.5.2) |
| iFay_Runtime | Niveau de Conformité Runtime (§0.5.3) |
| Registration_Authority | Hors du cadre de conformité de cette spécification (le flux d'enregistrement dépasse l'exécution du protocole CAP) |
