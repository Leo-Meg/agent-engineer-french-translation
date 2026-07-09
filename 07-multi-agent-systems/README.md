# Leçon 7 : systèmes multi-agents - quand un seul agent ne suffit pas

## Introduction

Jusqu'ici dans cette formation, nous nous sommes concentrés sur la construction d'agents uniques - un seul LLM avec des outils, une mémoire, et une boucle de planification. Cette approche fonctionne remarquablement bien pour de nombreuses tâches. Mais à mesure que le périmètre de ce que vous voulez automatiser grandit, un agent unique commence à plier sous le poids. Il devient lent. Il se perd. Il essaie d'être expert en tout et finit par n'être expert en rien.

Les systèmes multi-agents résolvent ce problème en répartissant le travail entre plusieurs agents spécialisés qui se coordonnent pour atteindre un objectif commun. Voyez cela comme la différence entre un freelance solo et une équipe bien organisée. Le freelance peut faire beaucoup de choses, mais une équipe de spécialistes - chacun maîtrisant son domaine - peut s'attaquer à des problèmes qu'aucune personne seule ne pourrait gérer.

Dans cette leçon, vous apprendrez quand et pourquoi utiliser plusieurs agents, les principaux patterns architecturaux pour les organiser, comment les agents communiquent, et comment gérer les défis de coordination propres aux systèmes distribués.

## Pourquoi utiliser plusieurs agents ?

### Les limites d'un agent unique

Un agent unique ayant accès à vingt outils, un prompt système énorme, et des instructions couvrant cinq domaines différents, c'est comme un nouvel employé à qui l'on aurait confié cinq fiches de poste différentes dès son premier jour. Il pourrait techniquement être capable d'accomplir chaque tâche, mais jongler avec toutes entraîne des erreurs, des ralentissements, et de la confusion.

Voici les raisons concrètes d'envisager une répartition sur plusieurs agents :

| Problème avec un agent unique | Comment le multi-agent aide |
|---|---|
| Le prompt devient énorme et difficile à maintenir | Chaque agent reçoit un prompt ciblé et gérable |
| Trop d'outils causent des erreurs de sélection | Chaque agent ne voit que les outils pertinents pour son rôle |
| Un échec fait planter tout le workflow | Les échecs sont isolés à un agent ; les autres continuent de fonctionner |
| Difficile de mettre à l'échelle les étapes goulot d'étranglement | Vous pouvez mettre à l'échelle des agents individuels indépendamment |
| Difficile à tester et à déboguer | Chaque agent peut être testé isolément |
| Point de défaillance unique pour la sécurité | Différents agents peuvent avoir différents niveaux de permission |

### ELI5 : l'analogie de l'hôpital

Les systèmes multi-agents, c'est comme un hôpital. Quand vous entrez avec un problème de santé, vous n'obtenez pas un seul médecin qui fait tout - prendre vos constantes, faire une prise de sang, lire la radio, vous opérer, et gérer votre facturation. Au lieu de cela, vous passez à travers un système de spécialistes : une infirmière de triage vous évalue, un médecin vous diagnostique, un radiologue lit vos scans, un chirurgien opère si nécessaire, et un administrateur gère l'assurance. Chaque personne est experte dans son rôle, et ils se coordonnent via des dossiers partagés et des protocoles de transmission.

Un système multi-agent IA fonctionne de la même manière. Chaque agent est un spécialiste avec un rôle ciblé, son propre ensemble d'outils, et des frontières claires sur ce qu'il gère et ce qu'il transmet.

### Rester en agent unique ou passer au multi-agent

Tous les problèmes n'ont pas besoin de plusieurs agents. Voici un cadre de décision simple :

**Restez avec un agent unique quand :**
- La tâche est bien définie et étroite (par ex., « résumer ce document »)
- Vous avez besoin de moins de 5 à 7 outils
- Le workflow est linéaire et prévisible
- Les exigences de latence sont strictes (le multi-agent ajoute du surcoût)

**Envisagez le multi-agent quand :**
- La tâche couvre plusieurs domaines (par ex., service client + paiements + conformité)
- Différentes étapes nécessitent différents niveaux de permission
- Vous voulez développer, tester, et déployer des parties indépendamment
- Le système doit mettre à l'échelle différentes capacités à des rythmes différents
- Vous avez besoin d'une isolation des défaillances - qu'une partie échoue ne devrait pas tout casser

## Agent unique contre multi-agent : la boutique de quartier contre le grand magasin

Imaginez une boutique de quartier tenue par une seule personne. Le propriétaire vous accueille, trouve votre produit, l'encaisse, gère les retours, et réapprovisionne les étagères. Pour une petite boutique avec une poignée de clients, cela fonctionne très bien. Le propriétaire sait tout faire et peut tout faire.

Maintenant, imaginez cette même personne essayant de gérer un grand magasin seule. Des milliers de clients, des dizaines de rayons, un inventaire complexe, des produits spécialisés. Ce serait le chaos. Les grands magasins fonctionnent parce qu'ils ont du personnel spécialisé : quelqu'un au rayon électronique, quelqu'un au rayon vêtements, des caissiers, des chefs de rayon, un comptoir des retours. Chaque personne a un rôle clair et ils se coordonnent via des processus définis.

| Aspect | Agent unique (boutique de quartier) | Multi-agent (grand magasin) |
|---|---|---|
| Gestion de la complexité | Bon pour les tâches simples | Conçu pour les workflows complexes |
| Spécialisation | Touche-à-tout | Expert dans son domaine |
| Impact d'un échec | Tout s'arrête | Seul l'agent concerné est affecté |
| Vitesse de développement | Rapide à prototyper | Plus rapide à itérer sur des parties individuelles |
| Surcoût de coordination | Aucun | Nécessite une logique de coordination explicite |
| Coût par tâche simple | Plus faible | Plus élevé (plus d'appels au LLM) |

## Architectures multi-agents

Il existe quatre patterns principaux pour organiser plusieurs agents. Chacun a des forces différentes, et vous les combinerez souvent dans des systèmes réels.

### 1. Séquentiel (chaîne de montage)

Dans une architecture séquentielle, les agents sont organisés en pipeline. L'agent A termine son travail et le transmet à l'agent B, qui le transmet à l'agent C, et ainsi de suite. Chaque agent transforme ou enrichit la sortie avant de la faire suivre.

**Comment ça marche :**
```
[Agent A] --> [Agent B] --> [Agent C] --> Sortie finale
(Recherche)   (Redaction)   (Relecture)
```

**Analogie :** pensez à une chaîne de montage automobile. Un poste soude le châssis, le suivant installe le moteur, le suivant peint la carrosserie. Chaque poste fait bien une chose, et la voiture avance le long de la chaîne.

**Quand l'utiliser :**
- La tâche a des étapes claires et ordonnées
- Chaque étape dépend de la sortie de la précédente
- Vous voulez un flux simple et prévisible

**Exemple :** un pipeline de contenu où un Agent de recherche rassemble des sources, un Agent de rédaction produit un brouillon, et un Agent de relecture vérifie la qualité et l'exactitude.

**Forces :**
- Facile à comprendre et à déboguer
- Flux de données clair
- Simple d'ajouter ou de retirer des étapes

**Faiblesses :**
- Aussi rapide que l'agent le plus lent
- Aucun parallélisme
- Si un agent en amont fait une erreur, elle se propage en aval

### 2. Hiérarchique (manager et travailleurs)

Dans une architecture hiérarchique, un agent superviseur (le « manager ») reçoit la tâche globale, la décompose en sous-tâches, et délègue chaque sous-tâche à un agent travailleur spécialisé. Le manager collecte les résultats, vérifie la qualité, et peut redéléguer si le travail n'est pas satisfaisant.

**Comment ça marche :**
```
              [Agent Manager]
             /       |       \
  [Travailleur A] [Travailleur B] [Travailleur C]
  (Recherche)     (Calcul)        (Redaction)
```

**Analogie :** c'est comme l'organigramme d'une entreprise. Le PDG n'écrit pas de code ni ne remplit les déclarations fiscales - il fixe la direction et délègue aux chefs de département, qui à leur tour délèguent à leurs équipes. Le PDG relit les résultats et prend les décisions finales.

**Quand l'utiliser :**
- La tâche nécessite une décomposition dynamique (vous ne pouvez pas prédire les sous-tâches à l'avance)
- Différentes sous-tâches nécessitent différentes capacités
- Vous avez besoin d'un point unique de coordination et de contrôle qualité

**Exemple :** un agent de gestion de projet qui reçoit « Planifie l'offsite d'équipe du T3 » et délègue la recherche à un agent, la création du calendrier à un autre, et l'évaluation des risques à un troisième.

**Forces :**
- Flexible - peut gérer des tâches variées
- Contrôle qualité centralisé
- Peut paralléliser les sous-tâches

**Faiblesses :**
- L'agent manager est un goulot d'étranglement et un point de défaillance unique
- Le manager doit être suffisamment intelligent pour bien décomposer les tâches
- Plus complexe à implémenter que le séquentiel

Dans l'[Agent Development Kit (ADK) de Google](https://google.github.io/adk-docs/agents/), les patterns hiérarchiques sont bien pris en charge. Vous pouvez définir un agent parent qui délègue à des sous-agents, chacun avec ses propres outils et instructions.

### 3. Collaboratif (réseau de pairs)

Dans une architecture collaborative, les agents travaillent ensemble en tant qu'égaux. Il n'y a pas de chef. Les agents partagent l'information, s'appuient sur le travail des autres, et convergent vers une solution par la communication.

**Comment ça marche :**
```
[Agent A] <--> [Agent B]
    ^              ^
    |              |
    v              v
[Agent C] <--> [Agent D]
```

**Analogie :** c'est comme une session de brainstorming avec un groupe de collègues. Tout le monde apporte des idées, réagit à ce que les autres ont dit, et le groupe converge vers un plan. Personne n'est aux commandes - les meilleures idées émergent grâce à la discussion.

**Quand l'utiliser :**
- Le problème bénéficie de perspectives diverses
- Aucun agent unique ne possède toute l'information nécessaire
- Vous voulez des sorties créatives ou exploratoires

**Exemple :** un système de relecture de code où un Agent Sécurité, un Agent Performance, et un Agent Lisibilité relisent chacun le même code et partagent leurs conclusions, puis collaborent pour produire une relecture unifiée.

**Forces :**
- Bon pour les problèmes complexes sans décomposition claire
- Des perspectives diverses améliorent la qualité
- Aucun point de défaillance unique

**Faiblesses :**
- Comportement plus difficile à prédire
- Risque de boucles infinies ou de discussion circulaire
- Plus difficile à déboguer

### 4. Compétitif (la meilleure réponse gagne)

Dans une architecture compétitive, plusieurs agents s'attaquent indépendamment au même problème, et un juge (un autre agent ou une fonction de notation) sélectionne la meilleure sortie.

**Comment ça marche :**
```
[Agent A] --\
[Agent B] ----> [Juge] --> Meilleure sortie
[Agent C] --/
```

**Analogie :** c'est comme un concours de design. Trois cabinets d'architecture soumettent chacun une proposition pour un nouveau bâtiment. Un jury examine les trois et sélectionne la meilleure. Le concours produit de meilleurs résultats qu'un seul cabinet n'aurait pu produire seul.

**Quand l'utiliser :**
- La qualité compte plus que le coût
- Le problème a plusieurs approches valides
- Vous voulez réduire le risque d'une mauvaise sortie

**Exemple :** trois agents de codage différents écrivent chacun une solution à un problème de programmation. Un Agent Juge lance les tests, vérifie la qualité du code, et sélectionne la meilleure implémentation.

**Forces :**
- Sorties de meilleure qualité grâce à la compétition
- Naturellement résilient - si un agent échoue, d'autres peuvent réussir
- Bon pour les tâches critiques où les erreurs sont coûteuses

**Faiblesses :**
- Coûteux (N fois le calcul pour N agents)
- Nécessite un bon mécanisme d'évaluation
- Gaspillage si les agents produisent des sorties similaires

### Résumé comparatif des architectures

| Architecture | Flux | Idéal pour | Complexité de coordination |
|---|---|---|---|
| Séquentiel | Pipeline linéaire | Tâches multi-étapes ordonnées | Faible |
| Hiérarchique | Arbre (manager + travailleurs) | Décomposition dynamique de tâches | Moyenne |
| Collaboratif | Maillage (pair-à-pair) | Problèmes complexes nécessitant des apports divers | Élevée |
| Compétitif | Parallèle avec juge | Décisions à fort enjeu | Moyenne |

## Patterns de communication

Les agents doivent se parler entre eux. La façon dont ils communiquent façonne le comportement du système, sa capacité de débogage, et sa performance. Il existe trois patterns principaux :

### La messagerie directe

Les agents s'envoient des messages directement entre eux. L'agent A connaît l'agent B et lui envoie une requête.

```
Agent A --"resume ce document"--> Agent B
Agent B --"voici le resume"--> Agent A
```

**Avantages :** simple, faible latence, facile à tracer.
**Inconvénients :** couplage fort - l'agent A doit connaître l'agent B. Ajouter de nouveaux agents nécessite de mettre à jour les agents existants.

### Le tableau blanc partagé (shared blackboard)

Tous les agents lisent et écrivent dans un espace de travail partagé (le « tableau blanc »). Les agents consultent le tableau à la recherche de nouvelles informations, font leur travail, et republient les résultats.

```
[Tableau blanc / etat partage]
    ^       ^       ^
    |       |       |
Agent A  Agent B  Agent C
```

**Avantages :** couplage faible - les agents n'ont pas besoin de se connaître entre eux. Facile d'ajouter de nouveaux agents. Bon pour les architectures collaboratives.
**Inconvénients :** risque de conflits quand plusieurs agents écrivent dans la même zone. Plus difficile de tracer la causalité (qui a changé quoi et pourquoi).

### Basé sur les événements (event-based)

Les agents publient des événements sur un bus de messages. D'autres agents s'abonnent aux événements qui les concernent et réagissent en conséquence.

```
Agent A --publie "order_refund_requested"--> [Bus d'evenements]
[Bus d'evenements] --notifie--> Agent B (Paiement)
[Bus d'evenements] --notifie--> Agent C (Conformite)
```

**Avantages :** hautement découplé. Passe bien à l'échelle. Familier aux ingénieurs ayant travaillé avec des microservices.
**Inconvénients :** plus d'infrastructure à gérer. Plus difficile à déboguer. Défis de cohérence à terme (eventual consistency).

### Quel pattern choisir

| Scénario | Pattern recommandé |
|---|---|
| Deux agents avec un flux requête-réponse clair | Messagerie directe |
| Plusieurs agents s'appuyant sur un contexte partagé | Tableau blanc partagé |
| Système de type microservices avec de nombreux agents | Basé sur les événements |
| Prototype simple | Messagerie directe |
| Système en production à grande échelle | Basé sur les événements |

## Les rôles d'agent

Dans les systèmes multi-agents bien conçus, chaque agent a un rôle clair. Voici quatre rôles courants que vous verrez dans de nombreuses architectures :

### Le Planificateur (Planner)

Le Planificateur prend un objectif de haut niveau et le décompose en une séquence d'étapes ou de sous-tâches. Il décide de ce qui doit se passer et dans quel ordre.

**Responsabilités :**
- Interpréter l'objectif de l'utilisateur
- Le décomposer en sous-tâches
- Déterminer les dépendances entre sous-tâches
- Créer un plan d'exécution

**Exemple :** face à « Organise un offsite d'équipe le mois prochain », le Planificateur pourrait produire : (1) vérifier les calendriers de l'équipe, (2) trouver des lieux disponibles, (3) comparer les prix, (4) réserver la meilleure option, (5) envoyer les invitations calendrier.

### Le Récupérateur (Retriever)

Le Récupérateur trouve de l'information provenant de sources externes - bases de données, API, documents, le web. Il sait où se trouvent les données et comment les obtenir.

**Responsabilités :**
- Rechercher dans les bases de connaissances et les entrepôts de documents
- Interroger des API et des bases de données
- Filtrer et classer les résultats par pertinence
- Renvoyer une information structurée aux autres agents

### L'Exécuteur (Executor)

L'Exécuteur entreprend des actions dans le monde réel. Il appelle des API, écrit des fichiers, envoie des e-mails, ou modifie des bases de données. C'est l'agent qui « fait les choses ».

**Responsabilités :**
- Exécuter les étapes identifiées par le Planificateur
- Appeler des API et des outils externes
- Gérer les erreurs et les nouvelles tentatives
- Rapporter les résultats

### L'Évaluateur (Evaluator)

L'Évaluateur vérifie la qualité du travail effectué par les autres agents. Il vérifie l'exactitude, la sécurité, et l'exhaustivité.

**Responsabilités :**
- Valider les sorties par rapport aux exigences
- Vérifier les erreurs, les hallucinations, ou les violations de politique
- Noter la qualité et décider si le travail doit être refait
- Fournir un retour pour l'amélioration

## Étude de cas réel : système de remboursement client

Parcourons un système multi-agent concret pour gérer les remboursements client dans une entreprise de e-commerce. Cet exemple utilise une architecture hiérarchique avec quatre agents spécialisés.

### Les agents

| Agent | Rôle | Outils | Permissions |
|---|---|---|---|
| Agent Client | Planificateur + interface | Recherche client, historique des commandes | Lecture des données client |
| Agent Paiement | Exécuteur | API de remboursement, passerelle de paiement | Traiter les remboursements jusqu'à 500 $ |
| Agent Conformité | Évaluateur | Base de données de politiques, détection de fraude | Accès en lecture seule |
| Agent Résolution | Manager / Orchestrateur | Aucun (coordonne les autres) | Délègue à tous les agents |

### Le déroulement

**Scénario :** un client écrit : « Je n'ai jamais reçu ma commande #12345 et je veux un remboursement. »

**Étape 1 : l'Agent Résolution reçoit la demande**

L'Agent Résolution est le point d'entrée. Il lit le message du client et décide quels agents doivent être impliqués.

```
L'Agent Resolution pense :
"C'est une demande de remboursement pour une commande manquante.
Je dois :
1. Verifier le client et les details de la commande
2. Verifier la conformite avec la politique de remboursement
3. Traiter le remboursement si approuve"
```

**Étape 2 : l'Agent Résolution délègue à l'Agent Client**

L'Agent Résolution demande à l'Agent Client de rechercher le client et la commande.

```
Agent Resolution -> Agent Client :
"Recherche la commande #12345 et fournis les details de la commande,
le statut de livraison, et l'historique du client."
```

L'Agent Client interroge la base de données des commandes, trouve que la commande #12345 a été marquée « expédiée » mais que le suivi montre qu'elle n'a jamais été livrée. Il renvoie cette information à l'Agent Résolution.

**Étape 3 : l'Agent Résolution délègue à l'Agent Conformité**

Avec les détails de la commande en main, l'Agent Résolution demande à l'Agent Conformité de vérifier si un remboursement est approprié.

```
Agent Resolution -> Agent Conformite :
"Commande #12345, 89,99 $, expediee mais jamais livree.
Le client a 2 demandes de remboursement anterieures dans l'annee.
Un remboursement est-il approprie selon notre politique ?"
```

L'Agent Conformité vérifie la politique de remboursement, s'assure qu'il ne s'agit pas d'un schéma de fraude, et répond que le remboursement est approuvé - le client est dans les limites de la politique et l'échec de livraison est confirmé par le transporteur.

**Étape 4 : l'Agent Résolution délègue à l'Agent Paiement**

Avec l'approbation de conformité, l'Agent Résolution demande à l'Agent Paiement de traiter le remboursement.

```
Agent Resolution -> Agent Paiement :
"Traite un remboursement de 89,99 $ vers le moyen de paiement
d'origine pour la commande #12345. Conformite approuvee."
```

L'Agent Paiement appelle l'API de la passerelle de paiement, traite le remboursement, et renvoie une confirmation avec l'ID de transaction du remboursement.

**Étape 5 : l'Agent Résolution répond au client**

L'Agent Résolution compile les résultats et génère une réponse destinée au client confirmant le remboursement.

### Pourquoi cette conception fonctionne

- **Séparation des responsabilités :** chaque agent gère un domaine. L'Agent Paiement ne touche jamais aux données client. L'Agent Conformité ne traite jamais de paiements.
- **Sécurité :** l'Agent Paiement a des permissions de remboursement, mais seulement jusqu'à 500 $. Les remboursements plus importants nécessitent une approbation humaine. L'Agent Client peut lire les données mais ne peut pas les modifier.
- **Isolation des défaillances :** si l'API de paiement est en panne, les agents Conformité et Client continuent de fonctionner. Le système peut mettre le remboursement en file d'attente et réessayer plus tard.
- **Testabilité :** vous pouvez tester chaque agent indépendamment. L'Agent Conformité rejette-t-il correctement un remboursement quand le client a trop de réclamations récentes ? Vous pouvez tester cela sans impliquer les paiements du tout.
- **Auditabilité :** chaque délégation et réponse est journalisée. Vous avez une trace claire de qui a décidé quoi et pourquoi.

### Construire cela avec Google ADK

L'[Agent Development Kit (ADK) de Google](https://google.github.io/adk-docs/agents/) fournit une prise en charge intégrée des patterns multi-agents. Vous pouvez définir des agents comme des classes avec leurs propres instructions, outils, et sous-agents. ADK gère le passage de messages entre agents et fournit un traçage pour le débogage.

Pour les agents de workflow qui suivent des patterns prévisibles (comme notre vérification de conformité séquentielle), ADK propose des [agents de workflow](https://google.github.io/adk-docs/agents/workflow-agents/) avec des constructions intégrées de séquence, de parallélisme, et de boucle.

## Défis de coordination

Les systèmes multi-agents sont des systèmes distribués, et les systèmes distribués ont des modes d'échec que les systèmes à agent unique n'ont pas. Voici les défis de coordination les plus courants et comment les gérer :

### Les interblocages (deadlocks)

**Ce que c'est :** deux agents ou plus attendent l'un que l'autre termine, si bien que rien ne progresse.

**Exemple :** l'agent A attend la sortie de l'agent B avant de continuer. L'agent B attend la sortie de l'agent A avant de continuer. Aucun des deux ne peut avancer.

**Comment le prévenir :**
- Concevoir des flux de données unidirectionnels quand c'est possible
- Ajouter des timeouts à toute communication agent-à-agent
- Utiliser un orchestrateur central pour détecter et briser les cycles
- Implémenter des circuit breakers qui échouent avec élégance après un timeout

### La délégation circulaire

**Ce que c'est :** l'agent A délègue à l'agent B, qui délègue de nouveau à l'agent A, créant une boucle infinie.

**Exemple :** un Agent Planificateur demande de l'information à un Agent Recherche. L'Agent Recherche décide qu'il a besoin de plus de contexte et demande à l'Agent Planificateur de clarifier. L'Agent Planificateur, n'ayant pas de nouvelle information, redemande à l'Agent Recherche. Boucle infinie.

**Comment le prévenir :**
- Fixer une profondeur maximale de délégation (par ex., pas plus de 3 transmissions pour une même tâche)
- Suivre l'historique de délégation et rejeter les requêtes qui créent des cycles
- Donner aux agents des frontières claires sur ce qu'ils doivent gérer contre ce qu'ils doivent escalader à un humain

### Les actions conflictuelles

**Ce que c'est :** deux agents entreprennent indépendamment des actions contradictoires sur la même ressource.

**Exemple :** un Agent Tarification fixe le prix d'un produit à 49,99 $ sur la base d'une analyse concurrentielle. Simultanément, un Agent Promotions fixe le prix du même produit à 29,99 $ pour une vente flash. Le prix final dépend de quel agent a écrit en dernier.

**Comment le prévenir :**
- Utiliser des mécanismes de verrouillage pour les ressources partagées
- Désigner un seul agent comme propriétaire de chaque ressource
- Implémenter un agent ou une politique de résolution de conflits
- Utiliser l'event sourcing (traçabilité par événements) afin que tous les changements soient suivis et réversibles

### La contention de ressources

**Ce que c'est :** plusieurs agents se disputent des ressources limitées (limites de débit d'API, budgets de tokens, connexions de base de données).

**Exemple :** dix agents essaient tous d'appeler la même API externe simultanément, atteignant la limite de débit et causant des échecs pour tous.

**Comment le prévenir :**
- Implémenter une limitation de débit et une mise en file d'attente au niveau du système
- Utiliser un pool de ressources partagées avec un ordonnancement équitable
- Donner aux agents critiques une priorité plus élevée pour les ressources partagées
- Surveiller l'utilisation des ressources et fixer des budgets par agent

### L'état incohérent

**Ce que c'est :** les agents ont des vues différentes du monde parce que les mises à jour de l'état partagé ne se sont pas propagées à tous.

**Exemple :** l'Agent Client vérifie l'inventaire et dit au client qu'un article est en stock. Pendant ce temps, l'Agent Exécution des commandes vient de vendre la dernière unité. Le client reçoit une confirmation pour un article qui n'est plus disponible.

**Comment le prévenir :**
- Utiliser une source unique de vérité pour l'état partagé
- Implémenter un verrouillage optimiste avec des numéros de version
- Concevoir les agents pour gérer les données obsolètes avec élégance (vérifier avant d'agir)
- Garder la fenêtre d'incohérence aussi petite que possible

## Principes de conception pour les systèmes multi-agents

Sur la base des patterns et défis ci-dessus, voici les principes clés à suivre :

### 1. Commencer simple

Commencez avec un agent unique. N'ajoutez d'autres agents que lorsque vous rencontrez des limitations claires. Une décomposition prématurée en plusieurs agents ajoute de la complexité sans bénéfice.

### 2. Définir des frontières claires

Chaque agent doit avoir un périmètre bien défini, ses propres outils, et des règles claires sur ce qu'il gère et ce qu'il transmet aux autres. Des frontières floues entraînent des doublons et des conflits.

### 3. Concevoir pour l'échec

Supposez que n'importe quel agent peut échouer à tout moment. Utilisez des timeouts, des nouvelles tentatives, des circuit breakers, et des solutions de repli (fallbacks). Un système multi-agent bien conçu se dégrade avec élégance plutôt que de s'effondrer entièrement.

### 4. Rendre la communication observable

Journalisez chaque message entre agents. Vous aurez besoin de ces journaux pour déboguer les problèmes. Si vous ne pouvez pas tracer le chemin complet d'une requête à travers votre système, vous ne pourrez pas la réparer quand elle échoue.

### 5. Limiter l'autonomie des agents

Chaque agent devrait avoir le minimum de permissions dont il a besoin. L'Agent Paiement ne devrait pas pouvoir lire les e-mails des clients. L'Agent Client ne devrait pas pouvoir traiter des remboursements. Cela limite le rayon d'impact quand un agent se comporte mal.

### 6. Utiliser des humains comme circuit breakers

Pour les décisions à fort enjeu, incluez une étape d'intervention humaine (human-in-the-loop). Les agents sont bons pour gérer les cas de routine. Les humains devraient gérer les exceptions, les cas limites, et les décisions aux conséquences importantes.

## Exercice pratique

Concevez un système multi-agent pour l'un des scénarios suivants. Pour le scénario choisi, définissez :
1. Les agents et leurs rôles
2. Le pattern architectural (séquentiel, hiérarchique, collaboratif, ou compétitif)
3. Le pattern de communication (direct, tableau blanc, ou basé sur les événements)
4. Au moins deux défis de coordination potentiels et vos stratégies d'atténuation

**Scénario A : relecture automatisée de code**
Un système qui relit les pull requests pour la qualité du code, les vulnérabilités de sécurité, les problèmes de performance, et la conformité au style.

**Scénario B : assistant de réservation de voyage**
Un système qui aide les utilisateurs à planifier et réserver un voyage - vols, hôtels, locations de voiture, et activités - tout en respectant un budget.

**Scénario C : pipeline de modération de contenu**
Un système qui relit le contenu généré par les utilisateurs à la recherche de violations de politique, de spam, de désinformation, et de contenu nuisible avant publication.

## Points clés à retenir

- Les systèmes multi-agents répartissent le travail entre agents spécialisés, chacun avec des responsabilités, des outils, et des permissions ciblés.
- Il existe quatre patterns architecturaux principaux : séquentiel (pipeline), hiérarchique (manager-travailleur), collaboratif (réseau de pairs), et compétitif (la meilleure réponse gagne). Choisissez selon la structure de votre tâche.
- Les patterns de communication - messagerie directe, tableau blanc partagé, et basé sur les événements - déterminent à quel point vos agents sont étroitement couplés.
- Les rôles d'agent courants (Planificateur, Récupérateur, Exécuteur, Évaluateur) fournissent un vocabulaire de départ pour concevoir votre système.
- Les défis de coordination comme les interblocages, la délégation circulaire, et les actions conflictuelles sont les véritables problèmes d'ingénierie des systèmes multi-agents. Concevez-les dès le départ.
- Commencez avec un agent unique et n'ajoutez de la complexité que lorsque vous en avez besoin. Le meilleur système multi-agent est le plus simple qui résout votre problème.

## Pour aller plus loin

- [Google ADK - Présentation des agents](https://google.github.io/adk-docs/agents/) - Comment construire des agents et des systèmes multi-agents avec l'Agent Development Kit
- [Google ADK - Agents de workflow](https://google.github.io/adk-docs/agents/workflow-agents/) - Patterns intégrés de séquence, parallélisme, et boucle pour les workflows multi-agents
