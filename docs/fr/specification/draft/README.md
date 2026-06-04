# Spécification Technique du Protocole CAP (Brouillon)

Ce répertoire contient la version brouillon de la spécification technique du Control Authority Protocol (CAP) v1. La spécification est développée sur la base du plan d'architecture dans `docs/fr/blueprint/`, couvrant les 6 capacités principales listées dans §3.1 du Chapitre 3 du plan d'architecture.

## Structure du Document

| Chapitre | Fichier | Contenu |
|------|------|------|
| Chapitre 0 | `00-Introduction et Conformité.md` | État du document, portée, mots-clés RFC 2119, niveaux de conformité, références normatives |
| Chapitre 1 | `01-Architecture et Rôles.md` | Rôles du protocole, chaîne de confiance, contrats d'interface externes |
| Chapitre 2 | `02-Modèle de Données.md` | Structures de données principales (Authorization_Descriptor, Trusted_Ticket, Session, Verification_Key) |
| Chapitre 3 | `03-Protocole d'Autorisation Hors Ligne.md` | Flux complet du protocole de cycle de vie d'Authorization_Descriptor |
| Chapitre 4 | `04-Protocole de Ticket En Ligne.md` | Flux complet de Trusted_Ticket et dégradation |
| Chapitre 5 | `05-Gestion des Sessions et Détection de Vivacité.md` | Machine à états Session, règles de liaison, battements de cœur, double détermination, récupération par timeout |
| Chapitre 6 | `06-Protocole de Transfert de Contrôle.md` | Trois types de politiques Handover_Policy, garanties d'atomicité, rollback par timeout |
| Chapitre 7 | `07-Mode d'Accès aux Ressources.md` | Sémantique de read/write/execute/configure, matrice de verrou lecture-écriture |
| Chapitre 8 | `08-Cryptographie et Signatures.md` | Ensemble d'algorithmes, formats de clés, distribution et rotation |
| Chapitre 9 | `09-Codes d'Erreur et Niveaux de Conformité.md` | Tableau de codes d'erreur standard, déclaration de conformité |
| Chapitre 10 | `10-Considérations de Sécurité.md` | Modèle de menace, risques connus et mitigations |

## Ordre de Lecture Recommandé

1. Première lecture : Chapitre 0 → Chapitre 1 → Chapitre 2 → Chapitre 3
2. Implémenter terminal : Chapitres 0–3 → Chapitres 5, 7 → Chapitres 8, 9
3. Implémenter émetteur : Chapitres 0–2 → Chapitres 3, 4 → Chapitre 8
4. Implémenter iFay_Runtime : Chapitres 0, 1 → Chapitre 5 → Chapitre 9
5. Revue de sécurité : Chapitre 10 + lecture croisée des chapitres connexes

## État du Brouillon

Ce brouillon est en phase de discussion. Avant la publication officielle :

- Les noms de champs, codes d'erreur et seuils de contraintes peuvent être ajustés
- La structure des chapitres peut être réorganisée
- Aucune rétrocompatibilité n'est garantie

Une fois les discussions stabilisées, les contenus de ce répertoire seront publiés sous `docs/fr/specification/2025-10-25/`, la première version officielle du protocole CAP.

## Ressources Connexes

- Plan d'architecture : `docs/fr/blueprint/`
- Définitions de Schema (brouillon) : `schema/draft/`
- Autres versions linguistiques : Cette spécification a actuellement les versions zh-CN, zh-TW, ja, ko, en, de et es ; les langues restantes seront traduites avant la publication officielle
