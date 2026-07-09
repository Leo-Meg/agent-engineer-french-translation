# Leçon 18 : orchestrateurs - gérer le flux de contrôle des agents

## Introduction

Dans les leçons précédentes, nous avons couvert ce que sont les agents, comment ils utilisent des outils, et comment plusieurs agents peuvent travailler ensemble. Mais nous n'avons pas encore examiné de près la couche qui fait tourner tout cela - la couche d'orchestration.

L'orchestrateur est le système de contrôle qui décide : que se passe-t-il ensuite ? Quel agent s'exécute ? Qu'est-ce qui entre dans le contexte ? Quand nous arrêtons-nous ? C'est la partie du système qui se situe entre l'objectif de l'utilisateur et l'exécution réelle, coordonnant tout.

Si les agents sont les travailleurs, l'orchestrateur est le chef de projet.

### ELI5 : pensez à un orchestrateur comme à un réalisateur de film

Un réalisateur de film ne joue pas, n'actionne pas la caméra, et ne s'occupe pas de l'éclairage. Il coordonne plutôt tous les spécialistes : « Cadreur, fais un gros plan. Acteur, dis ta réplique. Ingénieur du son, ajoute la musique. » Il décide de la séquence, gère les problèmes quand une scène ne fonctionne pas, et fait avancer le tout vers le film terminé.

Un orchestrateur d'agents fait la même chose - il coordonne quels agents s'exécutent, dans quel ordre, avec quelles entrées, et décide quoi faire quand quelque chose tourne mal.

> **Point clé à retenir :** l'orchestrateur gère le flux de contrôle de votre système d'agents. Choisir le bon pattern d'orchestration est l'une des décisions architecturales les plus importantes que vous ferez.

---

## Que fait réellement un orchestrateur ?

L'orchestrateur gère quatre préoccupations centrales :

### 1. Le flux de contrôle

Décider ce qui se passe ensuite. L'agent devrait-il appeler un outil ? Passer la main à un autre agent ? Demander une clarification à l'utilisateur ? S'arrêter parce que l'objectif est atteint ?

### 2. L'assemblage du contexte

Construire le bon contexte pour chaque étape. Cela signifie sélectionner quelle information entre dans la context window du LLM - instructions système, mémoire pertinente, résultats d'outils, historique de conversation - et empêcher la fenêtre de déborder.

### 3. La gestion de l'état

Suivre ce qui a été fait, ce qui doit encore se passer, ce qui a réussi, et ce qui a échoué. Dans les systèmes multi-agents, cela signifie aussi gérer l'état partagé entre les agents.

### 4. La gestion des erreurs

Décider quoi faire quand les choses tournent mal. L'agent devrait-il réessayer ? Essayer une approche différente ? Se rabattre sur une méthode plus simple ? Escalader vers un humain ?

L'orchestrateur exécute la boucle centrale de l'agent :

```
               +----> Assembler le contexte
               |            |
               |            v
Recevoir       |      Invoquer le LLM (raisonner)
l'objectif     |            |
               |            v
               |      Executer l'action (agir)
               |            |
               |            v
               +---- Observer le resultat
                            |
                Objectif atteint ? ---> Renvoyer le resultat
```

Chaque itération de cette boucle est une « étape ». L'orchestrateur décide quand reboucler et quand s'arrêter.

---

## Deux types d'orchestration

La décision de conception la plus fondamentale est de savoir où votre orchestrateur se situe sur le spectre entre contrôle déterministe et contrôle dynamique.

### Déterministe (basé sur un workflow)

Le flux de contrôle est prédéfini. L'orchestrateur suit un plan fixe - il ne consulte pas un LLM pour décider ce qui se passe ensuite. Les étapes s'exécutent dans un ordre prédéterminé avec des conditions prédéterminées.

**Forces :**
- Comportement prévisible - vous savez exactement ce qui va se passer
- Facile à déboguer - parcourez le workflow comme du code classique
- Rapide - aucun appel au LLM pour les décisions d'orchestration
- Fiable - aucun risque que l'orchestrateur dévie de sa trajectoire

**Limitations :**
- Ne peut pas gérer des situations nouvelles pour lesquelles le workflow n'a pas été conçu
- Nécessite une connaissance à l'avance de tous les chemins possibles
- Les changements au workflow nécessitent des changements de code

**Exemple :** un pipeline de traitement de documents qui s'exécute toujours ainsi : extraire le texte, classifier le type de document, extraire les entités, valider, stocker.

### Dynamique (piloté par LLM)

L'orchestrateur utilise un LLM pour décider ce qui se passe ensuite. À chaque étape, il raisonne sur l'état actuel et choisit la prochaine action. C'est la boucle ReAct classique.

**Forces :**
- Gère des tâches nouvelles et ouvertes
- Peut s'adapter quand les plans échouent
- Peut travailler sur des tâches que le développeur n'a pas anticipées

**Limitations :**
- Moins prévisible - la même entrée peut produire des chemins d'exécution différents
- Plus difficile à déboguer - « pourquoi l'agent a-t-il fait ça ? »
- Plus coûteux - les appels au LLM pour l'orchestration s'accumulent
- Peut se retrouver bloqué dans des boucles ou prendre de mauvaises décisions de routage

**Exemple :** un assistant de recherche qui décide dynamiquement s'il faut rechercher sur le web, interroger une base de données, lire un document, ou demander une clarification à l'utilisateur, en fonction de ce qu'il a appris jusqu'ici.

### Hybride (le choix pratique)

La plupart des systèmes en production combinent les deux approches. Ils utilisent une orchestration déterministe pour la structure globale, tout en permettant une flexibilité pilotée par LLM au sein des étapes individuelles.

**Exemple :** un système de support client avec un flux externe déterministe (recevoir le ticket, classifier, router vers un spécialiste, vérifier la résolution, clore) où chaque étape utilise en interne un agent LLM qui peut raisonner librement sur comment gérer sa tâche spécifique.

---

## Patterns d'orchestration centraux

Voici les patterns les plus largement utilisés, avec des indications sur quand chacun convient :

### Séquentiel (pipeline)

Les agents s'exécutent les uns après les autres dans un ordre défini. La sortie de chaque agent devient l'entrée du suivant.

```
Entree --> [Agent A] --> [Agent B] --> [Agent C] --> Sortie
```

**Quand l'utiliser :**
- Tâches avec des étapes claires qui s'appuient les unes sur les autres
- Workflows de raffinement (brouillon, relecture, édition)
- Pipelines de traitement de données (extraire, transformer, valider)

**Quand l'éviter :**
- Quand les étapes sont indépendantes et pourraient s'exécuter en parallèle
- Quand vous devez revenir en arrière (l'échec de l'agent C nécessite de relancer l'agent A)

**Exemple :** pipeline de génération de code : un agent d'analyse des exigences produit une spécification, un agent de codage écrit l'implémentation, un agent de test écrit les tests, un agent de relecture vérifie les problèmes.

Dans ADK, c'est le `SequentialAgent` :
```python
pipeline = SequentialAgent(
    name="code_pipeline",
    sub_agents=[analyzer, coder, tester, reviewer]
)
```

Consultez la [documentation SequentialAgent d'ADK](https://adk.dev/agents/workflow-agents/sequential-agents/) pour les détails d'implémentation.

### Parallèle (fan-out / gather)

Plusieurs agents s'exécutent en même temps sur la même entrée. Les résultats sont collectés et combinés.

```
            +--> [Agent A] --+
            |                |
Entree -----+--> [Agent B] --+--> Combiner --> Sortie
            |                |
            +--> [Agent C] --+
```

**Quand l'utiliser :**
- Analyse indépendante depuis plusieurs perspectives
- Tâches sensibles à la latence où l'exécution parallèle fait gagner du temps
- Obtenir des points de vue divers sur la même entrée

**Quand l'éviter :**
- Quand les agents ont besoin des résultats les uns des autres pour faire leur travail
- Quand les résultats pourraient entrer en conflit et que vous n'avez aucune stratégie de résolution

**Exemple :** une relecture de code où un agent sécurité, un agent performance, et un agent style relisent tous la même PR simultanément. Les résultats sont fusionnés en une seule relecture.

Dans ADK, c'est le `ParallelAgent` :
```python
review = ParallelAgent(
    name="code_review",
    sub_agents=[security_reviewer, performance_reviewer, style_reviewer]
)
```

Consultez la [documentation ParallelAgent d'ADK](https://adk.dev/agents/workflow-agents/parallel-agents/) pour les détails d'implémentation.

### Boucle (raffinement itératif)

Un agent s'exécute de manière répétée jusqu'à ce qu'une condition soit remplie. Cela inclut deux sous-patterns importants :

**Générateur-Critique (fabricant-vérificateur) :** un agent produit une sortie, un autre l'évalue, et la boucle continue jusqu'à ce que l'évaluateur approuve.

```
+--> [Agent generateur] --> [Agent critique] --+
|                              |                |
|         Pas assez bon -------+                |
|                                               |
+-----------------------------------------------+
                    |
              Assez bon --> Sortie
```

**Raffinement progressif :** un seul agent améliore sa sortie à travers plusieurs passes, comme un auteur révisant un brouillon.

**Quand l'utiliser :**
- Tâches sensibles à la qualité où les premières tentatives sont rarement suffisamment bonnes
- Tâches avec des critères d'acceptation clairs
- Workflows d'amélioration itérative

**Quand l'éviter :**
- Quand vous ne pouvez pas définir de critères d'arrêt clairs (risque de boucle infinie)
- Quand la première tentative est généralement suffisamment bonne

**Important :** fixez toujours un nombre maximal d'itérations. Sans cela, une boucle peut tourner indéfiniment si le critique n'approuve jamais.

Dans ADK, c'est le `LoopAgent` :
```python
refiner = LoopAgent(
    name="content_refiner",
    sub_agents=[writer, editor],
    max_iterations=5
)
```

Consultez la [documentation LoopAgent d'ADK](https://adk.dev/agents/workflow-agents/loop-agents/) pour les détails d'implémentation.

### Routage (handoff / dispatch)

Une entrée est classifiée et dirigée vers un agent spécialisé. Un seul agent gère chaque requête.

```
            +--> [Agent facturation]
            |
Entree --> [Routeur] +--> [Agent support technique]
            |
            +--> [Agent demandes generales]
```

Le routage peut être :
- **Déterministe :** classification basée sur des règles (si le message contient « facture », router vers la facturation)
- **Piloté par LLM :** l'agent routeur utilise le raisonnement pour choisir le meilleur spécialiste

**Quand l'utiliser :**
- Support client avec des départements spécialisés
- Systèmes multi-domaines où différentes entrées nécessitent une expertise différente
- Quand vous voulez un transfert de contrôle complet (un seul agent actif à la fois)

**Quand l'éviter :**
- Quand la requête ne rentre pas proprement dans des catégories
- Quand plusieurs agents doivent collaborer sur la même requête

### Hiérarchique (coordinateur-travailleur)

Un agent principal coordonne le processus tout en déléguant des tâches à des sous-agents spécialisés.

```
                +---> [Agent recherche]
                |
[Coordinateur] --+---> [Agent analyse]
                |
                +---> [Agent redaction]
```

Le coordinateur :
1. Décompose l'objectif global en sous-tâches
2. Assigne les sous-tâches au bon spécialiste
3. Surveille la progression et gère les dépendances
4. Combine les résultats en une sortie finale

**Quand l'utiliser :**
- Tâches complexes nécessitant plusieurs types d'expertise
- Tâches où le plan n'est pas connu à l'avance et doit être développé
- Travail de type recherche où les conclusions d'un domaine informent ce qu'il faut explorer ensuite

**Quand l'éviter :**
- Tâches simples qui ne justifient pas le surcoût de coordination
- Quand un pipeline séquentiel fonctionnerait tout aussi bien

Dans ADK, vous pouvez accomplir cela en enveloppant les sous-agents comme des outils via `AgentTool`, permettant au coordinateur de les appeler comme des fonctions.

### Chat de groupe (table ronde)

Plusieurs agents participent à une conversation partagée, coordonnée par un gestionnaire de chat.

```
[Gestionnaire de chat]
      |
      +---> [Agent A] : "Je pense que nous devrions..."
      |
      +---> [Agent B] : "En s'appuyant sur ca..."
      |
      +---> [Agent C] : "Je ne suis pas d'accord car..."
      |
      +---> [Agent A] : "Bon point, laisse-moi reviser..."
```

**Quand l'utiliser :**
- Construction de consensus
- Brainstorming où des perspectives diverses améliorent le résultat
- Validation itérative (plusieurs experts relisent et affinent)

**Quand l'éviter :**
- Quand l'efficience compte plus que l'exhaustivité (le chat de groupe est coûteux en tokens)
- Quand plus de trois agents participent (les conversations deviennent chaotiques)

---

## Choisir le bon pattern

| Pattern | Prévisibilité | Flexibilité | Coût en tokens | Idéal pour |
|---------|---------------|-------------|------------|----------|
| Séquentiel | Élevée | Faible | Faible | Processus étape par étape clairs |
| Parallèle | Élevée | Faible | Moyen (concurrent) | Tâches d'analyse indépendantes |
| Boucle | Moyenne | Moyenne | Variable | Raffinement de qualité |
| Routage | Élevée | Moyenne | Faible | Classification multi-domaines |
| Hiérarchique | Moyenne | Élevée | Plus élevé | Recherche complexe multi-étapes |
| Chat de groupe | Faible | Élevée | Le plus élevé | Consensus et brainstorming |

Un arbre de décision :

```
La tache est-elle un processus clair, etape par etape ?
  Oui --> Sequentiel

Y a-t-il des sous-taches independantes pouvant s'executer
simultanement ?
  Oui --> Parallele

La sortie a-t-elle besoin d'une amelioration iterative ?
  Oui --> Boucle

Le type de tache determine-t-il quel specialiste la gere ?
  Oui --> Routage

La tache est-elle complexe et necessite-t-elle planification
et delegation ?
  Oui --> Hierarchique

La tache beneficie-t-elle de perspectives multiples et de debat ?
  Oui --> Chat de groupe
```

---

## Composer des patterns

Les systèmes réels imbriquent souvent des patterns. Voici un exemple de système de création de contenu :

```
SequentialAgent("content_pipeline")
  |
  +-- ParallelAgent("research")
  |     +-- web_search_agent
  |     +-- database_query_agent
  |     +-- document_review_agent
  |
  +-- LlmAgent("writer")
  |     (utilise les resultats de la recherche pour rediger le contenu)
  |
  +-- LoopAgent("refinement")
        +-- editor_agent
        +-- fact_checker_agent
        (boucle jusqu'a ce que les deux approuvent)
```

Cela combine la recherche parallèle, la progression séquentielle, et le raffinement itératif en un seul système. Dans ADK, chacun de ces agents de workflow peut contenir des agents LLM, d'autres agents de workflow, ou des agents personnalisés.

---

## L'orchestration sur Google Cloud avec ADK

L'Agent Development Kit de Google fournit trois types d'agents de workflow intégrés pour l'orchestration déterministe, plus une coordination pilotée par LLM pour les scénarios dynamiques.

### Les agents de workflow intégrés

| Type d'agent | Flux de contrôle | Classe ADK |
|-----------|-------------|-----------|
| Séquentiel | Exécuter les agents dans l'ordre | `SequentialAgent` |
| Parallèle | Exécuter les agents simultanément | `ParallelAgent` |
| Boucle | Répéter jusqu'à ce qu'une condition soit remplie | `LoopAgent` |

Ceux-ci sont déterministes - aucun LLM n'est impliqué dans les décisions d'orchestration. Le LLM n'est utilisé qu'au sein des sous-agents individuels pour leurs tâches spécifiques.

### La coordination pilotée par LLM

Pour le routage dynamique, utilisez un `LlmAgent` parent (aussi appelé `Agent`) avec des sous-agents. Le parent utilise son LLM pour décider à quel sous-agent déléguer en fonction de la conversation et de l'état actuel. C'est ainsi que vous implémentez les patterns de routage et hiérarchiques.

### Les agents personnalisés

Pour une logique d'orchestration qui ne correspond pas aux types intégrés, vous pouvez étendre `BaseAgent` pour créer des agents personnalisés avec un flux de contrôle arbitraire.

### Agent en tant qu'outil

ADK vous permet d'envelopper n'importe quel agent comme un outil via `AgentTool`. Cela permet à un agent coordinateur d'appeler des sous-agents comme s'ils étaient des appels de fonction, recevant des résultats structurés en retour.

Pour les détails d'implémentation complets, consultez :
- [Les agents de workflow ADK](https://google.github.io/adk-docs/agents/workflow-agents/)
- [Les systèmes multi-agents ADK](https://google.github.io/adk-docs/agents/)
- [Patterns multi-agents dans ADK - blog des développeurs Google](https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/)

---

## Comparaison de frameworks

ADK est l'un des nombreux frameworks qui fournissent des capacités d'orchestration. Voici comment les principales options se comparent :

| Framework | Approche | Forces | Considérations |
|-----------|----------|-----------|---------------|
| **Google ADK** | Trois primitives déterministes (Sequential, Parallel, Loop) + coordination pilotée par LLM | Séparation nette entre workflow et raisonnement. Déploiement vers Vertex AI Agent Engine. | Framework plus récent, communauté plus petite que LangChain |
| **LangGraph** | Workflow basé sur un graphe avec des nœuds et des arêtes | Meilleure prise en charge pour le branchement complexe et la logique conditionnelle. Observabilité mature. | Courbe d'apprentissage plus raide |
| **CrewAI** | Modèle basé sur les rôles où les agents sont définis comme des membres d'équipe | Le temps de mise en valeur le plus rapide. Configuration intuitive pilotée par YAML. | Peut manquer de sophistication pour les scénarios d'entreprise complexes |
| **AutoGen** (Microsoft) | Architecture conversationnelle avec jeu de rôle dynamique | Bon pour l'intervention humaine et les conversations à plusieurs parties. | Complexité de configuration significative pour la production |
| **Claude Agent SDK** | Orchestrateur-travailleur avec des context windows isolées | Les sous-agents utilisent un contexte isolé, ne renvoyant que l'information pertinente. | Spécifique à Anthropic |

Le choix dépend de vos priorités : ADK si vous voulez une intégration Vertex AI et des primitives de workflow claires, LangGraph si vous avez besoin de flux complexes basés sur des graphes, CrewAI si vous voulez une configuration rapide avec des équipes basées sur des rôles.

---

## Bonnes pratiques

### Commencer simple, ajouter de la complexité quand c'est nécessaire

Un agent unique avec de bons outils surpasse souvent un système multi-agent avec une mauvaise orchestration. Commencez avec l'approche la plus simple qui fonctionne :

1. Agent unique avec des outils
2. Pipeline séquentiel (si vous avez besoin d'étapes)
3. Exécution parallèle (si vous avez besoin de vitesse)
4. Coordination multi-agent complète (si vous avez besoin de spécialisation)

Ne sautez pas vers un système multi-agent hiérarchique parce que ça sonne impressionnant. N'ajoutez des agents que quand un agent unique ne peut manifestement pas gérer la tâche.

### Faire correspondre le modèle à la tâche

Tous les agents de votre orchestration n'ont pas besoin du même modèle. Un routeur de classification peut utiliser un modèle rapide et bon marché (Gemini Flash-Lite). Un agent de raisonnement complexe devrait utiliser un modèle performant (Gemini Pro). Cela économise un coût significatif.

### Fixer des limites d'itération

Toute boucle ou orchestration récursive doit avoir un nombre maximal d'itérations. Sans cela, un agent qui ne satisfait jamais ses propres critères tournera indéfiniment. Une bonne valeur par défaut est de 3 à 5 itérations pour les boucles de raffinement.

### Valider entre les étapes

Dans un pipeline séquentiel, validez la sortie de chaque agent avant de la transmettre au suivant. Un résultat malformé ou hors sujet de l'agent A se propagera à travers les agents B et C, gaspillant des tokens et produisant des résultats inutiles.

### Gérer le contexte à travers les agents

Dans les systèmes multi-agents, les context windows grandissent vite. Stratégies pour garder cela sous contrôle :

- Résumer les sorties avant de les transmettre entre agents
- Utiliser des magasins d'état externes pour les grandes données partagées
- Ne donner à chaque agent que le contexte dont il a besoin, pas tout
- Utiliser la compaction de contexte (fenêtres glissantes, synthèse) pour les tâches de longue durée

### Instrumenter pour l'observabilité

Suivez la performance par agent et par exécution d'orchestration :
- Latence par étape
- Usage de tokens par agent
- Taux de succès/échec par étape
- Taux d'achèvement de tâche de bout en bout

Utilisez le traçage distribué (par ex., OpenTelemetry) pour suivre une requête à travers plusieurs agents. C'est essentiel pour le débogage quand les choses tournent mal.

Consultez la [documentation de traçage ADK](https://google.github.io/adk-docs/) et [Google Cloud Trace](https://cloud.google.com/trace) pour des directives d'implémentation.

### Concevoir pour l'échec

Les agents échouent. Les outils renvoient des erreurs. Les LLM hallucinent. Votre orchestrateur doit gérer cela avec élégance :

- **Nouvelle tentative avec backoff** pour les erreurs transitoires (timeouts d'API, limites de débit)
- **Stratégies de repli** pour les échecs persistants (essayer un outil différent, utiliser un modèle plus simple)
- **Circuit breakers** pour prévenir les échecs en cascade
- **Escalade humaine** en dernier recours pour les tâches critiques

---

## Anti-patterns courants

| Anti-pattern | Problème | Correctif |
|-------------|---------|-----|
| Orchestration excessive | Utiliser un système multi-agent pour une tâche qu'un agent unique peut gérer | Commencer avec un agent, en ajouter d'autres seulement si nécessaire |
| Agents sans spécialisation | Plusieurs agents qui font tous à peu près la même chose | Donner à chaque agent un rôle et une expertise clairement distincts |
| État mutable partagé | Des agents concurrents écrivant dans le même état, causant des conditions de course | Utiliser des messages immuables ou un verrouillage d'état approprié |
| Aucune limite d'itération | Des boucles qui tournent indéfiniment quand la condition de sortie n'est jamais remplie | Toujours fixer max_iterations |
| Saturation de la context window | Faire passer l'historique complet de conversation à travers chaque agent d'un pipeline | Résumer et élaguer entre les étapes |
| Déterministe alors qu'un flux dynamique est nécessaire | Utiliser un pipeline fixe pour des tâches nécessitant un raisonnement adaptatif | Utiliser un routage piloté par LLM pour les types de tâches imprévisibles |
| Dynamique alors qu'un flux déterministe suffirait | Utiliser un routage LLM pour des tâches avec une séquence claire et connue | Utiliser des agents de workflow pour économiser du coût et améliorer la fiabilité |

---

## Points clés à retenir

- L'orchestrateur gère le flux de contrôle, l'assemblage du contexte, l'état, et la gestion des erreurs
- Deux types fondamentaux : déterministe (prévisible, bon marché, limité) et piloté par LLM (flexible, coûteux, moins prévisible)
- La plupart des systèmes en production utilisent un hybride - une structure déterministe avec une flexibilité LLM au sein des étapes
- Patterns centraux : séquentiel, parallèle, boucle, routage, hiérarchique, chat de groupe
- Les patterns se composent - imbriquez-les pour construire des systèmes complexes à partir de pièces simples
- ADK fournit SequentialAgent, ParallelAgent, et LoopAgent pour l'orchestration déterministe, plus une coordination pilotée par LLM pour le routage dynamique
- Commencez simple. Un seul agent bien équipé vaut mieux qu'une équipe mal orchestrée.
- Fixez des limites d'itération, validez entre les étapes, gérez le contexte de façon proactive, et concevez pour l'échec

---

## Pour aller plus loin

- [Les agents de workflow ADK](https://google.github.io/adk-docs/agents/workflow-agents/)
- [Patterns multi-agents dans ADK](https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/)
- [Anthropic - construire des agents IA efficaces](https://www.anthropic.com/research/building-effective-agents)
- [Microsoft Azure - Patterns d'orchestration d'agents IA](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [Documentation LangGraph](https://langchain-ai.github.io/langgraph/)
- [Documentation CrewAI](https://docs.crewai.com/)

---

[Leçon précédente : Agent skills](../17-agent-skills/README.md) | [Retour à la vue d'ensemble de la formation](../README.md)
