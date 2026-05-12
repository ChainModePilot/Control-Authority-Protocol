## Chapitre 4 : Points d'intégration externes

Ce chapitre décrit les points d'intégration du protocole CAP avec les systèmes externes, précisant les limites de responsabilité, la direction d'interaction et les exigences de protocole de communication pour chaque point d'intégration. Le protocole CAP ne fonctionne pas de manière isolée ; il doit collaborer avec iFay_Runtime, le système d'exploitation du terminal, les pilotes matériels et Registration_Authority, entre autres systèmes externes, pour accomplir conjointement la gestion des autorisations de contrôle du terminal. Comprendre les frontières d'interface et les modes d'interaction de ces points d'intégration est un prérequis pour que les intégrateurs système implémentent correctement le protocole CAP.

### 4.1 Intégration avec iFay_Runtime

iFay_Runtime est l'environnement d'exécution des Fays (iFay ou coFay), responsable de la gestion du cycle de vie et de l'ordonnancement des instances Fay. Du point de vue du protocole CAP, iFay_Runtime est l'initiateur des demandes de contrôle — lorsqu'un Fay a besoin d'accéder aux ressources du terminal, iFay_Runtime initie les demandes de vérification d'autorisation auprès de la couche du protocole CAP en son nom.

**Responsabilités d'intégration :**

- **Initiation des demandes de contrôle** : iFay_Runtime soumet les demandes de vérification d'autorisation au Protocol_Engine au nom du Fay, les demandes contenant l'identifiant du Fay, la Terminal_Resource cible et les justificatifs d'autorisation (Authorization_Descriptor ou Trusted_Ticket)
- **Gestion du cycle de vie des Fays** : iFay_Runtime est responsable du démarrage, de la mise en pause, de la reprise et de la terminaison des instances Fay. Lorsqu'une instance Fay est terminée, iFay_Runtime notifie le Protocol_Engine de libérer toutes les Sessions actives détenues par ce Fay
- **Maintien du canal de détection de vivacité** : iFay_Runtime est responsable du maintien de la connexion persistante entre le Fay et le terminal, et de l'envoi des messages de battement de cœur au niveau applicatif selon l'intervalle configuré, supportant le fonctionnement normal du mécanisme Liveness_Detection
- **Réception des notifications d'état de session** : Le Protocol_Engine notifie iFay_Runtime des changements d'état des Sessions (création réussie, demande de transfert, terminaison forcée, etc.), et iFay_Runtime transmet ces notifications à l'instance Fay correspondante

**Direction d'interaction :** Bidirectionnelle (Bidirectional)

iFay_Runtime initie des demandes vers la couche du protocole CAP (vérification d'autorisation, libération de session, envoi de battements de cœur), et la couche du protocole CAP retourne des réponses et envoie des notifications à iFay_Runtime (résultats de vérification, changements d'état de session, demandes de transfert).

**Exigences de protocole de communication :**

- La communication entre iFay_Runtime et Protocol_Engine adopte un modèle requête-réponse basé sur les messages, le format des messages suivant la définition du Schema du protocole CAP
- Le canal de détection de vivacité nécessite la prise en charge des connexions persistantes, pour permettre les battements de cœur au niveau applicatif et les notifications d'état en temps réel
- Le canal de communication doit prendre en charge le chiffrement TLS, garantissant la confidentialité et l'intégrité des justificatifs d'autorisation et des informations de session pendant le transport
- Le format de sérialisation des messages doit être cohérent avec le fichier de définition de Schema (schema.json), garantissant l'interopérabilité entre les différentes implémentations

### 4.2 Intégration avec le système d'exploitation du terminal

Le système d'exploitation du terminal est le gestionnaire des Terminal_Resources, responsable de la gestion unifiée des ressources logicielles et matérielles sur le terminal. Le protocole CAP, via son intégration avec le système d'exploitation, traduit les résultats de la vérification d'autorisation en contrôle d'accès effectif aux ressources — le système d'exploitation, selon les instructions du Protocol_Engine, autorise ou refuse l'accès d'un Fay à des ressources spécifiques.

**Responsabilités d'intégration :**

- **Exécution du contrôle d'accès aux ressources** : Le système d'exploitation, selon les instructions de contrôle d'accès émises par le Protocol_Engine, autorise ou bloque au niveau système l'accès d'un Fay aux Terminal_Resources. Cela inclut le contrôle d'accès au système de fichiers, la gestion des permissions de processus, les autorisations d'accès aux dispositifs, etc.
- **Rapport de l'état des ressources** : Le système d'exploitation rapporte au Protocol_Engine l'état actuel des Terminal_Resources (disponible, occupé, indisponible), permettant au Protocol_Engine de prendre des décisions précises d'allocation des ressources
- **Isolation de l'environnement d'exécution** : Le système d'exploitation fournit une isolation au niveau processus ou bac à sable pour les opérations des différents Fays, garantissant que les opérations d'un Fay n'affectent pas l'accès aux ressources des autres Fays ou des utilisateurs humains
- **Transmission des événements de ressources** : Lorsque l'état d'une Terminal_Resource change (par exemple, déconnexion matérielle, plantage logiciel), le système d'exploitation transmet la notification d'événement au Protocol_Engine, déclenchant les processus appropriés de terminaison de Session ou de récupération de ressources

**Direction d'interaction :** Bidirectionnelle (Bidirectional)

La couche du protocole CAP émet des instructions de contrôle d'accès et des demandes de consultation de ressources vers le système d'exploitation, et le système d'exploitation rapporte l'état des ressources et transmet les événements de ressources à la couche du protocole CAP.

**Exigences de protocole de communication :**

- La communication entre la couche du protocole CAP et le système d'exploitation adopte un mécanisme de communication inter-processus (IPC) local, le mode spécifique dépendant de la plateforme du système d'exploitation (par exemple Unix Domain Socket, Named Pipe, D-Bus, etc.)
- Les instructions de contrôle d'accès doivent être exécutées en mode d'appel synchrone, garantissant que le Protocol_Engine peut connaître immédiatement le résultat de l'exécution de l'instruction
- Les notifications d'événements de ressources adoptent un mode de notification asynchrone, le système d'exploitation notifiant proactivement le Protocol_Engine lors des changements d'état des ressources
- L'interface de communication doit respecter le modèle de sécurité natif du système d'exploitation, garantissant que seul le processus Protocol_Engine autorisé peut émettre des instructions de contrôle d'accès

### 4.3 Intégration avec les pilotes matériels

Les pilotes matériels sont l'interface d'accès aux Terminal_Resources de type matériel (tels que caméras, microphones, modules GPS, dispositifs de stockage, etc.). Le protocole CAP, via son intégration avec les pilotes matériels, réalise une gestion fine du contrôle des ressources matérielles. Les pilotes matériels se situent sous le système d'exploitation dans le système d'intégration du protocole CAP, fournissant les capacités d'accès de bas niveau aux ressources matérielles.

**Responsabilités d'intégration :**

- **Fourniture de l'interface d'accès aux ressources matérielles** : Les pilotes matériels exposent à la couche du protocole CAP la description des capacités et l'interface d'opération des ressources matérielles, permettant au Protocol_Engine de connaître les modes d'accès supportés par les ressources matérielles (read, write, execute, configure)
- **Exécution du contrôle d'accès au niveau matériel** : Les pilotes matériels, selon les instructions de contrôle transmises par le Protocol_Engine via le système d'exploitation, exécutent au niveau matériel l'autorisation ou le refus d'accès. Par exemple, autoriser ou interdire à un Fay d'activer la caméra, d'accéder au microphone
- **Rapport de l'état du matériel** : Les pilotes matériels rapportent aux couches supérieures l'état de connexion, l'état de fonctionnement et les informations d'anomalie des dispositifs matériels, ces informations étant finalement transmises au Protocol_Engine pour les décisions de gestion des Sessions
- **Prise en charge du contrôle exclusif des ressources matérielles** : Pour les ressources matérielles nécessitant un accès exclusif (comme la sortie de flux vidéo de la caméra), les pilotes matériels garantissent au niveau bas qu'une seule partie contrôlante peut utiliser exclusivement la ressource à un instant donné

**Direction d'interaction :** Unidirectionnelle (Unidirectional) — Couche du protocole CAP → Pilotes matériels

La couche du protocole CAP émet des instructions de contrôle vers les pilotes matériels via le système d'exploitation, et les informations d'état des pilotes matériels remontent via la couche de gestion des dispositifs du système d'exploitation. La couche du protocole CAP n'établit pas de canal de communication direct avec les pilotes matériels, mais interagit indirectement via le système d'exploitation comme intermédiaire. Le chemin de remontée de l'état matériel est : pilotes matériels → système d'exploitation → Protocol_Engine, faisant partie du point d'intégration avec le système d'exploitation (4.2).

**Exigences de protocole de communication :**

- La couche du protocole CAP ne communique pas directement avec les pilotes matériels ; toutes les interactions sont effectuées indirectement via l'interface de gestion des dispositifs du système d'exploitation (par exemple DeviceIoControl, ioctl, sysfs, etc.)
- La description des capacités des ressources matérielles doit suivre un format de description de ressources unifié, permettant au Protocol_Engine de gérer de manière cohérente différents types de ressources matérielles
- Les pilotes matériels doivent prendre en charge l'interface standard de contrôle d'accès aux dispositifs fournie par le système d'exploitation, garantissant que les instructions de contrôle d'accès du protocole CAP peuvent être appliquées au niveau matériel
- Pour les ressources matérielles hautement sensibles (telles que caméras, microphones), les pilotes matériels doivent prendre en charge un mécanisme de verrouillage d'accès au niveau matériel, empêchant le contournement du contrôle d'accès au niveau du système d'exploitation

### 4.4 Intégration avec Registration_Authority

La Registration_Authority (autorité d'enregistrement) est un composant essentiel de l'infrastructure de confiance du protocole CAP, responsable de la gestion de l'enregistrement du matériel, des logiciels et des systèmes d'exploitation des terminaux, ainsi que de la distribution des clés de vérification (Verification_Key) aux terminaux. La Verification_Key obtenue par le terminal auprès de la Registration_Authority est le point d'ancrage de confiance de la vérification d'autorisation hors ligne — sans Verification_Key légitime, le terminal ne peut pas vérifier la signature numérique de l'Authorization_Descriptor.

**Responsabilités d'intégration :**

- **Enregistrement du terminal** : Les terminaux (incluant matériel, système d'exploitation et logiciels clients) complètent leur enregistrement auprès de la Registration_Authority, obtenant un identifiant de terminal unique et les informations de configuration initiales
- **Distribution des Verification_Keys** : La Registration_Authority distribue les Verification_Keys aux terminaux enregistrés, les terminaux utilisant ces clés pour vérifier la signature numérique des Authorization_Descriptors. Le processus de distribution des clés doit être effectué via un canal sécurisé, empêchant la falsification ou le vol des clés
- **Mise à jour et rotation des clés** : Lorsqu'une Verification_Key doit être mise à jour (par exemple, expiration de la clé, changement d'émetteur), la Registration_Authority est responsable de la distribution de la nouvelle clé aux terminaux et de la coordination d'une transition fluide pendant le processus de rotation des clés
- **Gestion de l'état d'enregistrement** : La Registration_Authority maintient l'état d'enregistrement des terminaux, incluant enregistré, suspendu et désinscrit. L'état d'enregistrement du terminal affecte sa capacité à recevoir les mises à jour de Verification_Key et à participer au processus de vérification d'autorisation du protocole CAP

**Direction d'interaction :** Unidirectionnelle (Unidirectional) — Registration_Authority → Terminal

La Registration_Authority distribue les Verification_Keys et les informations d'enregistrement aux terminaux, les terminaux les recevant passivement. Les terminaux n'initient pas de demandes de vérification d'autorisation ni ne rapportent leur état de fonctionnement à la Registration_Authority — la vérification d'autorisation du terminal est entièrement effectuée localement (en utilisant les Verification_Keys déjà distribuées), sans nécessiter de communication en temps réel avec la Registration_Authority. Les demandes d'enregistrement initiées par le terminal auprès de la Registration_Authority relèvent d'un processus d'initialisation ponctuel et ne font pas partie des interactions courantes du protocole CAP en fonctionnement.

**Exigences de protocole de communication :**

- La communication entre la Registration_Authority et les terminaux doit être effectuée via un canal sécurisé (par exemple TLS/mTLS), garantissant la confidentialité et l'intégrité des Verification_Keys pendant le transport
- Le protocole de distribution des clés doit prendre en charge deux modes : le pré-chargement hors ligne et la mise à jour en ligne. Les terminaux peuvent être pré-chargés avec une Verification_Key initiale en usine, puis recevoir les mises à jour de clés via un canal en ligne
- Les messages de mise à jour des clés doivent inclure un numéro de version et une date d'entrée en vigueur, prenant en charge une période de transition fluide entre anciennes et nouvelles clés, évitant les interruptions de vérification d'autorisation causées par la rotation des clés
- La Registration_Authority doit fournir un mécanisme de confirmation de la distribution des clés, garantissant que le terminal a bien reçu et stocké la nouvelle Verification_Key

### 4.5 Vue d'ensemble des directions d'interaction et protocoles de communication des points d'intégration

Le tableau suivant résume les informations des points d'intégration du protocole CAP avec les 4 systèmes externes, incluant la direction d'interaction et les exigences de protocole de communication :

| Point d'intégration | Système externe | Direction d'interaction | Exigences de protocole de communication |
|---------------------|----------------|----------------------|----------------------------------------|
| 4.1 | iFay_Runtime | Bidirectionnelle (Bidirectional) | Modèle requête-réponse basé sur les messages ; prise en charge des connexions persistantes (détection de vivacité) ; chiffrement TLS ; format des messages conforme à la définition du Schema CAP |
| 4.2 | Système d'exploitation du terminal | Bidirectionnelle (Bidirectional) | Communication inter-processus (IPC) locale ; appel synchrone + notification d'événements asynchrone ; respect du modèle de sécurité natif du système d'exploitation |
| 4.3 | Pilotes matériels | Unidirectionnelle (CAP → Pilotes matériels) | Communication indirecte via l'interface de gestion des dispositifs du système d'exploitation ; format de description de ressources unifié ; prise en charge du verrouillage d'accès au niveau matériel |
| 4.4 | Registration_Authority | Unidirectionnelle (RA → Terminal) | Canal sécurisé TLS/mTLS ; prise en charge du pré-chargement hors ligne et de la mise à jour en ligne ; gestion des versions de clés et rotation fluide |

**Explication des directions d'interaction :**

- **Bidirectionnelle (Bidirectional)** : Il existe des interactions bidirectionnelles de type requête-réponse ou notification d'événements entre la couche du protocole CAP et le système externe, les deux parties pouvant initier la communication de manière proactive
- **Unidirectionnelle (Unidirectional)** : Le flux d'information ne circule que d'une partie vers l'autre. Dans le point 4.3, la couche du protocole CAP émet des instructions vers les pilotes matériels via le système d'exploitation (sans communication directe) ; dans le point 4.4, la Registration_Authority distribue les clés et les informations d'enregistrement aux terminaux (les terminaux les reçoivent passivement)

**Principes de conception des protocoles de communication :**

- **Sécurité** : Tous les canaux de communication impliquant le transport de justificatifs d'autorisation et de clés doivent être chiffrés, empêchant les attaques de l'homme du milieu et les fuites d'informations
- **Adaptabilité aux plateformes** : L'intégration avec les systèmes d'exploitation et les pilotes matériels adopte les interfaces natives de la plateforme, garantissant la compatibilité sur différents systèmes d'exploitation
- **Interopérabilité** : L'intégration avec iFay_Runtime et Registration_Authority adopte des formats de messages standardisés (basés sur le Schema CAP), garantissant l'interopérabilité entre différentes implémentations
- **Tolérance aux pannes** : Les protocoles de communication doivent prendre en charge les mécanismes de nouvelle tentative et de dégradation, garantissant que les interruptions de communication n'affectent pas le fonctionnement normal des fonctionnalités principales du protocole CAP (en particulier la vérification d'autorisation hors ligne)
