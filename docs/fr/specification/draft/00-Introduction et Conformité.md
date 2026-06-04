# Chapitre 0 : Introduction et Conformité

## 0.1 État du Document

Ce document est la version brouillon de la spécification technique normative du Control Authority Protocol (CAP). La version brouillon ne garantit pas la rétrocompatibilité et peut subir des modifications arbitraires avant la publication officielle. Une fois ce brouillon stabilisé et ayant passé la vérification complète des tests, il sera publié comme première version officielle dans `specification/2025-10-25/`.

Ce document est destiné à être utilisé conjointement avec le plan d'architecture CAP (`docs/fr/blueprint/`) :

- Le **plan d'architecture** répond à « quoi, pourquoi et pour quoi » — il définit l'intention de conception du protocole, les limites de capacité et les mécanismes principaux
- La **spécification** répond à « comment et comment vérifier » — elle définit les formats de message du protocole, les étapes du flux, la gestion des erreurs et les exigences de conformité

Lorsque les descriptions non normatives du plan d'architecture entrent en conflit avec des dispositions normatives de cette spécification, cette spécification prévaut.

## 0.2 Portée

Cette spécification définit les détails techniques du protocole CAP v1, couvrant les 6 capacités principales listées à la Section 3.1 du Chapitre 3 du plan d'architecture :

1. Émission, stockage, validation, révocation et renouvellement de l'autorisation hors ligne (Authorization_Descriptor)
2. Émission, requête et conversion en autorisation hors ligne des tickets en ligne (Trusted_Ticket)
3. Cycle de vie complet de la gestion des sessions (Session)
4. Trois types de politiques et garanties d'atomicité de la politique de transfert de contrôle (Handover_Policy)
5. Mécanisme de double détermination de la détection de vivacité (Liveness_Detection)
6. Modèle de verrou lecture-écriture du mode d'accès aux ressources (Resource_Access_Mode)

Cette spécification exclut explicitement les fonctionnalités listées à la Section 3.2 du Chapitre 3 du plan d'architecture, incluant la migration de session entre terminaux, l'autorisation collaborative multi-terminaux, les chaînes de délégation d'autorisation, ABAC, le format standardisé de journaux d'audit, la spécification de transmission chiffrée des messages de protocole, le consensus de révocation distribué et l'élévation dynamique de permissions au sein d'une Session.

## 0.3 Terminologie de Conformité

Cette spécification suit les conventions de mots-clés de RFC 2119 et RFC 8174. Les mots-clés suivants, lorsqu'ils apparaissent en **majuscules**, ont une signification normative :

- **MUST / doit** : Exigence absolue. Les implémentations qui ne satisfont pas cette exigence ne sont pas conformes à cette spécification
- **MUST NOT / ne doit pas** : Interdiction absolue. Les implémentations qui violent cette interdiction ne sont pas conformes à cette spécification
- **SHOULD / devrait** : Forte recommandation. Peut s'en écarter pour des raisons valables avec une compréhension complète des conséquences
- **SHOULD NOT / ne devrait pas** : Forte dissuasion. Peut s'en écarter pour des raisons valables avec une compréhension complète des conséquences
- **MAY / peut** : Optionnel. Les implémentations peuvent décider elles-mêmes de fournir ou non

Le même vocabulaire n'apparaissant pas en majuscules ne porte que son sens littéral et n'a aucune force normative.

## 0.4 Alignement de la Terminologie

La terminologie utilisée dans cette spécification est entièrement cohérente avec le glossaire `00-Glossaire.md` du plan d'architecture. Lorsque cette spécification fait référence à un terme, elle utilise l'identifiant défini dans le plan d'architecture (par exemple, `Authorization_Descriptor`, `Fay`, `Terminal_Resource`).

Pour faciliter la référence, cette spécification marque les termes clés en **gras** lors de leur première occurrence dans chaque chapitre, avec de brèves définitions du glossaire du plan d'architecture. Les définitions complètes des termes sont régies par le plan d'architecture.

## 0.5 Niveaux de Conformité

Cette spécification définit 3 niveaux de conformité d'implémentation. Une implémentation DOIT satisfaire au moins le « Niveau de Conformité Terminal » pour revendiquer la conformité avec CAP v1.

### 0.5.1 Conformité Terminal (Terminal Conformance)

S'applique aux dispositifs terminaux implémentant `Descriptor_Validator`, `Protocol_Engine` et la logique de gestion de session. Les implémentations terminales DOIVENT :

1. Implémenter complètement le flux de validation Authorization_Descriptor défini au Chapitre 3
2. Implémenter complètement le cycle de vie Session et le mécanisme Liveness_Detection définis au Chapitre 5
3. Implémenter complètement la sémantique de verrou lecture-écriture Resource_Access_Mode définie au Chapitre 7
4. Rejeter toutes les requêtes qui ne passent pas la validation et retourner des codes d'erreur standardisés selon le Chapitre 9
5. Maintenir une liste de révocation locale et la synchroniser selon la politique définie au Chapitre 3 lorsqu'en ligne

### 0.5.2 Conformité Émetteur (Issuer Conformance)

S'applique aux entités de confiance implémentant `Descriptor_Issuer` ou `Ticket_Issuer`. Les implémentations émettrices DOIVENT :

1. Générer Authorization_Descriptor et Trusted_Ticket légitimes selon les contraintes de champs définies au Chapitre 2
2. Appliquer des signatures numériques aux justificatifs selon les exigences cryptographiques définies au Chapitre 8
3. Maintenir des enregistrements d'état des justificatifs émis et supporter les opérations de révocation
4. Implémenter la conversion Trusted_Ticket vers Authorization_Descriptor définie au Chapitre 4

### 0.5.3 Conformité Runtime (Runtime Conformance)

S'applique aux environnements d'exécution Fay implémentant `iFay_Runtime`. Les implémentations runtime DOIVENT :

1. Soumettre des requêtes de validation d'autorisation à Protocol_Engine selon le contrat d'interface défini au Chapitre 1
2. Maintenir des connexions persistantes et envoyer des battements de cœur de couche d'application à la fréquence définie au Chapitre 5
3. Recevoir et transmettre les notifications de changement d'état de session poussées par Protocol_Engine
4. Notifier proactivement Protocol_Engine pour libérer les Sessions associées lorsqu'une instance Fay se termine

Une implémentation peut satisfaire plusieurs niveaux de conformité simultanément. Par exemple, un terminal intégré peut servir à la fois d'implémentation terminale et d'implémentation runtime.

## 0.6 Références Normatives

Cette spécification fait référence normativement aux documents suivants :

- RFC 2119 / RFC 8174 : Conventions de mots-clés utilisées dans cette spécification
- RFC 8949 (CBOR) : Sérialisation binaire compacte pour Authorization_Descriptor (voir Chapitre 2)
- RFC 8032 (EdDSA) : Algorithme de signature numérique par défaut (voir Chapitre 8)
- RFC 7515 (JWS) : Sérialisation et signature JSON pour Trusted_Ticket (voir Chapitre 4)
- RFC 5280 (X.509) : Format de certificat optionnel (voir Chapitre 8)

Le fichier de définition de Schéma CAP (`schema/{version}/schema.json`) sert de complément formel au Chapitre 2 de cette spécification, avec une force normative équivalente à cette spécification. Lorsque schema.json entre en conflit avec la description textuelle de cette spécification, schema.json prévaut.

## 0.7 Structure du Document

Cette spécification est organisée dans l'ordre suivant :

| Chapitre | Sujet | Contenu Principal |
|------|------|---------|
| Chapitre 1 | Architecture et Rôles | Rôles du protocole, chaîne de confiance, contrats d'interface externes |
| Chapitre 2 | Modèle de Données | Définitions au niveau des champs des structures de données principales |
| Chapitre 3 | Protocole d'Autorisation Hors Ligne | Flux complet d'Authorization_Descriptor |
| Chapitre 4 | Protocole de Ticket En Ligne | Flux complet de Trusted_Ticket et dégradation |
| Chapitre 5 | Gestion des Sessions et Détection de Vivacité | Machine à états Session et battement de cœur |
| Chapitre 6 | Protocole de Transfert de Contrôle | Trois types de politiques Handover_Policy |
| Chapitre 7 | Mode d'Accès aux Ressources | Sémantique de read/write/execute/configure |
| Chapitre 8 | Cryptographie et Signatures | Ensemble d'algorithmes, formats de clés, distribution |
| Chapitre 9 | Codes d'Erreur et Niveaux de Conformité | Codes d'erreur standard, déclaration de niveau |
| Chapitre 10 | Considérations de Sécurité | Modèle de menace, risques connus |

Il est recommandé aux lecteurs de lire les Chapitres 0–2 dans l'ordre pour établir une base, puis de sauter aux chapitres pertinents selon le focus d'implémentation.
