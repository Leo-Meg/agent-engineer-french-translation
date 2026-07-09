# Leçon 4 : design patterns agentiques

## Ce que vous allez apprendre

- Ce que sont les design patterns (patrons de conception) agentiques et pourquoi ils comptent
- Les quatre patterns fondamentaux : ReAct, Reflection, Tool Use et Planning
- Quand utiliser chaque pattern, et les compromis associés
- Comment combiner des patterns dans des agents réels

## Prérequis

- [Leçon 1 : Qu'est-ce qu'un agent IA ?](../01-what-are-ai-agents/README.md)
- [Leçon 2 : Comment pensent les agents](../02-how-agents-think/README.md)
- [Leçon 3 : Les outils - donner des mains aux agents](../03-tools-giving-agents-hands/README.md)

---

## ELI5 : les design patterns, c'est comme des recettes de cuisine

Imaginez que vous vouliez préparer le dîner. Vous pourriez simplement attraper des ingrédients au hasard et espérer que ça marche. Ou vous pourriez suivre une recette - un enchaînement d'étapes éprouvé, que quelqu'un a déjà mis au point avec succès.

Les design patterns sont des recettes pour construire des agents IA. Ce sont des façons éprouvées d'organiser comment un agent pense, agit et apprend. Tout comme un livre de cuisine contient différentes recettes pour différents repas, il existe différents patterns pour différents types de comportements d'agent.

Et tout comme un grand chef peut combiner des techniques issues de plusieurs recettes, les meilleurs agents combinent généralement plusieurs patterns entre eux.

---

## Pourquoi les design patterns comptent

Si vous programmez depuis un certain temps, vous connaissez probablement des design patterns comme Observer, Strategy ou Factory. Ces patterns donnent aux ingénieurs un vocabulaire commun et des plans éprouvés pour résoudre des problèmes récurrents.

Les design patterns agentiques servent le même objectif, mais pour les agents IA. Ils décrivent des structures récurrentes dans la façon dont les agents :

- **Raisonnent** sur des problèmes
- **Entreprennent des actions** dans le monde
- **Apprennent** des résultats
- **Améliorent** leurs propres sorties

Sans ces patterns, construire un agent ressemble à écrire du code spaghetti - tout est emmêlé et difficile à déboguer. Avec eux, vous obtenez une architecture claire, plus facile à construire, tester et maintenir.

### Du simple à l'agentique

Toutes les interactions avec un LLM n'ont pas besoin d'un design pattern. Voici un spectre approximatif :

| Niveau | Description | Exemple | Patterns nécessaires |
|-------|------------|---------|----------------|
| **Simple prompt** | Une question, une réponse | « Quelle est la capitale de la France ? » | Aucun |
| **Sortie structurée** | Le LLM met en forme sa réponse | « Renvoie ceci en JSON » | Aucun |
| **Chaîne** | Plusieurs appels au LLM en séquence | Résumer, puis traduire | Minimal |
| **Agent** | Le LLM décide de la prochaine action | Faire une recherche sur un sujet et rédiger un rapport | ReAct, Tool Use, Planning |
| **Multi-agent** | Plusieurs agents collaborent | Une équipe d'agents qui construit un logiciel | Tout ce qui précède + coordination |

Les design patterns deviennent importants dès que l'on dépasse les simples chaînes pour entrer dans un comportement véritablement agentique - où le LLM prend des décisions sur la prochaine action à entreprendre.

---

## Pattern 1 : ReAct (reason + act, raisonner + agir)

ReAct est le design pattern agentique le plus fondamental. Si vous ne devez apprendre qu'un seul pattern, que ce soit celui-ci.

### L'idée

ReAct combine le **raisonnement** (réfléchir à quoi faire) et l'**action** (le faire réellement) dans une boucle entrelacée. L'agent :

1. **Pense** à la situation actuelle
2. **Agit** en appelant un outil ou en accomplissant une étape
3. **Observe** le résultat
4. **Recommence** jusqu'à ce que la tâche soit terminée

### L'analogie du serveur de restaurant

Pensez à un serveur dans un restaurant :

1. **Penser :** « La table 5 a commandé les pâtes. Je dois vérifier si la cuisine les a préparées. »
2. **Agir :** Se rendre en cuisine et demander.
3. **Observer :** Le chef dit qu'il faut 5 minutes de plus.
4. **Penser :** « D'accord, je devrais m'occuper des boissons de la table 3 en attendant. »
5. **Agir :** Aller à la table 3 et resservir les boissons.
6. **Observer :** La table 3 est satisfaite.
7. **Penser :** « Assez de temps s'est écoulé. Je vais vérifier les pâtes de la table 5. »
8. **Agir :** Retourner en cuisine.
9. **Observer :** Les pâtes sont prêtes.
10. **Agir :** Servir les pâtes à la table 5.

Le serveur ne planifie pas tout son service à l'avance. Il pense, agit, observe et s'adapte - en continu.

### À quoi ressemble ReAct

Voici le déroulement général :

```
Utilisateur : "Quel temps fait-il à Tokyo et dois-je prendre un parapluie ?"

Pensée de l'agent : je dois connaître la météo actuelle à Tokyo. Je vais
                     utiliser l'outil météo.
Action de l'agent : appeler weather_api(location="Tokyo")
Observation : {"temp": 18, "condition": "pluie", "forecast": "pluie continue jusqu'à demain"}

Pensée de l'agent : il pleut actuellement à Tokyo et la pluie devrait continuer.
                     L'utilisateur devrait prendre un parapluie.
Action de l'agent : répondre à l'utilisateur.

Réponse de l'agent : "Il fait actuellement 18°C et il pleut à Tokyo, avec de la
                      pluie prévue jusqu'à demain. Prenez absolument un parapluie !"
```

### Caractéristiques clés de ReAct

- **Raisonnement et action entrelacés.** L'agent n'agit pas aveuglément - il réfléchit à chaque étape.
- **Ancré dans les observations.** Chaque pensée est informée par ce que l'agent observe réellement, pas seulement par ce qu'il imagine.
- **Flexible.** L'agent peut changer de cap en fonction de ce qu'il découvre.
- **Transparent.** La trace de raisonnement facilite le débogage de ce à quoi pensait l'agent.

### Quand utiliser ReAct

| Bon cas d'usage | Mauvais cas d'usage |
|----------|----------|
| Tâches nécessitant des informations externes | Tâches de pure génération de texte |
| Problèmes multi-étapes avec un chemin incertain | Questions-réponses simples |
| Situations nécessitant une piste d'audit | Applications critiques en termes de latence |
| Tâches nécessitant de s'adapter à de nouvelles informations | Tâches avec une séquence fixe et connue |

### Pièges courants

- **Les boucles de raisonnement.** L'agent répète la même pensée sans progresser. Ajoutez un nombre maximal d'itérations.
- **Les actions hallucinées.** L'agent « appelle » un outil qui n'existe pas. Validez les noms d'outils avant l'exécution.
- **L'aveuglement aux observations.** L'agent ignore ce que l'outil a renvoyé et continue avec son hypothèse précédente. Assurez-vous que les observations sont clairement injectées dans le contexte.

---

## Pattern 2 : Reflection (réflexion)

### L'idée

Dans le pattern Reflection, un agent relit sa propre sortie et l'améliore. Plutôt que de produire une seule réponse et de passer à autre chose, l'agent génère un brouillon, le critique, puis le révise.

### L'analogie de l'écrivain

Pensez à un écrivain qui travaille sur un article :

1. **Brouillon :** écrire la première version.
2. **Relecture :** la relire. « Hmm, l'introduction est faible et le paragraphe 3 contredit le paragraphe 1. »
3. **Révision :** réécrire l'introduction et corriger la contradiction.
4. **Nouvelle relecture :** « Mieux. Mais la conclusion a besoin d'un appel à l'action plus fort. »
5. **Nouvelle révision :** améliorer la conclusion.
6. **Terminé :** la version finale est bien plus solide que le premier brouillon.

Aucun écrivain expérimenté ne publie un premier jet. De la même manière, les agents qui relisent leur propre sortie produisent des résultats nettement meilleurs.

### À quoi ressemble Reflection

```
Étape 1 - Génération :
  L'agent produit une réponse initiale à la demande de l'utilisateur.

Étape 2 - Critique :
  L'agent (ou un critique séparé) relit la réponse :
  « Ce code a un bug à la ligne 12 - l'index de la boucle est décalé de un.
   De plus, la fonction ne gère pas les erreurs pour une entrée vide. »

Étape 3 - Révision :
  L'agent corrige les problèmes identifiés et produit une version améliorée.

Étape 4 - Évaluation :
  « Le bug est corrigé et la gestion des erreurs est ajoutée. Le code gère
   maintenant les cas limites. Cela répond aux exigences. »
```

### Variantes de Reflection

| Variante | Comment ça marche | Exemple |
|-----------|-------------|---------|
| **Auto-réflexion** | Le même LLM relit sa propre sortie | « Relis ton code à la recherche de bugs » |
| **Agent critique** | Une instance de LLM séparée fait la relecture | Un agent dédié à la relecture de code |
| **Fondée sur une grille** | La réflexion est guidée par des critères précis | « Vérifie : l'exactitude, l'exhaustivité, le ton » |
| **Pilotée par les tests** | La sortie est testée par rapport à des vérifications concrètes | Lancer les tests unitaires, vérifier le formatage |

### Quand utiliser Reflection

| Bon cas d'usage | Mauvais cas d'usage |
|----------|----------|
| Génération de code (détecter les bugs avant la mise en production) | Réponses conversationnelles en temps réel |
| Tâches d'écriture (améliorer la clarté et la structure) | Recherches factuelles simples |
| Raisonnement complexe (détecter les erreurs logiques) | Tâches où la vitesse compte plus que la qualité |
| Toute tâche où la qualité compte plus que la vitesse | Tâches avec des réponses objectivement vérifiables |

### Conseils pratiques

- **Fixez un nombre maximal de tours de réflexion.** Deux à trois tours suffisent généralement. Au-delà, les bénéfices sont souvent décroissants.
- **Utilisez des critères précis.** « Améliore-le » est vague. « Vérifie les vulnérabilités d'injection SQL, la gestion des erreurs manquante, et les problèmes de performance » donne un objectif clair à la réflexion.
- **Envisagez d'utiliser un modèle ou un prompt différent pour le critique.** Un regard neuf détecte plus de problèmes qu'une relecture avec le même point de vue.
- **Suivez ce qui a changé.** Journalisez les différences entre chaque révision, afin de pouvoir vérifier que la réflexion a réellement amélioré les choses.

---

## Pattern 3 : Tool Use (utilisation d'outils)

### L'idée

Dans le pattern Tool Use, l'agent décide quels outils appeler, avec quels arguments, et dans quel ordre. Le LLM agit comme un moteur de raisonnement qui orchestre des capacités externes, plutôt que d'essayer de tout faire lui-même.

Nous avons couvert les outils en profondeur dans la [leçon 3](../03-tools-giving-agents-hands/README.md). Cette section se concentre sur le *pattern* de la façon dont les agents décident d'utiliser des outils.

### L'analogie de l'artisan

Pensez à un menuisier expérimenté qui travaille sur un projet. Il a un atelier plein d'outils - scies, perceuses, ponceuses, serre-joints. Pour chaque étape du projet, il :

1. Évalue ce qui doit être fait ensuite
2. Choisit le bon outil pour cette étape précise
3. Utilise l'outil
4. Inspecte le résultat
5. Décide de la suite

Il n'utilise pas un marteau pour tout. Il choisit l'outil adapté à la tâche. Et parfois, il réalise en cours de projet qu'il a besoin d'un outil qu'il ne pensait pas utiliser.

### À quoi ressemble Tool Use

```
Utilisateur : "Trouve les 3 dépôts les plus en vogue sur GitHub à propos
              des agents IA et résume ce que fait chacun."

Pensée de l'agent : je dois rechercher sur GitHub les dépôts d'agents IA en vogue.
Action de l'agent : github_search(query="AI agents", sort="stars", limit=3)
Observation : [repo1: "autogen", repo2: "crewai", repo3: "langchain-agents"]

Pensée de l'agent : maintenant je dois obtenir les détails de chaque dépôt.
Action de l'agent : github_get_repo(name="autogen")
Observation : {description: "...", readme: "...", stars: 42000}

Action de l'agent : github_get_repo(name="crewai")
Observation : {description: "...", readme: "...", stars: 28000}

Action de l'agent : github_get_repo(name="langchain-agents")
Observation : {description: "...", readme: "...", stars: 19000}

Pensée de l'agent : j'ai maintenant assez d'information pour résumer les trois.
Réponse de l'agent : "Voici les 3 dépôts d'agents IA les plus en vogue..."
```

### Stratégies de sélection d'outils

Les agents utilisent plusieurs stratégies pour décider quel outil utiliser :

| Stratégie | Description | Compromis |
|----------|------------|-----------|
| **Correspondance directe** | Le nom/la description de l'outil correspond au besoin | Rapide, mais fragile si les outils sont mal nommés |
| **Raisonnement sur les capacités** | L'agent raisonne sur ce que chaque outil peut faire | Plus flexible, mais utilise plus de tokens |
| **Exemples few-shot** | Des exemples dans le prompt montrent quand utiliser chaque outil | Fiable, mais occupe de l'espace de contexte |
| **Recommandations d'outils** | Le système suggère des outils pertinents selon la requête | Réduit la charge de décision pour le LLM |

### Appels d'outils parallèles ou séquentiels

Certaines tâches permettent d'appeler plusieurs outils à la fois :

- **Séquentiel :** rechercher un utilisateur, puis consulter son historique de commandes (il faut d'abord l'ID de l'utilisateur)
- **Parallèle :** vérifier la météo dans trois villes différentes (toutes indépendantes)

Les appels d'outils parallèles réduisent significativement la latence. Quand vous concevez votre agent, identifiez quels appels d'outils sont indépendants et peuvent s'exécuter simultanément.

### Quand utiliser Tool Use

Ce pattern s'applique à presque tout agent qui interagit avec des systèmes externes. Les décisions de conception clés sont :

- **Combien d'outils ?** Commencez petit. Un agent avec 3 à 5 outils bien conçus surpasse généralement un agent avec 50 outils mal conçus.
- **Quelle est la précision des schémas d'outils ?** De meilleures descriptions mènent à une meilleure sélection d'outils.
- **Que se passe-t-il quand un outil échoue ?** Les bons agents gèrent les erreurs avec élégance - nouvelle tentative, outil alternatif, ou demande d'aide à l'utilisateur.

---

## Pattern 4 : Planning (planification)

### L'idée

Dans le pattern Planning, l'agent crée un plan avant d'exécuter. Plutôt que de déterminer chaque étape au fur et à mesure (comme avec ReAct), l'agent réfléchit à l'avance et établit une approche structurée.

### L'analogie du chef de projet

Imaginez un chef de projet qui reçoit une demande de construction d'une nouvelle fonctionnalité :

1. **Décomposer :** « Nous devons mettre à jour le schéma de base de données, écrire les endpoints d'API, construire l'interface, et ajouter des tests. »
2. **Ordonner le travail :** « D'abord le schéma, puis l'API, puis l'interface, puis les tests - chaque étape dépend de la précédente. »
3. **Assigner les ressources :** « Le travail de base de données va à l'équipe backend, l'interface à l'équipe frontend. »
4. **Exécuter et suivre :** avancer dans le plan, en cochant les éléments au fur et à mesure.
5. **Ajuster si nécessaire :** « Le changement de schéma était plus complexe que prévu - je dois replanifier le calendrier. »

### À quoi ressemble Planning

```
Utilisateur : "Écris un article de blog complet sur les bonnes pratiques
              de sécurité Kubernetes."

Plan de l'agent :
  1. Rechercher les menaces de sécurité Kubernetes actuelles et les CVE
  2. Identifier les 5 à 7 meilleures pratiques de sécurité
  3. Pour chaque pratique, trouver des exemples et des commandes concrets
  4. Rédiger un plan avec introduction, sections principales, et conclusion
  5. Rédiger chaque section
  6. Relire l'article complet pour l'exactitude et la fluidité
  7. Ajouter des exemples de code et la mise en forme

Exécution de l'agent :
  [exécute les étapes 1 à 7 dans l'ordre, en ajustant si nécessaire]
```

### Stratégies de planification

| Stratégie | Comment ça marche | Idéal pour |
|----------|-------------|----------|
| **Planification séquentielle** | Créer une liste linéaire d'étapes | Tâches simples et bien comprises |
| **Planification hiérarchique** | Décomposer en objectifs de haut niveau, puis en sous-tâches | Projets complexes à plusieurs phases |
| **Planification conditionnelle** | Inclure des branches si/alors dans le plan | Tâches aux résultats incertains |
| **Planification itérative** | Planifier quelques étapes, exécuter, replanifier | Tâches où les étapes suivantes dépendent des résultats précédents |

### Plan-puis-exécute contre ReAct

Ces deux patterns représentent des philosophies différentes :

| Aspect | Planning | ReAct |
|--------|----------|-------|
| **Quand les décisions sont prises** | Surtout en amont | Étape par étape |
| **Adaptabilité** | Nécessite une replanification explicite | Naturellement adaptatif |
| **Efficacité** | Peut paralléliser les étapes indépendantes | Généralement séquentiel |
| **Transparence** | Plan complet visible en amont | Raisonnement visible à chaque étape |
| **Risque de travail gaspillé** | Plus élevé si le plan s'avère incorrect | Plus faible, s'adapte au fil de l'eau |
| **Idéal pour** | Tâches bien structurées | Tâches exploratoires |

En pratique, la plupart des agents mélangent les deux approches : ils établissent un plan approximatif en amont, puis utilisent un raisonnement de type ReAct pendant l'exécution.

### Quand utiliser Planning

| Bon cas d'usage | Mauvais cas d'usage |
|----------|----------|
| Tâches multi-étapes avec une structure claire | Tâches simples à une seule étape |
| Tâches où l'ordre compte | Agents purement réactifs/conversationnels |
| Travail pouvant être parallélisé | Tâches dont le chemin est totalement inconnu |
| Projets nécessitant un suivi de progression | Demandes rapides et ponctuelles |

---

## Comparer les patterns

Voici une comparaison côte à côte pour vous aider à choisir :

| Pattern | Idée centrale | Force | Faiblesse | Coût |
|---------|-----------|----------|----------|------|
| **ReAct** | Boucle penser-agir-observer | Flexible, transparent | Peut être lent, peut boucler | Moyen (plusieurs appels au LLM) |
| **Reflection** | Auto-relecture et amélioration | Sortie de meilleure qualité | Ajoute de la latence | Élevé (plusieurs passes) |
| **Tool Use** | Orchestrer des outils externes | Étend les capacités de l'agent | Dépend de la qualité des outils | Variable (selon les outils) |
| **Planning** | Planifier avant d'exécuter | Structuré, efficace | Fragile si le plan est erroné | Moyen à élevé (planification + exécution) |

### Arbre de décision

Posez-vous ces questions :

1. **L'agent a-t-il besoin d'informations ou d'actions externes ?** Oui -> Tool Use
2. **La tâche est-elle multi-étapes avec un chemin incertain ?** Oui -> ReAct
3. **La qualité est-elle critique et la tâche a-t-elle des critères clairs ?** Oui -> Reflection
4. **La tâche est-elle complexe mais bien structurée ?** Oui -> Planning
5. **La réponse à la plupart de ces questions est-elle « oui » ?** -> Combinez les patterns

---

## Combiner les patterns

Les agents du monde réel n'utilisent presque jamais un seul pattern de manière isolée. Les agents les plus efficaces superposent plusieurs patterns.

### Combinaisons courantes

**ReAct + Tool Use** (la combinaison la plus courante)

L'agent raisonne sur quoi faire, utilise des outils pour agir, observe les résultats, et raisonne à nouveau. C'est l'ossature de la plupart des agents pratiques.

```
Penser -> Utiliser un outil -> Observer -> Penser -> Utiliser un outil -> Observer -> Répondre
```

**Planning + ReAct + Tool Use**

L'agent crée un plan, puis exécute chaque étape en utilisant un raisonnement de type ReAct avec des outils.

```
Planifier -> [Penser -> Agir -> Observer] -> [Penser -> Agir -> Observer] -> ... -> Terminé
```

**Planning + Reflection**

L'agent crée un plan, l'exécute, puis relit le résultat global avant de le livrer.

```
Planifier -> Exécuter -> Relire -> Réviser -> Livrer
```

**La panoplie complète : Planning + ReAct + Tool Use + Reflection**

Pour les tâches complexes et à fort enjeu, vous pourriez utiliser les quatre :

```
Planifier l'approche
  -> Exécuter chaque étape avec ReAct + Outils
    -> Relire le résultat global
      -> Réviser si nécessaire
        -> Livrer
```

### Exemple : un agent de génération de code

Voici comment un agent de génération de code pourrait combiner les patterns :

1. **Planning :** « Je dois écrire une API REST. Étapes : définir le modèle de données, créer les endpoints, ajouter la validation, écrire les tests. »
2. **ReAct + Tool Use :** pour chaque étape, l'agent raisonne sur quoi faire, utilise des outils (lecteur de fichiers, recherche de code, linter) pour rassembler de l'information et écrire du code.
3. **Reflection :** après avoir écrit le code, l'agent le relit par rapport aux bonnes pratiques. « Est-ce que cela gère les erreurs ? L'entrée est-elle validée ? Y a-t-il des problèmes de sécurité ? »
4. **Révision :** l'agent corrige les problèmes détectés pendant la réflexion.

### Quand ne pas combiner

Plus de patterns n'est pas toujours mieux. Chaque pattern ajoute :

- **De la latence :** plus d'appels au LLM signifie plus de temps
- **Du coût :** plus de tokens signifie plus d'argent
- **De la complexité :** plus de pièces mobiles signifie plus de débogage

Pour un agent simple de questions-réponses, ReAct + Tool Use est probablement tout ce dont vous avez besoin. Réservez la panoplie complète aux tâches complexes et à forte valeur, où la qualité justifie le coût.

---

## Les patterns sur Google Cloud

Le [Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview) de Google Cloud fournit l'infrastructure nécessaire pour construire des agents qui utilisent ces patterns. L'[Agent Development Kit (ADK)](https://google.github.io/adk-docs/) vous donne les briques pour les implémenter.

Concepts clés dans l'écosystème Google Cloud :

- **Agent Engine** gère le cycle de vie de vos agents - déploiement, mise à l'échelle, et monitoring
- **ADK** fournit le framework pour définir le comportement de l'agent, les outils, et l'orchestration
- **Les modèles Gemini** servent d'ossature LLM qui alimente le raisonnement dans chaque pattern

Nous mettrons cela en pratique dans la [leçon 12](../12-getting-started-with-vertex-and-adk/README.md) et la [leçon 13](../13-building-your-first-agent/README.md).

---

## Points clés à retenir

1. **Les design patterns agentiques sont des plans éprouvés** pour organiser comment les agents pensent et agissent. Ils vous donnent un vocabulaire commun et un point de départ pour l'architecture.

2. **ReAct est la fondation.** La boucle penser-agir-observer est le pattern le plus fondamental et le point de départ de la plupart des agents.

3. **Reflection améliore considérablement la qualité**, mais coûte du temps et des tokens. Utilisez-le quand la qualité compte plus que la vitesse.

4. **Tool Use étend ce que les agents peuvent faire** au-delà des connaissances intégrées du LLM. Une bonne conception d'outils est aussi importante qu'une bonne conception de prompt.

5. **Planning apporte de la structure** aux tâches complexes. Cela fonctionne mieux quand la tâche est bien comprise et que les étapes peuvent être établies à l'avance.

6. **Combinez les patterns avec discernement.** Plus de patterns signifie plus de capacités, mais aussi plus de complexité et de coût. Commencez simple et ajoutez des patterns selon les besoins.

7. **Il n'y a pas de meilleur pattern unique.** Le bon choix dépend de votre tâche, de vos exigences de qualité, et de vos contraintes de latence et de coût.

---

## Pour aller plus loin

- [Présentation de Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview)
- [Documentation de l'Agent Development Kit (ADK)](https://google.github.io/adk-docs/)
- [Codelabs Google Cloud AI](https://codelabs.developers.google.com/?cat=AI)

---

**Leçon suivante :** [Mémoire et contexte - comment les agents se souviennent](../05-memory-and-context/README.md)
