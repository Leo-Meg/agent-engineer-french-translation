# Leçon 6 : planification et raisonnement - comment les agents abordent les tâches complexes

## Ce que vous allez apprendre

- Pourquoi la planification est essentielle pour les agents qui traitent des tâches complexes
- La boucle de résolution de problèmes agentique : recevoir la mission, examiner la scène, penser, agir, observer
- Le plan-puis-exécute contre les approches réactives, et leurs compromis respectifs
- La planification hiérarchique pour décomposer de grandes tâches en éléments gérables
- La replanification : comment les agents s'adaptent quand les choses tournent mal
- Les techniques de raisonnement : Chain-of-Thought, Tree-of-Thoughts, et l'autocohérence
- Le rôle de la couche d'orchestration dans la gestion des plans
- Les modes d'échec courants et comment les éviter

## Prérequis

- [Leçon 2 : Comment pensent les agents](../02-how-agents-think/README.md)
- [Leçon 4 : Design patterns agentiques](../04-agentic-design-patterns/README.md)
- [Leçon 5 : Mémoire et contexte](../05-memory-and-context/README.md)

---

## ELI5 : la planification, c'est comme faire sa valise pour un voyage

Imaginez que vous fassiez votre valise pour un voyage de deux semaines. Vous pourriez simplement attraper des affaires et les fourrer dans la valise. Peut-être aurez-vous de la chance et aurez-vous tout ce qu'il faut. Plus probablement, vous oublierez votre brosse à dents et emporterez trois vestes que vous ne porterez jamais.

Une meilleure approche : réfléchissez à votre destination, au temps qu'il fera, aux activités prévues, et *ensuite* faites une liste. Cochez les éléments au fur et à mesure que vous les emballez. Si vous découvrez à mi-parcours que vous manquez de place, vous reprioritisez - vous abandonnez les affaires « au cas où » et gardez l'essentiel.

C'est ce que la planification fait pour un agent IA. Plutôt que de réagir aveuglément à chaque étape, l'agent réfléchit à l'avance, établit un plan, et le suit méthodiquement. Et quand le plan rencontre un obstacle, un bon agent s'ajuste plutôt que de poursuivre avec une approche défaillante.

---

## Pourquoi la planification compte

Les tâches simples n'ont pas besoin de planification. Si quelqu'un demande « Quelle heure est-il ? », l'agent consulte simplement l'horloge. Aucun plan n'est nécessaire.

Mais considérez une demande comme : « Recherche les 5 principaux concurrents sur le marché des bases de données cloud, compare leurs tarifs et fonctionnalités, et rédige une recommandation sur celui que nous devrions utiliser pour notre nouveau projet. »

Cette tâche nécessite :
- D'identifier qui sont les concurrents
- De faire des recherches sur chacun (tarifs, fonctionnalités, limitations)
- De comprendre les exigences du projet de l'utilisateur
- De comparer les options par rapport à ces exigences
- De synthétiser le tout en une recommandation

Sans planification, l'agent pourrait commencer à rechercher un concurrent avec un luxe de détails extrême, manquer d'espace de contexte, et oublier de couvrir les quatre autres. Ou il pourrait sauter directement à une recommandation sans faire de recherche approfondie.

La planification donne à l'agent une feuille de route. Il sait où il va, quelles sont les prochaines étapes, et comment répartir son temps et ses ressources.

### Ce que la planification vous apporte

| Bénéfice | Sans planification | Avec planification |
|---------|-----------------|---------------|
| **Achèvement de la tâche** | Peut manquer des étapes ou se bloquer | Couverture systématique de toutes les étapes |
| **Efficacité des ressources** | Gaspille des tokens sur des digressions | Alloue l'effort là où il compte |
| **Transparence** | Difficile de voir ce que fait l'agent | Suivi clair de la progression |
| **Récupération après erreur** | Se perd quand les choses tournent mal | Peut identifier où le plan s'est rompu et s'ajuster |
| **Parallélisme** | Tout est séquentiel | Les étapes indépendantes peuvent s'exécuter simultanément |

---

## La boucle de résolution de problèmes agentique

Au cœur de tout agent planificateur se trouve une boucle de résolution de problèmes. Différents frameworks la décrivent différemment, mais les étapes centrales restent cohérentes :

### Recevoir la mission -> examiner la scène -> penser -> agir -> observer

```
+-------------------------------------------------------------------+
|                                                                   |
|   +------------+     +------------+     +---------+               |
|   | Recevoir la|     | Examiner   |     | Penser  |               |
|   | mission    | --> | la scene   | --> |         |               |
|   +------------+     +------------+     +----+----+               |
|                                              |                    |
|                                              v                    |
|                        +---------+     +-----+-----+              |
|                        | Observer| <-- |    Agir    |             |
|                        +----+----+     +-----------+              |
|                             |                                     |
|                             v                                     |
|                      +------+------+                              |
|                      | Objectif    |                              |
|                      | atteint ?   |                              |
|                      | Oui -> Fini |                              |
|                      | Non -> Pense|                              |
|                      +-------------+                              |
|                                                                   |
+-------------------------------------------------------------------+
```

### Étape par étape

**1. Recevoir la mission**

L'agent reçoit sa tâche. Cela peut être une demande utilisateur, un événement déclencheur, ou une sous-tâche déléguée par un autre agent.

```
Mission : "Cree un rapport de synthese de l'activite GitHub de notre
          equipe cette semaine."
```

**2. Examiner la scène**

L'agent rassemble du contexte sur sa situation actuelle. Quels outils sont disponibles ? Quelles informations possède-t-il déjà ? Quelles contraintes existent ?

```
Outils disponibles : github_api, document_writer, calendar_api
Contexte connu : equipe = "platform-eng", date actuelle = 15 mars
Contraintes : le rapport doit tenir sur moins de 2 pages
```

**3. Penser**

L'agent raisonne sur la prochaine action à entreprendre. C'est ici que se déroule la planification - l'agent examine ses options et décide d'un plan d'action.

```
Pensee : "Je dois :
  1. Obtenir la liste des membres de l'equipe
  2. Pour chaque membre, recuperer ses commits, PR et relectures
     de cette semaine
  3. Agreger les donnees
  4. Rediger une synthese mettant en avant les contributions cles
  Je vais commencer par obtenir la liste des membres de l'equipe."
```

**4. Agir**

L'agent exécute une action - appeler un outil, générer du texte, ou prendre une décision.

```
Action : github_api.get_team_members(team="platform-eng")
```

**5. Observer**

L'agent examine le résultat de son action. A-t-elle réussi ? Quelle information a-t-elle fournie ? Le plan doit-il changer ?

```
Observation : l'equipe compte 8 membres : [alice, bob, carol, dave,
              eve, frank, grace, hank]
```

La boucle revient ensuite à **Penser**, où l'agent décide de la prochaine étape en fonction de ce qu'il a observé. Cela continue jusqu'à ce que la mission soit accomplie.

---

## Plan-puis-exécute contre approches réactives

Il existe deux philosophies fondamentales sur la façon dont les agents abordent les tâches. La plupart des agents réels mélangent les deux, mais comprendre les formes pures vous aide à prendre des décisions de conception.

### Plan-puis-exécute

**Comment ça marche :** l'agent crée un plan complet (ou presque complet) avant d'entreprendre la moindre action. Puis il exécute le plan étape par étape.

```
Utilisateur : "Fais migrer notre application de Python 2 vers Python 3."

Phase de planification :
  Etape 1 : auditer la codebase pour la syntaxe specifique a Python 2
  Etape 2 : identifier les dependances incompatibles
  Etape 3 : mettre a jour les dependances vers des versions
           compatibles Python 3
  Etape 4 : convertir les instructions print en fonctions print
  Etape 5 : corriger la gestion des chaines/unicode
  Etape 6 : mettre a jour les operateurs de division entiere
  Etape 7 : lancer la suite de tests et corriger les echecs
  Etape 8 : mettre a jour le pipeline CI/CD pour utiliser Python 3

Phase d'execution :
  [execute les etapes 1 a 8 dans l'ordre]
```

**Forces :**
- Structure claire et suivi de progression
- Peut identifier les dépendances entre étapes en amont
- Permet l'exécution parallèle des étapes indépendantes
- Facile d'estimer l'effort total et de communiquer la progression

**Faiblesses :**
- Le plan peut être erroné ou incomplet (on ne sait pas ce qu'on ne sait pas)
- Rigide - difficile de s'adapter aux surprises
- La planification en amont prend du temps et des tokens
- Peut mener à une sur-planification pour des tâches simples

### L'approche réactive

**Comment ça marche :** l'agent avance étape par étape. Il examine la situation actuelle, décide de la meilleure action suivante, l'entreprend, et réévalue. C'est essentiellement le pattern ReAct de la [leçon 4](../04-agentic-design-patterns/README.md).

```
Utilisateur : "Fais migrer notre application de Python 2 vers Python 3."

Etape 1 :
  Penser : "D'abord, je devrais comprendre la codebase. Recherchons
           la syntaxe Python 2."
  Agir : rechercher les patterns Python 2
  Observer : 47 instructions print, 12 problemes unicode,
            3 imports obsoletes trouves

Etape 2 :
  Penser : "Les instructions print sont le probleme le plus courant.
           Commencons par la."
  Agir : convertir les instructions print
  Observer : 47 instructions print converties

Etape 3 :
  Penser : "Maintenant, attaquons-nous aux problemes unicode..."
  [continue etape par etape]
```

**Forces :**
- Hautement adaptative - répond à ce qu'elle trouve réellement
- Aucun effort de planification gaspillé
- Fonctionne bien pour les tâches exploratoires
- Gère naturellement les surprises

**Faiblesses :**
- Aucune vue d'ensemble de la progression
- Peut manquer des étapes importantes qu'une planification en amont aurait détectées
- Difficile à paralléliser - chaque étape dépend de l'observation précédente
- Peut dévier de sa trajectoire sans plan directeur

### Le juste milieu pragmatique

La plupart des agents en production utilisent une approche hybride :

1. **Une planification légère en amont :** créer un plan approximatif et de haut niveau (3 à 7 étapes)
2. **Une exécution réactive :** utiliser un raisonnement de type ReAct au sein de chaque étape
3. **Une replanification périodique :** après les étapes majeures, réévaluer et ajuster le plan

```
Plan de haut niveau (leger) :
  1. Auditer la codebase
  2. Corriger les problemes de syntaxe
  3. Mettre a jour les dependances
  4. Lancer les tests et corriger les echecs

Execution de l'etape 1 (reactive) :
  Penser -> Agir -> Observer -> Penser -> Agir -> Observer -> Etape terminee

Replanification apres l'etape 1 :
  "L'audit a revele plus de problemes que prevu. Ajout de l'etape 2b
   pour le code de migration de base de donnees qui utilise une
   bibliotheque obsolete."
```

Cela vous donne la structure de la planification avec l'adaptabilité de l'exécution réactive.

### Quand privilégier chaque approche

| Situation | Privilégier la planification | Privilégier l'approche réactive |
|-----------|---------------|----------------|
| La tâche est bien comprise | Oui | |
| La tâche est exploratoire | | Oui |
| Plusieurs sous-tâches indépendantes | Oui | |
| Forte incertitude sur ce que l'on va trouver | | Oui |
| Besoin de communiquer la progression | Oui | |
| La vitesse compte plus que l'exhaustivité | | Oui |
| L'échec est coûteux | Oui | |
| La tâche est simple (moins de 3 étapes) | | Oui |

---

## La planification hiérarchique

### Décomposer de grandes tâches en tâches plus petites

Certaines tâches sont trop complexes pour être planifiées avec une simple liste linéaire. La planification hiérarchique décompose un grand objectif en sous-objectifs, eux-mêmes décomposés en sous-sous-objectifs - comme une structure de découpage de projet (work breakdown structure) en gestion de projet.

### L'analogie de la gestion de projet

Pensez à la façon dont un projet logiciel est organisé :

```
Epic : Lancer un nouveau systeme de paiement
  |
  +-- Story : Concevoir l'API de paiement
  |     +-- Tache : Definir les endpoints
  |     +-- Tache : Concevoir les modeles de donnees
  |     +-- Tache : Rediger la specification de l'API
  |
  +-- Story : Implementer le traitement des paiements
  |     +-- Tache : Integrer avec la passerelle de paiement
  |     +-- Tache : Gerer les cas d'erreur
  |     +-- Tache : Ajouter une logique de nouvelles tentatives
  |
  +-- Story : Ajouter le monitoring et les alertes
        +-- Tache : Mettre en place la journalisation
        +-- Tache : Definir les SLO
        +-- Tache : Configurer les regles d'alerte
```

Un agent utilise la même structure. Par exemple, « Écrire un article de blog sur la mise en réseau Kubernetes » se décompose en :

```
Sous-objectif 1 : Recherche
  Tache 1.1 : identifier les concepts reseau cles
  Tache 1.2 : trouver les developpements recents et les bonnes pratiques

Sous-objectif 2 : Plan et redaction
  Tache 2.1 : creer la structure des sections
  Tache 2.2 : rediger chaque section avec des exemples de code

Sous-objectif 3 : Relecture
  Tache 3.1 : verifier l'exactitude technique
  Tache 3.2 : verifier que les exemples de code fonctionnent
```

### Avantages de la planification hiérarchique

- **Des blocs gérables.** Chaque tâche est suffisamment petite pour être exécutée sans perdre le fil.
- **Des dépendances claires.** Vous pouvez voir quelles tâches dépendent d'autres et lesquelles peuvent s'exécuter en parallèle.
- **Le suivi de progression.** Vous savez exactement où vous en êtes dans le plan global.
- **La délégation.** Dans les systèmes multi-agents, différents sous-objectifs peuvent être assignés à des agents spécialisés.

### Niveaux d'abstraction

| Niveau | Ce qu'il décrit | Exemple |
|-------|------------------|---------|
| **Objectif** | À quoi ressemble le succès | « Déployer l'application en production » |
| **Sous-objectif** | Les phases majeures du travail | « Préparer l'environnement » |
| **Tâche** | Des actions spécifiques | « Créer le service Cloud Run » |
| **Étape** | Des opérations atomiques | « Exécuter `gcloud run deploy` » |

Les agents planifient généralement au niveau du sous-objectif et de la tâche. Les étapes sont gérées par une exécution de type ReAct au sein de chaque tâche.

---

## Replanification : s'adapter quand les choses tournent mal

Aucun plan ne survit au premier contact avec la réalité. Les bons agents détectent quand un plan échoue et s'ajustent.

### Quand replanifier

| Déclencheur | Exemple | Réponse |
|---------|---------|---------|
| **Échec de tâche** | L'API renvoie une erreur | Essayer une approche alternative ou passer et revenir plus tard |
| **Nouvelle information** | Découverte que la base de données utilise un schéma différent de celui attendu | Mettre à jour le plan pour tenir compte du schéma réel |
| **Exigences modifiées** | L'utilisateur ajoute une nouvelle exigence en cours de tâche | Intégrer la nouvelle exigence au plan |
| **Contraintes de ressources** | L'espace de contexte ou le quota d'API s'épuise | Simplifier les étapes restantes |
| **Dépendance bloquée** | Un service requis est en panne | Réordonner les tâches pour travailler d'abord sur les éléments non bloqués |

### Stratégies de replanification

**1. L'ajustement local**

Corriger le problème immédiat sans changer le plan global.

```
Etape du plan original : "Interroger la table users pour les comptes actifs"
Echec : "La table 'users' n'existe pas"
Ajustement : "Interroger plutot la table 'accounts' (elle contient
             les memes donnees)"
```

**2. L'insertion d'étapes**

Ajouter de nouvelles étapes pour gérer une situation inattendue.

```
Plan original :
  1. Recuperer les donnees
  2. Traiter les donnees
  3. Generer le rapport

Apres avoir decouvert que les donnees necessitent un nettoyage :
  1. Recuperer les donnees
  1b. Nettoyer et valider les donnees    <-- insere
  2. Traiter les donnees
  3. Generer le rapport
```

**3. La révision du plan**

Restructurer significativement le plan restant en fonction de ce que l'on a appris.

```
Plan original : "Faire migrer le monolithe vers des microservices"
Apres analyse : "La codebase est trop fortement couplee pour une
                migration directe."
Plan revise : "D'abord, introduire des frontieres de modules au sein
              du monolithe (pattern strangler fig), puis extraire
              progressivement les services."
```

**4. La modification de l'objectif**

Dans les cas extrêmes, l'agent reconnaît que l'objectif original n'est pas atteignable et propose un objectif révisé. Par exemple, si la cible était « réduire la latence à moins de 50 ms » mais que la requête de base de données seule prend 80 ms, l'agent pourrait proposer de viser 100 ms avec de la mise en cache pour les endpoints fréquemment consultés.

Le principe clé : essayez d'abord un correctif local, et ne restructurez le plan restant que si le correctif local n'est pas possible.

---

## Techniques de raisonnement

La planification nécessite du raisonnement - la capacité de réfléchir logiquement aux problèmes. Plusieurs techniques aident les agents à raisonner plus efficacement.

### Chain-of-Thought (CoT)

**Ce que c'est :** l'agent travaille un problème étape par étape, en montrant son raisonnement à chaque étape. Plutôt que de sauter directement à une réponse, il « pense à voix haute ».

**Analogie :** montrer son travail lors d'un contrôle de mathématiques. Plutôt que d'écrire seulement la réponse, vous écrivez chaque étape du calcul.

```
Sans Chain-of-Thought :
  Q : "Si un serveur traite 100 requetes/seconde et que nous devons en
      traiter 10 000/seconde, combien de serveurs nous faut-il ?"
  R : "100 serveurs"

Avec Chain-of-Thought :
  Q : [meme question]
  R : "Travaillons cela :
      - Un serveur traite 100 requetes/seconde
      - Nous avons besoin de 10 000 requetes/seconde au total
      - 10 000 / 100 = 100 serveurs pour la capacite brute
      - Mais nous devrions tenir compte de la redondance. Avec une
        redondance N+1, il nous en faut 101.
      - Et les serveurs ne fonctionnent pas a 100% d'efficacite. A un
        taux d'utilisation cible de 80% :
        10 000 / (100 * 0.8) = 125 serveurs
      - Avec une redondance N+1 : 126 serveurs
      Reponse : 126 serveurs pour la production, en supposant un taux
                d'utilisation cible de 80% et une redondance N+1."
```

Le raisonnement étape par étape détecte des nuances (redondance, utilisation) qu'une réponse instantanée aurait manquées.

**Quand utiliser le CoT :**
- Problèmes de mathématiques et de logique
- Raisonnement multi-étapes
- Tâches où les étapes intermédiaires comptent
- Débogage et analyse de cause racine

### Tree-of-Thoughts (ToT)

**Ce que c'est :** plutôt que de suivre une seule chaîne de raisonnement, l'agent explore plusieurs chemins possibles et évalue lequel est le plus prometteur. Voyez cela comme un brainstorming de plusieurs approches avant de s'engager sur l'une d'elles.

**Analogie :** un joueur d'échecs qui envisage plusieurs coups possibles, réfléchit quelques coups à l'avance pour chacun, puis choisit le meilleur chemin.

```
Probleme : "Notre API expire (timeout) sous forte charge. Comment
           devrions-nous la corriger ?"

Branche 1 : ajouter de la mise en cache
  -> Evaluation : "Reduirait la charge de la base de donnees d'environ
     60%. L'implementation prend 2 jours. Risque : complexite de
     l'invalidation du cache."
  Score : 7/10

Branche 2 : optimiser les requetes de base de donnees
  -> Evaluation : "Pourrait ameliorer le temps de requete d'environ
     40%. L'implementation prend 3 jours. Risque : faible, approche
     bien maitrisee."
  Score : 6/10

Branche 3 : ajouter de la mise a l'echelle horizontale
  -> Evaluation : "Gere n'importe quel niveau de charge. L'implementation
     prend 1 jour avec Kubernetes. Risque : augmente le cout
     d'infrastructure."
  Score : 8/10

Decision : commencer par la mise a l'echelle horizontale (gain rapide),
          puis ajouter la mise en cache pour l'efficacite a long terme.
```

**Quand utiliser le ToT :**
- Problèmes avec plusieurs approches valides
- Décisions stratégiques
- Tâches où la première idée n'est peut-être pas la meilleure
- Décisions d'architecture et de conception

### L'autocohérence (self-consistency)

**Ce que c'est :** l'agent résout le même problème plusieurs fois en utilisant différents chemins de raisonnement, puis vérifie si les réponses concordent. Si la plupart des chemins mènent à la même conclusion, la confiance est élevée. S'ils divergent, une investigation plus approfondie est nécessaire.

**Analogie :** demander à trois mécaniciens différents leur avis sur un problème de voiture. S'ils disent tous « alternateur défectueux », vous pouvez avoir confiance. S'ils disent chacun quelque chose de différent, il vous faut plus de diagnostics.

```
Probleme : "Ce changement de code est-il sûr a deployer un vendredi ?"

Chemin de raisonnement 1 (analyse de risque) :
  "Le changement modifie le flux de paiement. Des changements de
   paiement un vendredi signifient des incidents le week-end.
   Verdict : Non."

Chemin de raisonnement 2 (analyse de perimetre) :
  "Le changement fait 5 lignes et a une couverture de tests de 95%.
   Les changements petits et bien testes sont a faible risque.
   Verdict : Oui, avec du monitoring."

Chemin de raisonnement 3 (analyse historique) :
  "Des changements similaires ont cause des problemes 15% du temps
   par le passe. C'est au-dessus de notre seuil de 10%.
   Verdict : Non."

Consensus : 2 sur 3 disent Non. Recommandation : attendre lundi.
```

**Quand utiliser l'autocohérence :**
- Décisions à fort enjeu
- Tâches où les erreurs sont coûteuses
- Problèmes ambigus sans réponse unique claire
- Validation d'un raisonnement critique

### Comparer les techniques de raisonnement

| Technique | Approche | Force | Coût | Idéal pour |
|-----------|----------|----------|------|----------|
| **Chain-of-Thought** | Linéaire, étape par étape | Exhaustivité | 1x (une seule passe) | La plupart des tâches de raisonnement |
| **Tree-of-Thoughts** | Explore plusieurs chemins | Trouve la meilleure approche | 3-5x (plusieurs branches) | Décisions stratégiques |
| **Autocohérence** | Plusieurs tentatives indépendantes | Calibration de la confiance | 3-5x (plusieurs passes) | Validation à fort enjeu |

---

## La couche d'orchestration

La couche d'orchestration est le système de contrôle qui gère comment un agent planifie, raisonne, et exécute. Voyez-la comme le chef d'orchestre - il ne joue d'aucun instrument, mais il décide qui joue quoi et quand.

### Ce que fait la couche d'orchestration

```
+----------------------------------------------------------+
|                COUCHE D'ORCHESTRATION                    |
|                                                          |
|  +----------+  +-----------+  +----------+  +--------+  |
|  |Gestionnaire| Moteur     |  |Monitoring|  |Logique |  |
|  |de plan   |  |d'execution|  |& evaluation| replanif.|  |
|  +----------+  +-----------+  +----------+  +--------+  |
|                                                          |
|  Responsabilites :                                       |
|  - Decomposer les objectifs en etapes executables        |
|  - Decider de l'ordre d'execution et du parallelisme      |
|  - Router les etapes vers les bons outils ou sous-agents |
|  - Suivre la progression et l'etat                       |
|  - Detecter les echecs et declencher la replanification  |
|  - Gerer l'utilisation de la context window              |
|  - Appliquer les guardrails et les controles de securite |
+----------------------------------------------------------+
```

### L'orchestration en pratique

Dans le [Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview) de Google Cloud, la couche d'orchestration gère :

- **Le routage :** diriger les requêtes utilisateur vers le bon agent ou outil
- **La gestion d'état :** suivre où en est l'agent dans son plan
- **L'exécution d'outils :** appeler les outils et traiter les résultats
- **La gestion des erreurs :** capturer les échecs et décider comment récupérer
- **La gestion du contexte :** garder la context window organisée

L'[Agent Development Kit (ADK)](https://google.github.io/adk-docs/) vous donne les briques pour personnaliser le comportement d'orchestration. Vous définissez les outils, les instructions, et le comportement de l'agent - le framework gère la boucle d'exécution.

### Décisions d'orchestration clés

| Décision | Options | Compromis |
|----------|---------|-----------|
| **Combien planifier en amont** | Plan complet contre étape suivante uniquement | Exhaustivité contre flexibilité |
| **Quand replanifier** | Après chaque étape contre seulement en cas d'échec | Adaptabilité contre surcoût |
| **Comment gérer les échecs** | Nouvelle tentative, passer, abandonner, ou replanifier | Résilience contre coût |
| **Séquentiel contre parallèle** | Une étape à la fois contre concurrent | Simplicité contre vitesse |
| **Combien de raisonnement montrer** | Interne uniquement contre exposé à l'utilisateur | Transparence contre bruit |

---

## Modes d'échec courants

Même des agents planificateurs bien conçus peuvent échouer de manières prévisibles. Connaître ces modes d'échec vous aide à construire des défenses contre eux.

### 1. Les boucles infinies

**Ce qui se passe :** l'agent se retrouve bloqué à répéter la même action ou à replanifier indéfiniment sans progresser.

**Exemple :**
```
Penser : "Je dois trouver l'e-mail de l'utilisateur. Cherchons dans
         la base de donnees."
Agir : search_database(query="user email")
Observer : aucun resultat trouve.
Penser : "Je dois trouver l'e-mail de l'utilisateur. Cherchons dans
         la base de donnees."
Agir : search_database(query="user email")
Observer : aucun resultat trouve.
[se repete indefiniment]
```

**Prévention :**
- Fixer un nombre maximal d'itérations pour toute boucle
- Suivre les actions entreprises et détecter la répétition
- Après N tentatives échouées de la même action, forcer l'agent à essayer une approche différente
- Inclure une option « abandonner avec élégance » - demander de l'aide à l'utilisateur

### 2. La dérive du plan

**Ce qui se passe :** l'agent s'éloigne progressivement de l'objectif original, suivant des digressions intéressantes au lieu de rester sur la bonne voie.

**Exemple :**
```
Objectif original : "Ecrire une synthese de la performance des ventes du T4"
Etape 1 : recuperer les donnees de ventes du T4 [sur la bonne voie]
Etape 2 : remarquer une anomalie dans les donnees de novembre
          [legerement hors piste]
Etape 3 : approfondir l'anomalie de novembre [en train de deriver]
Etape 4 : rechercher les tendances du secteur qui pourraient
          expliquer l'anomalie [perdu]
Etape 5 : ecrire un rapport sur les tendances du secteur
          [completement hors sujet]
```

**Prévention :**
- Vérifier périodiquement : « Ce que je fais est-il aligné avec l'objectif original ? »
- Inclure l'objectif original dans chaque étape de raisonnement
- Fixer des limites de périmètre dans le plan
- Utiliser une étape d'évaluation séparée pour vérifier la pertinence

### 3. La sur-planification de tâches simples

**Ce qui se passe :** l'agent passe plus de temps à planifier qu'il n'en faudrait pour simplement accomplir la tâche.

**Exemple :**
```
Utilisateur : "Combien font 2 + 2 ?"

Plan de l'agent :
  Etape 1 : identifier l'operation mathematique (addition)
  Etape 2 : identifier les operandes (2 et 2)
  Etape 3 : verifier que les operandes sont des nombres valides
  Etape 4 : effectuer l'addition
  Etape 5 : verifier le resultat
  Etape 6 : mettre en forme la reponse

[Dis simplement "4" !]
```

**Prévention :**
- Estimer la complexité de la tâche avant de décider de planifier
- Les tâches simples (une seule étape, réponse claire) devraient se passer complètement de planification
- Fixer un budget de temps de planification proportionnel à la complexité de la tâche

### 4. Les échecs en cascade

**Ce qui se passe :** une étape échouée fait échouer les étapes suivantes en réaction en chaîne, et l'agent ne détecte pas la cause racine.

**Exemple :**
```
Etape 1 : recuperer le profil utilisateur -> renvoie une erreur
          (authentification expiree)
Etape 2 : traiter les preferences utilisateur -> echoue (pas de
          donnees de profil)
Etape 3 : generer des recommandations -> echoue (pas de preferences)
Etape 4 : mettre en forme la sortie -> echoue (pas de recommandations)
Agent : "Je n'ai pas pu terminer la tache a cause d'erreurs de
        formatage."
  [Faux ! Le vrai probleme etait l'authentification expiree a l'etape 1]
```

**Prévention :**
- Traiter la sortie de chaque étape comme une validation d'entrée pour l'étape suivante
- En cas d'échec, remonter la chaîne pour trouver la cause racine
- Échouer rapidement sur les dépendances critiques plutôt que d'essayer de continuer sans elles
- Signaler la véritable cause racine, pas seulement le dernier symptôme

### 5. L'épuisement du contexte

**Ce qui se passe :** le plan et son historique d'exécution remplissent la context window, ne laissant plus de place pour que l'agent raisonne sur l'étape actuelle.

**Prévention :**
- Résumer les étapes terminées plutôt que de conserver tous les détails
- Stocker les résultats intermédiaires en externe et les référencer par ID
- Budgéter l'espace de contexte : réserver une quantité fixe pour le raisonnement sur l'étape actuelle
- Voir la [leçon 5 : Mémoire et contexte](../05-memory-and-context/README.md) pour des stratégies détaillées

### Résumé des modes d'échec

| Mode d'échec | Symptôme | Prévention clé |
|-------------|---------|---------------|
| Boucles infinies | Même action répétée | Nombre maximal d'itérations + détection de répétition |
| Dérive du plan | L'agent s'éloigne du sujet | Vérification périodique de l'alignement avec l'objectif |
| Sur-planification | Planification excessive pour des tâches simples | Estimation de la complexité avant planification |
| Échecs en cascade | Mauvaise cause racine signalée | Remontée de chaîne en cas d'échec + échec rapide |
| Épuisement du contexte | L'agent perd sa cohérence | Budgétisation du contexte + synthèse |

---

## Pour tout assembler : un exemple pratique

Voici un exemple condensé montrant comment la planification, le raisonnement, et la replanification se combinent :

```
Utilisateur : "Analyse nos logs d'erreurs de la semaine passee et cree
              des tickets Jira pour les principaux problemes."

PLAN :
  Phase 1 : interroger les logs d'erreurs (7 derniers jours, severite
           ERROR + FATAL)
  Phase 2 : grouper par type, classer par frequence, identifier le top 5
  Phase 3 : pour chaque probleme, verifier les tickets existants, en
           creer un nouveau si besoin
  Phase 4 : resumer les conclusions

EXECUTION (ReAct au sein de chaque phase) :
  Agir : log_query(severity=["ERROR","FATAL"], days=7)
  Observer : 12 847 entrees renvoyees
  Agir : log_analyze(group_by="error_message", sort="count_desc")
  Observer : problemes principaux identifies (timeouts de connexion,
            pointeur nul, limites de debit)

  Agir : jira_search(query="Connection timeout payment-service")
  Observer : aucun ticket existant -> creation du ticket PLATFORM-1234

REPLANIFICATION (en cours d'execution) :
  Observer : PLATFORM-999 existe deja pour NullPointerException
  Ajustement : mettre a jour le ticket existant avec le nouveau nombre
              d'occurrences plutot que de creer un doublon

LIVRAISON :
  "12 847 erreurs analysees. 4 nouveaux tickets crees, 1 existant
   mis a jour."
```

Remarquez comment l'agent utilise une planification hiérarchique en amont, une exécution de type ReAct au sein de chaque phase, et une replanification quand il découvre un ticket existant. Tous les patterns travaillent ensemble.

---

## Points clés à retenir

1. **La planification transforme les tâches complexes en étapes gérables.** Sans elle, les agents manquent des étapes importantes ou gaspillent des efforts sur des digressions.

2. **La boucle de résolution de problèmes (Mission, Scène, Penser, Agir, Observer) est le moteur de tout agent planificateur.** Comprendre cette boucle vous aide à concevoir et à déboguer le comportement de l'agent.

3. **Le plan-puis-exécute et les approches réactives sont les deux extrémités d'un spectre.** La plupart des agents pratiques utilisent un hybride : une planification légère en amont avec une exécution réactive.

4. **La planification hiérarchique gère les tâches complexes** en les décomposant en objectifs, sous-objectifs, tâches, et étapes. Cela reflète le fonctionnement de la gestion de projet.

5. **La replanification est essentielle.** Les plans vont se rompre. Les bons agents détectent les échecs tôt et s'ajustent. La capacité à s'adapter distingue les agents utiles des agents fragiles.

6. **Les techniques de raisonnement (CoT, ToT, autocohérence) rendent la planification plus efficace.** Chain-of-Thought pour la logique étape par étape, Tree-of-Thoughts pour explorer des alternatives, l'autocohérence pour valider des décisions critiques.

7. **Faites attention aux modes d'échec courants.** Les boucles infinies, la dérive du plan, la sur-planification, les échecs en cascade, et l'épuisement du contexte sont des problèmes prévisibles avec des solutions connues.

8. **La couche d'orchestration relie le tout.** Elle gère le cycle de vie du plan, route l'exécution, gère les erreurs, et garde l'agent sur la bonne voie.

---

## Pour aller plus loin

- [Présentation de Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview)
- [Documentation de l'Agent Development Kit (ADK)](https://google.github.io/adk-docs/)
- [Codelabs Google Cloud AI](https://codelabs.developers.google.com/?cat=AI)

---

**Leçon suivante :** [Systèmes multi-agents](../07-multi-agent-systems/README.md)
