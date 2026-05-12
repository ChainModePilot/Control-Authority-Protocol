## Chapitre 5 : Relation entre Schema et livrables

Ce chapitre décrit la structure du système de livrables du projet CAP, précise la position centrale des fichiers de définition de Schema du protocole dans les livrables, ainsi que la correspondance entre les versions de Schema et les versions du protocole. Comprendre les relations hiérarchiques et les dépendances entre les livrables est la base pour que les implémenteurs du protocole et les contributeurs de la communauté participent à la collaboration du projet.

### 5.1 Trois catégories de livrables principaux

Le système de livrables du projet CAP se compose de trois catégories de livrables principaux, formant une chaîne de livraison complète de la documentation à la définition puis à l'implémentation :

| Catégorie de livrable | Description | Exemple de chemin dans le dépôt |
|----------------------|-------------|-------------------------------|
| **Documentation multilingue du protocole** | Comprend le plan d'architecture (Blueprint) et les spécifications techniques du protocole (Specification), publiés de manière équivalente en 9 langues, fournissant une description lisible par l'humain de l'intention de conception, du périmètre des capacités et des détails techniques du protocole | `docs/{lang}/blueprint/`, `docs/{lang}/specification/` |
| **Fichiers de définition de Schema du protocole** | Définition faisant autorité du format des messages du protocole, fournie en plusieurs formats lisibles par la machine et par l'humain, servant de référence pour l'implémentation des SDK et les tests d'interopérabilité | `schema/{version}/` |
| **Implémentations SDK dans différents langages** | Implémentations concrètes du protocole dans des langages de programmation spécifiques, développées sur la base des fichiers de définition de Schema, fournissant aux développeurs de différentes piles technologiques des capacités d'intégration du protocole prêtes à l'emploi | `sdk/{language}/` |

Relations hiérarchiques entre les trois catégories de livrables :

- **La documentation multilingue du protocole** se situe au niveau stratégique et normatif, définissant la conception de haut niveau de « ce que fait » et « comment fait » le protocole
- **Les fichiers de définition de Schema du protocole** se situent au niveau de la définition, transformant les formats de messages de la spécification du protocole en définitions formalisées précises et traitables par la machine
- **Les implémentations SDK dans différents langages** se situent au niveau de l'implémentation, transformant les capacités du protocole en interfaces de programmation directement invocables sur la base des fichiers de définition de Schema

Les mises à jour des trois catégories de livrables suivent un chemin de propagation descendant : les modifications de la documentation du protocole entraînent la mise à jour des définitions de Schema, et les mises à jour des définitions de Schema entraînent l'adaptation des implémentations SDK.

### 5.2 Rôle des fichiers de définition de Schema

Les fichiers de définition de Schema sont la **définition faisant autorité (Authoritative Definition)** du format des messages du protocole CAP, assumant les rôles principaux suivants dans le système de livrables du projet :

- **Source unique de vérité (Single Source of Truth) pour le format des messages** : Les fichiers de définition de Schema décrivent avec précision les noms de champs, les types de données, les contraintes obligatoire/optionnel et les structures imbriquées de tous les types de messages du protocole CAP. En cas d'ambiguïté entre la description textuelle de la documentation du protocole et la définition de Schema, la définition de Schema fait foi
- **Référence de développement pour l'implémentation des SDK** : La logique de sérialisation/désérialisation des messages des SDK dans les différents langages doit strictement suivre la définition de Schema. Les développeurs de SDK consultent les fichiers de définition de Schema pour connaître la structure précise de chaque type de message, garantissant une cohérence totale du format des messages entre les implémentations SDK dans différents langages
- **Référence de validation pour les tests d'interopérabilité** : Les tests d'interopérabilité entre les SDK de différents langages utilisent la définition de Schema comme standard de validation. Un message sérialisé par un SDK doit pouvoir être correctement désérialisé par un autre SDK ; le fichier de définition de Schema est le seul critère pour déterminer la « correction »
- **Base pour la validation automatisée** : Les fichiers de définition de Schema (en particulier schema.json) peuvent être directement consommés par des outils automatisés pour la validation automatique du format des messages, la génération de code et la génération de documentation, réduisant les erreurs humaines

### 5.3 Correspondance entre versions de Schema et versions du protocole

Chaque version officiellement publiée du protocole CAP correspond à une version de Schema, les fichiers Schema étant stockés dans un répertoire nommé d'après la date de publication de la version du protocole, garantissant une correspondance de version claire et traçable.

**Règles de nommage des répertoires :**

| État du protocole | Répertoire Schema | Description |
|-------------------|-------------------|-------------|
| Brouillon (Draft) | `schema/draft/` | Définitions de Schema en cours de développement, modifiables à tout moment, sans engagement de compatibilité |
| Publication officielle | `schema/{AAAA-MM-JJ}/` | Nommé d'après la date de publication, par exemple `schema/2025-10-25/`, immuable après publication (immutable) |

**Règles de correspondance des versions :**

- **Correspondance un-à-un** : Chaque version officiellement publiée du protocole correspond exactement à un répertoire de version de Schema, le nom du répertoire utilisant la date de publication de cette version du protocole
- **Partage du brouillon** : Toutes les modifications de Schema en phase de brouillon partagent le répertoire `schema/draft/` ; la phase de brouillon ne distingue pas les numéros de version
- **Immutabilité** : Les fichiers dans les répertoires de Schema officiellement publiés ne sont plus modifiés après publication. Toute modification de Schema (y compris les corrections d'erreurs) est réalisée via la publication d'une nouvelle version
- **Conservation historique** : Les répertoires de Schema des anciennes versions sont conservés de manière permanente dans le dépôt pour référence historique et vérification de la rétrocompatibilité

**Correspondance des versions actuelles :**

| Version du protocole | Répertoire Schema | État |
|---------------------|-------------------|------|
| draft | `schema/draft/` | En développement |
| 2025-10-25 | `schema/2025-10-25/` | Publié |

### 5.4 Description des formats de fichiers Schema

Chaque répertoire de version de Schema contient 3 formats de fichiers de définition de Schema, chacun destiné à des scénarios d'utilisation et des consommateurs différents :

| Format | Nom de fichier | Usage | Consommateurs principaux |
|--------|---------------|-------|-------------------------|
| **JSON Schema** | `schema.json` | Définition du format des messages lisible par la machine, utilisée pour la validation automatisée, la génération de code et la vérification d'interopérabilité inter-langages | Outils automatisés, pipelines CI/CD, générateurs de code SDK |
| **TypeScript** | `schema.ts` | Définitions de types TypeScript, fournissant un support de types natif pour l'écosystème TypeScript/JavaScript, utilisées pour le développement de SDK TS/JS et la vérification de types dans l'IDE | Développeurs de SDK TypeScript/JavaScript, développeurs d'intégration front-end |
| **MDX** | `schema.mdx` | Format de document rendu, présentant les définitions de Schema de manière conviviale pour l'humain, utilisé pour l'affichage sur les sites de documentation et l'apprentissage du protocole | Apprenants du protocole, sites de documentation, réviseurs techniques |

**Relations entre les trois formats :**

- **schema.json est le format faisant autorité** : En cas de divergence entre les trois formats, schema.json fait foi. schema.ts et schema.mdx doivent maintenir une cohérence sémantique avec schema.json
- **schema.ts est le format de commodité pour le développement** : schema.ts transforme les définitions de types de schema.json en types natifs TypeScript, permettant aux développeurs TS/JS d'obtenir directement des suggestions de types et une vérification à la compilation dans l'IDE
- **schema.mdx est le format de présentation** : schema.mdx transforme les définitions structurées de schema.json en contenu documentaire rendu, prenant en charge la navigation interactive des définitions de Schema sur les sites de documentation

**Exemple de structure de répertoires :**

```
schema/
├── draft/
│   ├── schema.json
│   ├── schema.ts
│   └── schema.mdx
└── 2025-10-25/
    ├── schema.json
    ├── schema.ts
    └── schema.mdx
```
