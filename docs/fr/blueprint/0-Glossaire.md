# Plan d'architecture CAP

Le Control Authority Protocol (CAP) définit la manière dont les terminaux vérifient qu'un Fay (iFay ou coFay) a été autorisé par son Human Prime à accéder légitimement aux ressources du terminal. Le protocole repose sur un fichier de description d'autorisation hors ligne (Authorization_Descriptor) comme mécanisme principal, complété par un ticket de confiance en ligne (Trusted_Ticket), couvrant la vérification d'autorisation, la gestion des sessions, le transfert de contrôle, les modes d'accès aux ressources et la détection de vivacité. Il fournit un cadre standardisé de contrôle des autorisations permettant aux agents intelligents de l'écosystème iFay de prendre en charge de manière sécurisée les logiciels et matériels des terminaux.

## Glossaire

- **CAP (Control Authority Protocol)** : Protocole de contrôle des autorisations, définissant la manière dont les terminaux vérifient si un Fay a été autorisé à accéder légitimement aux ressources du terminal
- **iFay** : Entité Fay indépendante, agent intelligent rattaché à une Natural_Person
- **coFay** : Entité Fay collaborative, agent intelligent collaboratif rattaché à un Official_Post
- **Natural_Person** : Personne physique, l'individu humain auquel un iFay est rattaché, source ultime de l'autorisation
- **Official_Post** : Poste officiel, le poste organisationnel auquel un coFay est rattaché, source ultime de l'autorisation
- **iFay_Runtime** : Environnement d'exécution iFay, responsable de la gestion du cycle de vie des instances Fay, de l'ordonnancement et de l'initiation des demandes de contrôle
- **Authorization_Descriptor** : Fichier de description d'autorisation, un fichier chiffré stocké localement sur le terminal décrivant la portée des ressources, les types de permissions et la période de validité pour lesquels un Fay est autorisé, constituant le mécanisme principal de l'autorisation hors ligne
- **Trusted_Ticket** : Ticket de confiance, un justificatif en ligne émis par un Ticket_Issuer dans les scénarios connectés, servant de mécanisme complémentaire à l'autorisation hors ligne
- **Terminal_Resource** : Ressource du terminal, dispositifs matériels ou logiciels clients sur le terminal pouvant être consultés et exploités
- **Descriptor_Issuer** : Émetteur de descripteurs d'autorisation, entité de confiance autorisée par une Natural_Person ou un Official_Post à générer et émettre des Authorization_Descriptors
- **Descriptor_Validator** : Validateur de descripteurs, composant côté terminal responsable de la vérification de la légitimité et de la validité des Authorization_Descriptors
- **Registration_Authority** : Autorité d'enregistrement, entité de confiance responsable de la gestion de l'enregistrement du matériel, des logiciels et des systèmes d'exploitation des terminaux, ainsi que de la distribution des Verification_Keys
- **Verification_Key** : Clé de vérification, clé obtenue par le terminal lors de l'enregistrement, utilisée pour vérifier la signature numérique des Authorization_Descriptors
- **Protocol_Engine** : Moteur de protocole, composant système exécutant la logique principale du Control Authority Protocol
- **Session** : Session de contrôle, cycle de vie complet depuis l'approbation de la vérification d'autorisation jusqu'à la fin de l'accès
- **Handover_Policy** : Politique de transfert de contrôle, définissant les règles de décision pour le transfert de contrôle entre plusieurs Fays ou entre Fays et humains
- **Resource_Access_Mode** : Mode d'accès aux ressources, mécanisme de gestion hiérarchisée de l'accès aux ressources par type d'opération (modèle de verrou lecture-écriture)
- **Liveness_Detection** : Détection de vivacité, détection de l'activité d'une session Fay par la combinaison de connexions persistantes et de battements de cœur au niveau applicatif
- **Capability_Matrix** : Matrice de capacités, description structurée des capacités principales du protocole CAP dans le plan d'architecture
- **Audit_Logger** : Enregistreur d'audit, composant responsable de l'enregistrement de toutes les opérations de vérification d'autorisation et d'accès aux ressources
