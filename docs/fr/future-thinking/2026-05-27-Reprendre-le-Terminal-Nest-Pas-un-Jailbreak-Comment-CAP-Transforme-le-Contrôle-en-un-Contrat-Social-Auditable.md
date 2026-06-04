<!-- Date de Mise à Jour : 2026-05-27 -->

# Reprendre le Terminal N'est Pas un Jailbreak : Comment CAP Transforme le Contrôle en un Contrat Social Auditable

Dans le discours public, la phrase « l'IA prend le contrôle du terminal » sonne comme une annonce de catastrophe.
Parce que pour la plupart des gens, une seule image leur vient à l'esprit : tu t'endors et elle traite ton téléphone comme le sien ; tu ne regardes pas et elle traite ton ordinateur comme le sien ; tu ne l'as pas autorisée, mais elle peut tout de même « faire quelques choses au passage ».

Cette peur n'est pas sans fondement.
Au cours des deux dernières années, nous avons vu trop d'incidents : permissions hors de contrôle, données supprimées par accident, appels excédant l'autorité, décisions en boîte noire, impossibilité d'attribuer la responsabilité.
Au moment où l'IA passe de « répondre à des questions » à « agir sur le monde », la tolérance de la société tombe à zéro instantanément.

C'est pourquoi j'ai insisté : **l'IA ne peut pas être laissée à elle-même ; elle doit agir sous tutelle humaine.**
Mais si cette déclaration reste une simple posture, elle ne change rien à la réalité. Il faut l'écrire dans le protocole, dans le runtime, dans chaque porte du contrôle du terminal.

Le Control Authority Protocol (CAP) est exactement ce type de « protocole de porte ».
Il n'est pas responsable de rendre l'IA plus intelligente ; il est seulement responsable de répondre à une question extrêmement simple mais qui décide néanmoins si la société peut accepter l'IA :

**Ce Fay (iFay ou coFay) est-il autorisé à faire cette chose ?**

---

## 1. La Responsabilité Unique de CAP dans l'Écosystème iFay : Vérification d'Autorisation et Gestion du Contrôle

Le positionnement de CAP est sobre mais sans compromis :

Il porte une responsabilité centrale unique — **vérifier si un Fay a obtenu l'autorisation d'un Human Prime (le sujet de responsabilité de personne physique) ou d'un Official Post (poste organisationnel / rôle public), accédant ainsi légitimement aux ressources du terminal**.

Cette phrase se décompose en cinq actions d'ingénierie :

1) **Vérification d'autorisation** : Le terminal vérifie si le justificatif d'autorisation porté par un Fay est légitime, valide et non révoqué.
2) **Gestion de session** : Après vérification réussie, une session de contrôle est établie et son cycle de vie complet géré.
3) **Coordination du contrôle** : Coordonner le transfert du contrôle des ressources entre humains et Fays, et entre Fays.
4) **Accès aux ressources par niveaux** : Stratifier l'accès aux ressources en read/write/execute/configure, évitant le motif « une fois autorisé, tout est ouvert ».
5) **Détection de vivacité** : Détection de battement de cœur et récupération par timeout pour empêcher les « sessions zombies » de bloquer les ressources du terminal sur le long terme.

Je l'appelle un « protocole de contrat social » parce qu'il transforme « peux-tu ou ne peux-tu pas faire ceci » d'un indice psychologique de l'UI en une vérification factuelle au niveau du système.

---

## 2. Ce que CAP Ne Fait Pas Explicitement : Capacité, Identité et Intelligence Ne Sont Pas de Son Ressort

Un protocole sain doit savoir ce qu'il ne fait pas ; sinon il devient une colle universelle qui finit par perdre le contrôle.

CAP n'est explicitement pas responsable de :

- La création et la gestion de l'identité de Fay (c'est le système FayID/identité).
- Le raisonnement intelligent et la planification de Fay (c'est la couche Ego/cognitive).
- La logique métier des ressources du terminal (cela appartient au terminal lui-même).
- L'implémentation du transport réseau sous-jacent (WebSocket/gRPC n'importent pas).
- Les mécanismes de sécurité internes du système d'exploitation (le modèle de bac à sable/permissions de l'OS n'est pas remplacé par CAP).

Plus la limite de responsabilité de CAP est claire, plus il peut devenir une infrastructure de gouvernance auditable, plutôt qu'un « patch de sécurité » progressivement érodé par des raisons commerciales.

---

## 3. Pourquoi le « Contrôle » Doit Être Transformé en Protocole : Sinon la Responsabilité Ne Peut Jamais Être Tracée

La forme la plus courante d'autorisation aujourd'hui est :
une appli affiche une boîte de dialogue, tu touches « Autoriser », et ensuite elle a une « permission à long terme ».

Même à l'ère où les humains utilisaient des logiciels, c'était déjà assez mauvais ;
à l'ère où les Fays prennent le contrôle des terminaux, cela devient un véritable désastre.

Parce que les actions d'un Fay ne sont pas un seul clic, mais un comportement continu :
ouvrir une appli, lire des données, former des jugements, exécuter des actions, gérer des exceptions, continuer l'exécution…
Si ce comportement n'a pas de limite claire appelée « session de contrôle », la responsabilité s'évapore avec le temps.

En transformant le contrôle en un protocole, CAP délivre deux valeurs centrales :

1) **Transformer l'autorisation d'une sensation psychologique en un justificatif vérifiable**
2) **Transformer les actions d'opérations dispersées en sessions auditables**

Quand un incident se produit, on peut enfin répondre :
Qui a initié cette session, quand et dans quel cadre autorisé ? Combien de temps la session a-t-elle duré ? Quelles ressources ont été manipulées ? Qui l'a finalement révoquée / qui a fait le timeout et l'a récupérée ?

C'est la condition préalable à la « traçabilité de la responsabilité ».

---

## 4. Hors Ligne d'Abord, En Ligne en Auxiliaire : le Réalisme de CAP

J'approuve fortement le principe de conception central de CAP : **hors ligne d'abord, en ligne en auxiliaire**.

Derrière cela se trouve un jugement réaliste :
Les pannes de réseau ne devraient ni dépouiller les humains du contrôle, ni dépouiller les Fays de la disponibilité sous tutelle.

CAP soutient ce jugement avec deux mécanismes :

- **Autorisation hors ligne (Authorization_Descriptor)** : Fichier chiffré, vérifiable localement.
- **Tickets en ligne (Trusted_Ticket)** : Fournit une révocation en temps réel et un ajustement dynamique lorsqu'il y a réseau.

La force de cette combinaison est la « dégradation gracieuse » :

Avec réseau, tu obtiens une sécurité en temps réel plus forte ;
sans réseau, le système peut continuer à fonctionner sous autorisation vérifiable, au lieu de renvoyer les humains à l'ère manuelle.

Si tu veux qu'iFay devienne une extension humaine, tu dois accepter une réalité :
la vie humaine n'est pas toujours en ligne.

---

## 5. Le Point de Danger de CAP : Plus Il Se Rapproche du Terminal, Plus Il Doit Être Lié à un Système de Tutelle

CAP lui-même n'est pas responsable de la « sémantique de tutelle ».
Il ne répond qu'à « as-tu été autorisé à faire cette chose ».

Mais au moment où tu transformes CAP de « vérification de permission » en « reprise illimitée », il devient un outil de jailbreak.
Donc la conception de CAP doit être liée à un système de tutelle de niveau supérieur :

- **Human View** : Les humains peuvent confirmer, rejouer et intervenir à tout moment.
- **Faying** : Le transfert de contrôle doit être explicite, témoignable et révocable.
- **Rogue Fay** : Lorsque la relation de tutelle ne tient pas, préférer s'arrêter plutôt que d'agir vers l'extérieur.

Je ne m'oppose pas à ce que l'IA prenne le contrôle du terminal.
Ce à quoi je m'oppose, c'est à « une reprise sans point d'imputation de responsabilité ».

Reprendre le terminal n'est pas un jailbreak ; cela devrait ressembler à signer un contrat :
avec des limites claires, auditable, révocable, traçable.

Ce que CAP fait, c'est rendre le « contrat » exécutable au niveau du terminal.

---

## 6. Le Scénario coFay : le Contrôle pour les Rôles Publics Est Plus Sensible

Lorsqu'iFay reprend ton terminal personnel, la responsabilité te revient.
Lorsque coFay reprend des systèmes hospitaliers, des systèmes aéronautiques ou des systèmes gouvernementaux, la responsabilité revient aux organisations et aux postes publics.

Cette différence dicte une chose :
le contrôle pour les rôles publics ne peut pas s'appuyer uniquement sur des « réglementations internes » — il doit être auditable, contestable et explicable à l'extérieur.

Sinon, l'efficacité se transforme directement en boîte noire du pouvoir :

- Pourquoi le triage t'a-t-il mis en bout de file ?
- Pourquoi le contrôle des risques t'a-t-il rejeté ?
- Pourquoi le système éducatif t'a-t-il collé une étiquette ?

Lorsque ces décisions impliquent un coFay, CAP doit garantir :
source d'autorisation claire, limite de session claire, accès aux ressources par niveaux clair, chemin de révocation clair.

C'est la condition de base pour que les services publics restent dignes de confiance à l'ère de l'IA.

---

## 7. Conclusion : la Vraie Difficulté N'est Pas « Peut-Elle Reprendre » mais « Osons-Nous La Laisser Reprendre »

Techniquement, « reprendre le terminal » n'est pas un mystère.
Ce qui est vraiment difficile, c'est : comment faire en sorte que la société ose confier le terminal.

Je crois que la réponse n'est ni un modèle plus puissant ni une UI plus belle, mais :

1) L'IA doit agir sous tutelle humaine ;
2) Le contrôle doit être transformé en protocole ;
3) L'autorisation doit être vérifiable ;
4) Les sessions doivent être auditables ;
5) La révocation doit être atteignable ;
6) La perte de contact doit déclencher une récupération automatique.

CAP porte le maillon le plus sobre dans l'écosystème iFay :
transformer « je t'autorise à faire ceci » en un fait exécutable au niveau du terminal.

Tant que cette ligne minimale n'est pas contournée, l'IA peut survivre dans la société sur le long terme.
Sinon, nous ne faisons qu'accélérer la création de la prochaine panique.

---

## Documents Connexes
- CAP｜Positionnement et Périmètre du Protocole (Français) : https://ifay.ai/fr/docs/Control-Authority-Protocol/blueprint/01-Positionnement-et-Périmètre-du-Protocole
- iFay｜Feuille de Route (EN) : https://ifay.ai/en/docs/iFay/blueprint/04-Roadmap
