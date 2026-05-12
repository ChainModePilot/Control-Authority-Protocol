## Chapitre 2 : Matrice des capacités principales

Ce chapitre fournit une description au niveau stratégique des cinq capacités principales du protocole CAP, couvrant l'autorisation hors ligne, les tickets en ligne, la gestion des sessions, les politiques de transfert de contrôle et les modes d'accès aux ressources. Chaque capacité est détaillée selon les dimensions de la motivation de conception, du cycle de vie, des mécanismes clés et de la configuration des politiques, fournissant des orientations de haut niveau pour l'élaboration ultérieure des spécifications techniques du protocole.

### 2.1 Autorisation hors ligne (Authorization_Descriptor)

#### Motivation de conception

L'Authorization_Descriptor est le mécanisme d'autorisation principal du protocole CAP. Sa motivation de conception découle d'un jugement fondamental : **un terminal ne devrait pas être complètement privé des droits de prise de contrôle d'un Fay lorsqu'il est hors ligne, car le Fay peut avoir été préalablement autorisé par son hôte humain**.

Dans les scénarios réels, les terminaux se trouvent fréquemment dans des états où le réseau est indisponible ou instable — mode avion, espaces souterrains, zones isolées, pannes réseau, etc. Si la vérification d'autorisation reposait entièrement sur des services en ligne, dès l'interruption du réseau, un Fay perdrait immédiatement tout contrôle sur les ressources du terminal, même si l'hôte humain (Natural_Person) ou le poste officiel (Official_Post) avait précédemment accordé une autorisation explicite. Par exemple, l'iFay d'un photographe animalier est en train de l'aider à piloter un drone pour des prises de vue aériennes lorsque le signal du téléphone est perdu après avoir volé dans une vallée montagneuse — sans mécanisme d'autorisation hors ligne, l'iFay perdrait immédiatement le contrôle du drone, risquant de le faire s'écraser. Une telle conception rendrait la disponibilité d'un Fay fortement dépendante de l'état du réseau, en contradiction avec la vision de l'écosystème iFay d'une « prise de contrôle transparente des terminaux par les agents intelligents ».

L'Authorization_Descriptor persiste la relation d'autorisation localement sur le terminal sous forme de fichier chiffré, permettant au terminal d'effectuer indépendamment la vérification d'autorisation en mode hors ligne. Ce fichier contient la portée des ressources, les types de permissions et la période de validité pour lesquels le Fay est autorisé. Le Descriptor_Validator côté terminal peut vérifier sa légitimité et sa validité sans nécessiter de connectivité réseau.

#### Aperçu du cycle de vie

Le cycle de vie d'un Authorization_Descriptor comprend les 5 phases suivantes :

1. **Émission (Issuance)** : Le Descriptor_Issuer génère et émet un Authorization_Descriptor après avoir obtenu l'autorisation explicite de l'autorisant (Natural_Person ou Official_Post). Le processus d'émission comprend la détermination de la portée de l'autorisation, la définition de la période de validité, la génération de signatures numériques et d'autres étapes. Une fois l'émission terminée, l'Authorization_Descriptor est transmis à l'entité Fay correspondante. Par exemple, l'utilisateur Zhang San autorise son iFay via l'interface de gestion des autorisations à utiliser l'appareil photo et le microphone de l'ordinateur portable pendant les 30 prochains jours, et le Descriptor_Issuer génère un Authorization_Descriptor contenant ces permissions et le transmet à l'iFay.

2. **Stockage local (Local Storage)** : Le Fay soumet l'Authorization_Descriptor obtenu au terminal cible, qui le stocke sous forme chiffrée dans une zone de stockage sécurisée locale. Le stockage local garantit que le terminal peut accéder aux informations d'autorisation en mode hors ligne, constituant la base physique du mécanisme d'autorisation hors ligne. Par exemple, l'iFay soumet le fichier d'autorisation à l'ordinateur portable de Zhang San, qui le chiffre et le stocke dans la puce de sécurité locale.

3. **Validation (Validation)** : Lorsqu'un Fay demande l'accès aux ressources du terminal, le Descriptor_Validator côté terminal effectue la validation de l'Authorization_Descriptor stocké localement. La validation comprend : la légitimité de la signature numérique (vérifiée à l'aide de la Verification_Key), l'expiration de la période de validité, la couverture de la portée d'autorisation pour les ressources et types d'opérations demandés, et la vérification de la révocation éventuelle. Par exemple, lorsque l'iFay demande à activer l'appareil photo, le terminal vérifie si la signature du fichier d'autorisation est légitime, s'il est dans la période de validité de 30 jours et s'il inclut la permission d'utiliser l'appareil photo.

4. **Révocation (Revocation)** : Lorsque l'autorisant décide de retirer l'autorisation, une déclaration de révocation est émise pour invalider l'Authorization_Descriptor correspondant. Les déclarations de révocation sont distribuées aux terminaux par plusieurs canaux (voir « Stratégie de distribution des déclarations de révocation » ci-dessous). Lors des validations ultérieures, le terminal rejettera les Authorization_Descriptors qui ont été révoqués. Par exemple, Zhang San constate que le comportement de l'iFay ne correspond pas à ses attentes et révoque immédiatement son autorisation d'utiliser l'appareil photo ; le terminal rejettera la demande d'accès à l'appareil photo de l'iFay lors de la prochaine validation.

5. **Renouvellement (Renewal)** : Lorsqu'un Authorization_Descriptor est sur le point d'expirer ou que la portée de l'autorisation doit être ajustée, le Descriptor_Issuer émet un nouvel Authorization_Descriptor pour remplacer l'ancienne version. Le processus de renouvellement est essentiellement une nouvelle émission ; l'ancienne version devient automatiquement invalide après l'entrée en vigueur de la nouvelle version. Par exemple, la période de validité de 30 jours est sur le point d'expirer, Zhang San décide de renouveler et d'autoriser en plus l'iFay à utiliser les dispositifs de stockage ; le Descriptor_Issuer émet un nouveau fichier d'autorisation pour remplacer l'ancienne version.

#### Modèle de chaîne de confiance

La fiabilité de l'autorisation hors ligne repose sur une chaîne de confiance complète. Le chemin de propagation de la confiance de l'autorisant au validateur côté terminal est le suivant :

```
Autorisant (Natural_Person / Official_Post)
    │
    │ Délégation d'autorisation
    ▼
Descriptor_Issuer (Émetteur de descripteurs d'autorisation)
    │
    │ Émet l'Authorization_Descriptor (avec signature numérique)
    ▼
Fay (iFay / coFay)
    │
    │ Soumet l'Authorization_Descriptor
    ▼
Descriptor_Validator côté terminal
    │
    │ Vérifie la signature à l'aide de la Verification_Key
    ▼
Résultat de la validation (Approuvé / Rejeté)
```

Maillons clés de la chaîne de confiance :

- **Autorisant → Descriptor_Issuer** : L'autorisant délègue le pouvoir d'émission au Descriptor_Issuer via un mécanisme sécurisé de délégation d'autorisation, garantissant que seuls les émetteurs autorisés peuvent générer des Authorization_Descriptors légitimes
- **Descriptor_Issuer → Fay** : Le Descriptor_Issuer signe numériquement l'Authorization_Descriptor à l'aide de sa clé privée, et le Fay reçoit le fichier d'autorisation signé
- **Fay → Descriptor_Validator** : Le Fay soumet l'Authorization_Descriptor au terminal, et le Descriptor_Validator côté terminal vérifie la légitimité de la signature à l'aide de la Verification_Key distribuée par la Registration_Authority
- **Registration_Authority → Terminal** : La Registration_Authority, en tant qu'infrastructure de confiance, est responsable de la distribution des Verification_Keys aux terminaux, garantissant que les terminaux peuvent vérifier la source de signature des Authorization_Descriptors

#### Stratégie de distribution des déclarations de révocation

Dans les scénarios hors ligne, la distribution rapide des déclarations de révocation constitue un défi majeur. Les terminaux doivent obtenir les informations de révocation les plus récentes possible sans pouvoir interroger les services de révocation en ligne en temps réel. Le protocole CAP adopte les stratégies de distribution suivantes :

- **Liste de révocation locale (Local Revocation List)** : Le terminal maintient une liste de révocation locale enregistrant les identifiants des Authorization_Descriptors révoqués connus. Chaque fois que le terminal se connecte au réseau, il synchronise automatiquement la dernière liste de révocation depuis le service de révocation
- **Synchronisation proactive lors de la connexion** : Lorsque le terminal passe de l'état hors ligne à l'état en ligne, il priorise la synchronisation de la liste de révocation pour garantir que les informations de révocation locales sont aussi à jour que possible
- **Période de validité comme filet de sécurité** : Les Authorization_Descriptors portent leur propre période de validité. Même si une déclaration de révocation n'arrive pas à temps, les Authorization_Descriptors expirés deviennent automatiquement invalides, limitant la fenêtre d'impact maximale des retards de révocation
- **Prise d'effet à la prochaine validation** : Le terminal vérifie la liste de révocation locale lors de chaque validation d'Authorization_Descriptor. Dès qu'une déclaration de révocation atteint le terminal, les demandes de validation ultérieures rejetteront immédiatement l'autorisation révoquée

### 2.2 Tickets en ligne (Trusted_Ticket)

#### Positionnement et cas d'utilisation

Le Trusted_Ticket est le mécanisme complémentaire d'autorisation en ligne du protocole CAP dans les environnements connectés. Son positionnement principal est : **fournir des capacités d'émission d'autorisation en temps réel et de consultation de l'état de révocation dans les environnements connectés**, compensant les insuffisances de l'autorisation hors ligne en termes de réactivité et de vitesse de réponse aux révocations.

Les cas d'utilisation typiques du Trusted_Ticket comprennent :

- **Autorisation temporaire** : Lorsqu'un hôte humain a besoin d'accorder temporairement à un Fay l'accès à une ressource spécifique, un Trusted_Ticket peut être émis instantanément via les services en ligne sans passer par le processus complet d'émission d'Authorization_Descriptor. Par exemple, un utilisateur a temporairement besoin que son iFay l'aide à imprimer un document ; d'un simple appui sur son téléphone, le service en ligne émet instantanément un ticket temporaire limité à l'utilisation de l'imprimante, sans passer par le processus complet d'émission d'autorisation hors ligne
- **Vérification de révocation en temps réel** : En état connecté, le terminal peut interroger en temps réel l'état de révocation des autorisations via le mécanisme Trusted_Ticket, obtenant des informations de révocation plus récentes que la liste de révocation locale. Par exemple, un administrateur d'entreprise vient de révoquer le contrôle d'un coFay sur les équipements de la salle de conférence ; le terminal connecté peut immédiatement prendre connaissance de cette révocation via le mécanisme de ticket en ligne, plutôt que d'attendre la prochaine synchronisation de la liste de révocation locale
- **Ajustement dynamique des permissions** : Dans les environnements connectés, la portée de l'autorisation peut être ajustée dynamiquement via les Trusted_Tickets sans réémettre un Authorization_Descriptor. Par exemple, un iFay n'avait initialement que la permission de lire des fichiers ; l'utilisateur élève temporairement sa permission en écriture depuis son téléphone, prenant effet immédiatement via un ticket en ligne

#### Relation avec l'Authorization_Descriptor

La relation entre le Trusted_Ticket et l'Authorization_Descriptor est complémentaire et non substitutive :

- **L'Authorization_Descriptor est le noyau** : L'autorisation hors ligne est toujours le mécanisme fondamental du protocole CAP. Les Trusted_Tickets ne peuvent pas exister indépendamment en dehors du système Authorization_Descriptor
- **L'émission en ligne peut être convertie au format hors ligne** : Les Trusted_Tickets émis en ligne peuvent être convertis au format local Authorization_Descriptor pour une utilisation hors ligne. Cela signifie que l'autorisation obtenue via les Trusted_Tickets ne devient pas immédiatement invalide en cas d'interruption réseau
- **Relation de priorité** : Lorsque le terminal détient simultanément un Trusted_Ticket et un Authorization_Descriptor, les informations d'autorisation en temps réel fournies par le Trusted_Ticket sont prioritaires, car elles offrent une meilleure réactivité

#### Stratégie de dégradation du mode en ligne vers le mode hors ligne

Lorsque les services en ligne deviennent indisponibles, le protocole CAP exécute une stratégie de dégradation fluide :

1. **Détection de l'état des services en ligne** : Le terminal surveille en continu l'état de la connexion avec le service d'autorisation en ligne
2. **Basculement automatique** : Lorsque les services en ligne sont indisponibles, le terminal bascule automatiquement vers la vérification locale de l'Authorization_Descriptor sans interrompre l'accès du Fay aux ressources
3. **Conversion des Trusted_Tickets** : Les Trusted_Tickets émis pendant la période connectée ont déjà été convertis au format local Authorization_Descriptor pour le stockage, garantissant leur utilisabilité après la dégradation
4. **Synchronisation après rétablissement** : Lorsque les services en ligne redeviennent disponibles, le terminal reprend automatiquement l'utilisation du mécanisme Trusted_Ticket et synchronise les informations de révocation éventuellement manquées pendant la période hors ligne

Le processus de dégradation est transparent pour le Fay — le Fay n'a pas besoin de savoir si le terminal utilise actuellement des tickets en ligne ou une autorisation hors ligne. Les résultats de la vérification d'autorisation restent cohérents dans les deux modes.

### 2.3 Gestion des sessions (Session)

#### Cycle de vie

Une Session (session de contrôle) est le cycle de vie complet depuis l'approbation de la vérification d'autorisation jusqu'à la fin de l'accès, comprenant les 3 phases suivantes :

1. **Création (Creation)** : Lorsque la vérification d'autorisation d'un Fay est approuvée, le Protocol_Engine crée une instance de Session pour celui-ci. Lors de la création, la Session enregistre l'identifiant du Fay associé, la Terminal_Resource cible, la liste des permissions autorisées et l'heure de création. Une fois la création réussie, le Fay obtient le contrôle de la ressource cible. Par exemple, un iFay demande à utiliser le navigateur de l'ordinateur portable ; après la vérification d'autorisation, le système crée une session de contrôle liée au navigateur, et l'iFay peut opérer le navigateur à partir de ce moment.

2. **Active (Active)** : Après la création, la Session entre dans l'état actif, durant lequel le Fay peut effectuer des opérations sur la Terminal_Resource liée dans le cadre des permissions autorisées. Pendant la période active, le mécanisme Liveness_Detection surveille en continu l'état de vivacité de la session, s'assurant que le Fay utilise toujours activement la session. Par exemple, l'iFay est en train de rechercher des informations de vol via le navigateur pour l'utilisateur, envoyant continuellement des battements de cœur pendant ce temps pour indiquer qu'il est toujours en utilisation active.

3. **Terminaison (Termination)** : Une Session peut être terminée par les méthodes suivantes :
   - **Libération active** : Le Fay libère activement la Session après avoir terminé ses opérations, restituant le contrôle de la Terminal_Resource. Par exemple, l'iFay libère activement le contrôle du navigateur après avoir terminé la recherche de vols
   - **Terminaison par expiration** : La Liveness_Detection détecte que la session du Fay n'est plus active (expiration du battement de cœur), terminant automatiquement la Session et récupérant les ressources. Par exemple, l'iFay cesse d'envoyer des battements de cœur en raison d'un crash de l'environnement d'exécution, et le terminal récupère automatiquement le contrôle du navigateur après l'expiration du délai
   - **Terminaison par révocation** : Lorsque l'autorisation est révoquée, les Sessions actives associées sont terminées de force. Par exemple, l'utilisateur découvre que l'iFay consulte du contenu inapproprié et révoque immédiatement l'autorisation ; la session du navigateur est terminée de force

Après la terminaison de la Session, le contrôle du Fay sur la Terminal_Resource est immédiatement libéré, et la ressource revient à un état où elle peut être acquise par d'autres parties contrôlantes.

#### Relation de liaison avec la Terminal_Resource

Une relation de liaison stricte existe entre les Sessions et les Terminal_Resources : **une Session active correspond au contrôle d'une Terminal_Resource**.

Les règles fondamentales de cette relation de liaison :

- **Liaison un-à-un** : Chaque Session active est liée à une Terminal_Resource spécifique, et le Fay effectue des opérations sur cette ressource via la Session. Par exemple, un iFay obtient une Session liée à la « caméra frontale » — il ne peut opérer que la caméra frontale via cette Session et ne peut pas l'utiliser pour opérer le microphone
- **Contrôle exclusif** : Pour les modes d'opération nécessitant un accès exclusif (write, execute, configure), la même Terminal_Resource ne peut avoir au plus qu'une seule Session exclusive active à un instant donné. Par exemple, lorsque iFay-A écrit des données sur une carte SD en mode write, iFay-B ne peut pas obtenir simultanément une Session write pour la carte SD
- **Lecture concurrente** : Pour le mode read, la même Terminal_Resource peut avoir plusieurs Sessions en lecture seule actives simultanément. Par exemple, iFay-A et iFay-B peuvent simultanément lire les données de position du module GPS en mode read
- **Couplage du cycle de vie** : Lorsqu'une Terminal_Resource devient indisponible (par exemple, matériel déconnecté), toutes les Sessions actives liées à cette ressource seront terminées. Par exemple, après qu'un utilisateur débranche une caméra USB, toutes les Sessions liées à cette caméra sont automatiquement terminées

#### Mécanisme de détection de vivacité

La Liveness_Detection détecte si une session Fay est toujours active par la combinaison de connexions persistantes et de battements de cœur au niveau applicatif. Son objectif de conception est de récupérer rapidement les ressources occupées par les sessions expirées.

Le mécanisme de fonctionnement de la détection de vivacité :

- **Maintien de la connexion persistante** : Le Fay et le terminal maintiennent un canal de communication via une connexion persistante, l'état de la connexion servant de signal de base pour la détection de vivacité
- **Battement de cœur au niveau applicatif** : Au-dessus de la connexion persistante, le Fay envoie périodiquement des messages de battement de cœur au niveau applicatif indiquant qu'il utilise toujours activement la Session en cours. L'intervalle de battement de cœur et le seuil d'expiration sont configurables
- **Double détermination** : Une Session n'est considérée comme expirée que lorsque la connexion persistante est interrompue et que le battement de cœur au niveau applicatif a expiré simultanément. Ce mécanisme de double détermination évite les faux positifs causés par de brèves fluctuations réseau
- **Récupération des ressources** : Lorsqu'une Session est déterminée comme expirée, le Protocol_Engine termine automatiquement la Session et libère la Terminal_Resource qu'elle occupait, rendant la ressource disponible pour d'autres parties contrôlantes

### 2.4 Politique de transfert de contrôle (Handover_Policy)

#### Scénarios principaux

Le transfert de contrôle intervient dans les scénarios où plusieurs parties contrôlantes doivent utiliser successivement la même Terminal_Resource. Le protocole CAP définit deux types de scénarios de transfert principaux :

- **Transfert de contrôle entre Fays** : Un Fay transfère son contrôle sur une Terminal_Resource à un autre Fay. Par exemple, après qu'un iFay responsable de la collecte de données a terminé sa tâche, il transfère le contrôle de la caméra à un iFay responsable des appels vidéo ; ou dans une usine intelligente, après qu'un coFay responsable du contrôle qualité a terminé de photographier les produits, il transfère le contrôle de la caméra industrielle au coFay responsable de l'emballage
- **Transfert de contrôle entre Fays et utilisateurs humains** : Un Fay restitue le contrôle à un utilisateur humain, ou un utilisateur humain délègue le contrôle à un Fay. Par exemple, un drone est piloté de manière autonome par un iFay lorsque le pilote remarque des conditions météorologiques anormales et doit reprendre le contrôle manuellement — l'iFay cède immédiatement le contrôle de vol ; une fois le temps amélioré, le pilote rend le contrôle à l'iFay pour continuer le vol autonome. Autre exemple, un utilisateur est en train d'éditer manuellement un document et a besoin que l'iFay l'aide à mettre en forme, transférant temporairement le contrôle de l'éditeur de documents à l'iFay ; une fois la mise en forme terminée, l'iFay restitue le contrôle

#### Capacités de configuration par politiques

La Handover_Policy fournit des capacités de configuration par politiques, prenant en charge les 3 types de politiques suivants :

1. **Script de règles de priorité (Priority Rule Script)** : Détermine la priorité du transfert de contrôle via des scripts de règles prédéfinis. Les scripts de règles calculent des scores de priorité basés sur des facteurs tels que le rôle du Fay, l'urgence de la tâche et le type de ressource, la partie contrôlante ayant la priorité la plus élevée obtenant le contrôle de la ressource. Cette politique convient aux scénarios où les règles de transfert sont claires et peuvent être prédéfinies. Par exemple, dans un scénario médical, l'iFay responsable des alertes d'urgence a toujours une priorité plus élevée que l'iFay responsable de la collecte de données de routine ; lorsque les deux demandent simultanément l'utilisation du module de communication, l'iFay d'alerte d'urgence obtient automatiquement le contrôle.

2. **Décision en temps réel par modèle IA (AI Model Real-time Decision)** : Un modèle IA détermine l'attribution du contrôle en temps réel en fonction du contexte actuel. Le modèle IA prend des décisions en considérant de manière globale des facteurs tels que l'état des tâches des différentes parties contrôlantes, l'urgence d'utilisation des ressources et les schémas historiques de transfert. Cette politique convient aux scénarios dynamiques où les règles de transfert sont complexes et difficiles à couvrir par des règles statiques. Par exemple, dans une maison intelligente, un iFay responsable de la surveillance de sécurité et un iFay responsable des appels vidéo demandent simultanément la caméra ; le modèle IA détermine la priorité en temps réel en fonction d'informations contextuelles telles que l'existence d'une menace de sécurité actuelle et l'urgence de l'appel.

3. **Décision de l'utilisateur humain (Human User Decision)** : Le pouvoir de décision pour le transfert de contrôle est confié à l'utilisateur humain. Lorsque plusieurs parties contrôlantes sont en concurrence pour la même ressource, le terminal présente une interface de sélection à l'utilisateur humain, qui décide à quelle partie accorder le contrôle. Cette politique convient aux ressources hautement sensibles ou aux scénarios nécessitant un jugement humain. Par exemple, deux iFays demandent simultanément à utiliser l'application bancaire de l'utilisateur ; le terminal affiche une invite demandant à l'utilisateur de décider lequel autoriser, car les opérations financières nécessitent une validation humaine finale.

Les trois types de politiques peuvent être configurés indépendamment à la granularité de la Terminal_Resource — différentes ressources sur le même terminal peuvent adopter différentes politiques de transfert.

#### Garantie d'atomicité

Le processus de transfert de contrôle doit satisfaire les exigences d'atomicité : **à tout instant donné, une Terminal_Resource a au plus une seule partie contrôlante active**.

Principes de mise en œuvre de la garantie d'atomicité :

- **Libération avant acquisition** : Durant le processus de transfert, la Session de la partie contrôlante d'origine est d'abord terminée, puis la Session de la nouvelle partie contrôlante est créée, garantissant que deux parties contrôlantes ne détiennent jamais simultanément le contrôle de la même ressource. Par exemple, lors du transfert du contrôle de vol d'un drone de iFay-A à iFay-B, la session de contrôle de iFay-A doit d'abord être terminée avant que la session de contrôle de iFay-B puisse être créée — il n'y aura jamais de situation où deux iFays contrôlent simultanément le vol du drone
- **Indivisibilité** : L'opération de transfert est exécutée comme un tout — soit elle réussit complètement (libération par la partie d'origine + acquisition par la nouvelle partie), soit elle échoue complètement (retour à l'état antérieur au transfert)
- **Aucune exposition d'état intermédiaire** : Les observateurs externes voient un état de contrôle des ressources cohérent à tout instant et n'observent pas les états intermédiaires du processus de transfert

#### Principes de gestion des délais d'expiration

Le processus de transfert de contrôle peut ne pas se terminer dans le délai prévu pour diverses raisons (latence réseau, absence de réponse de la partie contrôlante, etc.). Le principe de gestion des délais d'expiration du protocole CAP est : **un transfert non terminé dans le délai imparti doit être annulé et revenir à l'état antérieur au transfert, en maintenant la session de la partie contrôlante d'origine inchangée**.

Règles de gestion spécifiques :

- **Seuil d'expiration configurable** : Chaque politique de transfert peut configurer un seuil d'expiration indépendant, adapté aux exigences temporelles de différents scénarios
- **Protection par annulation** : Après le déclenchement de l'expiration, l'opération de transfert est automatiquement annulée, et la Session de la partie contrôlante d'origine reste dans son état actif sans être affectée. Par exemple, iFay-A contrôle l'appareil photo et tente de transférer le contrôle à iFay-B, mais iFay-B ne parvient pas à répondre dans le délai imparti en raison d'une latence réseau ; le transfert est automatiquement annulé et iFay-A continue de maintenir le contrôle de l'appareil photo
- **Mécanisme de notification** : Après l'annulation par expiration, le Protocol_Engine envoie une notification d'échec de transfert aux parties contrôlantes concernées, expliquant la raison de l'expiration
- **Stratégie de nouvelle tentative** : Après l'échec du transfert, la configuration de la politique détermine s'il faut réessayer automatiquement ou attendre une intervention manuelle

### 2.5 Mode d'accès aux ressources (Resource_Access_Mode)

#### Modèle hiérarchisé

Le Resource_Access_Mode classe l'accès aux ressources en 4 modes selon le type d'opération, formant une hiérarchie de permissions du plus bas au plus élevé :

1. **read (lecture)** : Accès en lecture seule à la Terminal_Resource, sans modification de l'état de la ressource. Par exemple, lecture des données en temps réel d'un capteur de température, consultation des photos dans l'album de l'utilisateur, obtention de la position actuelle du module GPS. Le mode read est partageable, permettant à plusieurs Fays d'accéder simultanément à la même ressource en mode read — par exemple, plusieurs iFays peuvent lire simultanément les données d'un même capteur de température sans interférence mutuelle.

2. **write (écriture)** : Écriture de données ou modification de l'état de la Terminal_Resource. Par exemple, sauvegarde de photos prises sur un dispositif de stockage, modification de la valeur de température d'un appareil domotique, écriture de contenu dans un document. Le mode write est exclusif, n'autorisant qu'une seule partie contrôlante à accéder à la ressource en mode write à un instant donné — par exemple, deux iFays ne peuvent pas écrire simultanément des données dans le même fichier, car cela entraînerait une corruption des données.

3. **execute (exécution)** : Exécution de commandes opérationnelles sur la Terminal_Resource. Par exemple, lancement d'une application de navigation sur le téléphone, déclenchement de la commande de décollage d'un drone, exécution d'une tâche d'impression sur une imprimante. Le mode execute est exclusif, garantissant que l'exécution des commandes opérationnelles n'est pas perturbée par d'autres parties contrôlantes — par exemple, lorsqu'un iFay contrôle un drone pour exécuter une procédure d'atterrissage, un autre iFay ne peut pas envoyer simultanément une commande de décollage.

4. **configure (configuration)** : Modification de la configuration au niveau système de la Terminal_Resource. Par exemple, modification des paramètres de résolution et de fréquence d'images de l'appareil photo, ajustement des règles du pare-feu réseau, changement des paramètres d'appairage du module Bluetooth. Le mode configure est exclusif et à haut niveau de privilège, nécessitant généralement un niveau d'autorisation plus élevé pour être exécuté — par exemple, la modification de la politique de sécurité d'un routeur nécessite un niveau d'autorisation plus élevé que la simple lecture de l'état du réseau.

#### Essence du verrou lecture-écriture

Le contrôle de concurrence du Resource_Access_Mode est essentiellement un modèle de verrou lecture-écriture (Read-Write Lock) :

- **Les opérations read permettent l'accès concurrent de plusieurs Fays** : Plusieurs Fays peuvent détenir simultanément des Sessions en mode read sur la même Terminal_Resource, sans interférence mutuelle. Cela est similaire au verrou partagé (Shared Lock) dans le modèle de verrou lecture-écriture. Par exemple, trois iFays peuvent lire simultanément les données de température et d'humidité d'un même capteur météorologique sans se bloquer mutuellement
- **Les opérations write, execute et configure nécessitent un contrôle exclusif** : Lorsqu'un Fay accède à une ressource en mode write, execute ou configure, aucun autre Fay ne peut accéder simultanément à cette ressource dans un mode d'écriture quelconque. Cela est similaire au verrou exclusif (Exclusive Lock) dans le modèle de verrou lecture-écriture. Par exemple, lorsqu'un iFay écrit un fichier vidéo sur une carte SD, un autre iFay ne peut pas écrire simultanément des photos sur la même carte SD
- **Exclusion mutuelle lecture-écriture** : Lorsqu'une Session active en mode write, execute ou configure existe sur une ressource, les nouvelles demandes read sont également bloquées jusqu'à la libération de la Session exclusive. Inversement, lorsque des Sessions read actives existent sur une ressource, les demandes write, execute et configure attendent la fin de toutes les Sessions read. Par exemple, un iFay est en train de modifier un fichier de configuration (mode write), et un autre iFay souhaitant lire ce fichier doit également attendre la fin de l'écriture, afin d'éviter de lire un état intermédiaire incohérent

Ce modèle de verrou lecture-écriture maximise les performances de concurrence des opérations read tout en garantissant la cohérence des données et la sécurité des opérations.

#### Relation avec les permissions de Session

Le Resource_Access_Mode est étroitement lié aux permissions d'autorisation de la Session : **la liste des permissions d'autorisation de la Session détermine les types d'opérations que le Fay peut exécuter**.

La relation spécifique est la suivante :

- **Contrainte par la liste de permissions** : Chaque Session porte une liste de permissions d'autorisation lors de sa création, cette liste provenant de la portée des permissions définie dans l'Authorization_Descriptor ou le Trusted_Ticket. Le Fay ne peut accéder à la ressource que dans les modes inclus dans la liste de permissions
- **Principe du moindre privilège** : La liste de permissions de la Session doit suivre le principe du moindre privilège, ne contenant que les modes de permission minimaux nécessaires au Fay pour accomplir sa tâche en cours
- **Impossibilité d'élévation de permissions** : Après la création de la Session, sa liste de permissions ne peut pas être élevée pendant le cycle de vie de la session. Si le Fay a besoin d'un mode d'accès à privilège plus élevé, il doit créer une nouvelle Session via une nouvelle vérification d'autorisation
- **Pas d'inclusion hiérarchique des permissions** : Un mode de permission élevé n'inclut pas automatiquement les capacités des modes de permission inférieurs. Par exemple, détenir la permission configure ne signifie pas posséder automatiquement la permission read ; chaque mode d'accès nécessite une autorisation indépendante
