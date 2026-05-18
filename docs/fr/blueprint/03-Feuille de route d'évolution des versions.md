## Chapitre 3 : Feuille de route d'évolution des versions

Ce chapitre définit la feuille de route d'évolution des versions du protocole CAP, précise le périmètre fonctionnel et les exclusions de la version v1, établit les règles de nommage des versions et la stratégie de compatibilité, et décrit le processus de publication depuis le brouillon jusqu'à la version officielle. La feuille de route des versions fournit des orientations claires pour le développement itératif du protocole et la collaboration communautaire.

### 3.1 Périmètre fonctionnel de la version v1

La version v1 du protocole CAP se concentre sur l'établissement d'un cadre de base complet pour la gestion des autorisations de contrôle des terminaux, comprenant les 6 capacités principales suivantes :

1. **Autorisation hors ligne (Authorization_Descriptor)** : Mécanisme principal de la version v1. Implémentation de la gestion complète du cycle de vie de l'Authorization_Descriptor, incluant l'émission, le stockage local, la validation, la révocation et le renouvellement. Le terminal peut effectuer indépendamment la vérification d'autorisation en mode hors ligne via l'Authorization_Descriptor stocké localement, garantissant la disponibilité de base du Fay dans les environnements sans réseau. La version v1 implémentera le modèle complet de chaîne de confiance et le mécanisme de liste de révocation locale.

2. **Tickets en ligne (Trusted_Ticket)** : Mécanisme complémentaire de la version v1. Implémentation des capacités d'émission d'autorisation en temps réel et de consultation de l'état de révocation dans les environnements connectés, prise en charge de la conversion des Trusted_Tickets au format local Authorization_Descriptor, ainsi que de la stratégie de dégradation fluide du mode en ligne vers le mode hors ligne. La version v1 assure le fonctionnement coordonné entre les tickets en ligne et l'autorisation hors ligne.

3. **Gestion des sessions (Session lifecycle)** : Implémentation de la gestion complète du cycle de vie des sessions de contrôle, couvrant les trois phases de création, activité et terminaison. La version v1 prend en charge la relation de liaison un-à-un entre Session et Terminal_Resource, et implémente les trois modes de terminaison de session : libération active, terminaison par expiration et terminaison par révocation.

4. **Politique de transfert de contrôle (Handover_Policy)** : Implémentation de la capacité de transfert de contrôle entre plusieurs parties contrôlantes, couvrant les scénarios de transfert entre Fays ainsi qu'entre Fays et utilisateurs humains. La version v1 prend en charge trois types de politiques (script de règles de priorité, décision en temps réel par modèle IA, décision de l'utilisateur humain), garantit l'atomicité du processus de transfert et implémente le mécanisme d'annulation par expiration.

5. **Détection de vivacité (Liveness_Detection)** : Implémentation du mécanisme de détection de vivacité des sessions basé sur la combinaison de connexions persistantes et de battements de cœur au niveau applicatif. La version v1 prend en charge la logique de double détermination (interruption de la connexion persistante et expiration du battement de cœur simultanées), les intervalles de battement de cœur et seuils d'expiration configurables, ainsi que la terminaison automatique des sessions expirées et la récupération des ressources.

6. **Mode d'accès aux ressources (Resource_Access_Mode)** : Implémentation de la gestion hiérarchisée de l'accès aux ressources par type d'opération, prenant en charge les quatre modes d'accès read, write, execute et configure. La version v1 implémente le modèle complet de verrou lecture-écriture, prenant en charge l'accès concurrent multipartite en mode read et le contrôle exclusif en modes write, execute et configure.

Ces 6 capacités principales constituent l'ensemble fonctionnel complet de la version v1, fournissant un cadre de base pour le contrôle des autorisations permettant aux agents intelligents de l'écosystème iFay de prendre en charge les terminaux de manière sécurisée.

### 3.2 Fonctionnalités explicitement exclues de la v1

Les fonctionnalités suivantes ne sont explicitement pas incluses dans la version v1 et sont identifiées comme candidates pour les versions futures :

| Fonctionnalité exclue | Description | Version candidate |
|----------------------|-------------|-------------------|
| **Migration de session inter-terminaux** | Migration d'une Session active d'un terminal à un autre, en maintenant la continuité de l'état de la session | v2+ |
| **Autorisation collaborative multi-terminaux** | Mécanisme de synchronisation de l'état d'autorisation et de vérification collaborative entre plusieurs terminaux | v2+ |
| **Chaîne de délégation d'autorisation (Delegation Chain)** | Mécanisme d'autorisation en cascade permettant à un Fay de déléguer une partie de ses autorisations à d'autres Fays | v2+ |
| **Contrôle d'accès aux ressources à granularité fine (ABAC)** | Politique de contrôle d'accès basée sur les attributs, prenant en charge des expressions de conditions d'accès aux ressources plus complexes | v2+ |
| **Format standardisé des journaux d'audit** | Format de sortie unifié des journaux d'audit (Audit_Logger) et spécification d'interface de requête | v2+ |
| **Spécification de chiffrement des messages du protocole** | Définition standardisée du mode de chiffrement et du processus de négociation de clés pour les messages du protocole CAP au niveau de la couche de transport réseau | v2+ |
| **Consensus distribué de révocation hors ligne** | Mécanisme de consensus entre plusieurs terminaux pour atteindre un accord sur l'état de révocation dans un environnement hors ligne | v3+ |
| **Élévation dynamique de permissions (intra-Session)** | Élévation dynamique des permissions d'accès au sein du cycle de vie d'une Session active sans nécessiter la création d'une nouvelle Session | v3+ |

La version v1 suit le principe « établir d'abord les fondations, puis étendre les capacités », en priorisant la complétude et la stabilité des 6 capacités principales, et en réservant les fonctionnalités avancées aux itérations de versions ultérieures.

### 3.3 Règles de nommage des versions et stratégie de compatibilité

#### Règles de nommage des versions

Le protocole CAP adopte le format de versionnage sémantique (Semantic Versioning) : **`MAJOR.MINOR.PATCH`**

- **MAJOR (numéro de version majeure)** : Incrémenté lorsque le protocole subit des modifications majeures non rétrocompatibles. Par exemple, restructuration du format des messages principaux, changement fondamental du flux de vérification d'autorisation. Un changement de version majeure signifie que les implémentations existantes nécessitent des modifications d'adaptation
- **MINOR (numéro de version mineure)** : Incrémenté lorsque le protocole ajoute des fonctionnalités ou capacités rétrocompatibles. Par exemple, ajout d'un nouveau mode d'accès aux ressources, extension des types de politiques de Handover_Policy. Un changement de version mineure n'affecte pas le fonctionnement normal des implémentations existantes
- **PATCH (numéro de correctif)** : Incrémenté lorsque le protocole fait l'objet de corrections d'erreurs ou de clarifications documentaires rétrocompatibles. Par exemple, correction de descriptions ambiguës dans la documentation de spécification, ajout d'explications pour le traitement des cas limites

Les versions brouillon utilisent l'identifiant **`draft`**, sans numéro de version. Les versions brouillon ne garantissent aucune compatibilité et peuvent subir des modifications destructrices à tout moment.

Exemples de numéros de version officiellement publiés : `1.0.0`, `1.1.0`, `1.1.1`, `2.0.0`

#### Stratégie de compatibilité

La stratégie de compatibilité du protocole CAP suit les principes suivants :

- **Rétrocompatibilité au sein d'une même version majeure** : Au sein d'une même version MAJOR, les nouvelles versions de l'implémentation du protocole doivent pouvoir traiter les messages des anciennes versions. Par exemple, un terminal implémentant la v1.2.0 doit pouvoir traiter les Authorization_Descriptors au format v1.0.0
- **Pas de garantie de compatibilité entre versions majeures** : La rétrocompatibilité n'est pas garantie entre différentes versions MAJOR. Lors de l'incrémentation du numéro de version majeure, la spécification du protocole listera explicitement toutes les modifications destructrices et le guide de migration
- **Alignement des versions Schema et protocole** : Chaque version du protocole correspond à une version de Schema, les fichiers Schema étant stockés dans un répertoire nommé d'après le numéro de version du protocole (par exemple `schema/2025-10-25/`), garantissant une correspondance de version claire et traçable
- **Déclaration de version minimale supportée** : Les implémentations du protocole peuvent déclarer la version minimale du protocole qu'elles supportent ; les messages de protocole inférieurs à cette version seront refusés

### 3.4 Processus de publication du brouillon à la version officielle

La publication du protocole CAP, du brouillon à la version officielle, suit le processus suivant :

**Phase 1 : Brouillon (Draft)**

- Les spécifications du protocole et les définitions de Schema sont stockées dans le répertoire `draft/` (par exemple `schema/draft/`, `specification/draft/`)
- La phase de brouillon autorise des modifications destructrices fréquentes, sans engagement de compatibilité
- Le contenu du brouillon accepte les retours et les revues de la communauté, avec des itérations d'optimisation continues
- Les documents et Schema de la phase de brouillon peuvent être mis à jour à tout moment, sans nécessité de suivre les règles d'incrémentation des numéros de version

**Phase 2 : Version candidate (Release Candidate)**

- Lorsque le contenu du brouillon tend vers la stabilité, il entre en phase de version candidate
- Les versions candidates utilisent le suffixe `-rc.N` (par exemple `1.0.0-rc.1`)
- La phase de version candidate n'accepte que les corrections d'erreurs et les clarifications documentaires, pas de nouvelles fonctionnalités
- Les versions candidates doivent passer une validation complète par les tests, incluant la validation de Schema, la vérification d'équivalence structurelle des documents multilingues et les tests de propriétés

**Phase 3 : Publication officielle (Release)**

- Après une validation suffisante, la version candidate est promue en version officielle en supprimant le suffixe `-rc.N`
- Les spécifications du protocole et les définitions de Schema de la version officielle sont copiées du répertoire `draft/` vers un répertoire nommé d'après la date de version (par exemple `schema/2025-10-25/`)
- Après la publication de la version officielle, les spécifications du protocole et les définitions de Schema de cette version ne sont plus modifiées (immutable) ; toute modification est publiée via une nouvelle version
- Lors de la publication d'une version officielle, les documents de plan d'architecture et les spécifications du protocole dans les 9 langues doivent être simultanément complétés et avoir passé la vérification d'équivalence structurelle

**Phase 4 : Maintenance (Maintenance)**

- Après la publication de la version officielle, celle-ci entre en phase de maintenance, n'acceptant que les corrections d'erreurs de niveau PATCH
- Lorsqu'une nouvelle version MAJOR ou MINOR est publiée, l'ancienne version entre dans une période de maintenance limitée, pour être finalement marquée comme obsolète (deprecated)
- Les documents et définitions de Schema des versions obsolètes sont conservés dans le dépôt pour référence historique, mais ne reçoivent plus aucune mise à jour
