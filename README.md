# Agent Engineer - une formation pour ingénieurs logiciels

Apprenez les fondamentaux des agents IA et comment les construire avec Google Cloud AI.

## À qui s'adresse cette formation ?

Aux ingénieurs logiciels qui souhaitent comprendre ce que sont les agents IA, comment ils fonctionnent, et comment les construire. Aucune expérience préalable en IA/ML (apprentissage automatique) n'est requise - juste de la curiosité et quelques notions de Python.

## Vue d'ensemble de la formation

Cette formation est divisée en trois parties :

**Partie 1 : Fondamentaux (101)** - Comprendre les concepts clés qui sous-tendent les agents IA. Ces leçons sont indépendantes de toute plateforme et visent à construire votre modèle mental.

**Partie 2 : Concevoir et déployer (201)** - Mettre ces fondamentaux en pratique avec Google Cloud AI, Vertex AI et l'Agent Development Kit (ADK).

**Partie 3 : Approfondissements (301)** - Aller plus loin sur des sujets spécifiques, essentiels au développement d'agents en conditions réelles.

## Leçons

### Partie 1 : fondamentaux

| # | Leçon | Ce que vous allez apprendre |
|---|--------|-------------------|
| 01 | [Qu'est-ce qu'un agent IA ?](./01-what-are-ai-agents/README.md) | La vue d'ensemble - ce que sont les agents, pourquoi ils comptent, et quand les utiliser |
| 02 | [Comment pensent les agents](./02-how-agents-think/README.md) | Les LLM comme moteur de raisonnement - comment les modèles planifient, décident et génèrent |
| 03 | [Outils - donner des mains aux agents](./03-tools-giving-agents-hands/README.md) | Function calling, conception d'outils, et connexion des agents au monde réel |
| 04 | [Design patterns agentiques](./04-agentic-design-patterns/README.md) | ReAct, reflection, planification, et autres patterns fondamentaux |
| 05 | [Mémoire et contexte](./05-memory-and-context/README.md) | Comment les agents se souviennent des choses - sessions, context windows, et mémoire à long terme |
| 06 | [Planification et raisonnement](./06-planning-and-reasoning/README.md) | Comment les agents décomposent des tâches complexes et prennent des décisions |
| 07 | [Systèmes multi-agents](./07-multi-agent-systems/README.md) | Quand un seul agent ne suffit pas - coordination, délégation et travail d'équipe |
| 08 | [RAG agentique](./08-agentic-rag/README.md) | Au-delà de la récupération classique - des agents qui recherchent, évaluent et affinent |
| 09 | [Évaluer et tester les agents](./09-evaluating-and-testing-agents/README.md) | Comment savoir si votre agent fonctionne vraiment - métriques, evals et observabilité |
| 10 | [Guardrails et sécurité](./10-guardrails-and-safety/README.md) | Garder des agents dignes de confiance - sécurité, alignment et IA responsable |

### Partie 2 : concevoir et déployer

| # | Leçon | Ce que vous allez apprendre |
|---|--------|-------------------|
| 11 | [Du prototype à la production](./11-from-prototype-to-production/README.md) | Le parcours de la démo au déploiement - CI/CD, rollout et opérations |
| 12 | [Premiers pas avec Vertex AI et ADK](./12-getting-started-with-vertex-and-adk/README.md) | La stack Google Cloud AI pour les agents - ce qui est disponible et comment tout s'articule |
| 13 | [Construire votre premier agent](./13-building-your-first-agent/README.md) | Pratique - construisez un agent fonctionnel avec ADK, étape par étape |
| 14 | [Protocoles d'agents - MCP et A2A](./14-agent-protocols-mcp-and-a2a/README.md) | Comment les agents communiquent avec les outils et entre eux via des standards ouverts |

### Partie 3 : approfondissements

| # | Leçon | Ce que vous allez apprendre |
|---|--------|-------------------|
| 15 | [AGENTS.md](./15-agents-md/README.md) | Donner aux agents IA de codage du contexte sur votre projet grâce à un fichier de configuration standard |
| 16 | [MCP en détail](./16-mcp-deep-dive/README.md) | Comment MCP fonctionne en coulisses, MCP face aux outils CLI, et considérations de sécurité |
| 17 | [Agent skills](./17-agent-skills/README.md) | Empaqueter une expertise métier réutilisable sous forme de modules de skills portables |
| 18 | [Orchestrateurs](./18-orchestrators/README.md) | Gérer le flux de contrôle des agents - patterns, frameworks et bonnes pratiques |
| 19 | [Pour aller plus loin](./19-where-to-go-from-here/README.md) | Ressources, codelabs, communauté et prochaines étapes |

## Comment utiliser cette formation

- **Lisez dans l'ordre** si vous découvrez les agents. Chaque leçon s'appuie sur la précédente.
- **Naviguez librement** si vous en maîtrisez déjà les bases. Chaque leçon est suffisamment autonome pour être lue indépendamment.
- **Suivez les liens** vers la documentation officielle, les codelabs (ateliers de code pratiques) et les tutoriels pour vous exercer. Nous renvoyons volontairement vers des ressources activement maintenues plutôt que de dupliquer une documentation d'API ou des exemples de code qui risqueraient de devenir obsolètes.

## Philosophie

Cette formation suit quelques principes :

- **Les analogies d'abord.** Nous utilisons des comparaisons du quotidien pour expliquer des concepts complexes avant d'entrer dans les détails techniques.
- **Les fondamentaux avant les frameworks.** Comprendre le « pourquoi » avant le « comment ». Les frameworks changent, mais les idées fondamentales perdurent.
- **Renvoyer vers les sources, ne pas dupliquer.** Pour les références d'API, les exemples de code et les instructions d'installation, nous renvoyons vers la documentation officielle de Google Cloud et ses codelabs. Cela permet à notre contenu de rester centré sur les concepts et garantit que vous consultez toujours une information à jour.
- **Une honnêteté sur les compromis.** Chaque choix d'architecture a un coût. Nous nous efforçons de présenter les deux faces de la médaille.

## Prérequis

- Connaissances de base en Python (fonctions, classes, requêtes HTTP)
- Un compte Google Cloud ([essai gratuit disponible](https://cloud.google.com/free))
- Une familiarité avec les API REST et JSON

## Ressources complémentaires

- [Documentation Google Cloud AI](https://cloud.google.com/ai)
- [Documentation Vertex AI](https://cloud.google.com/vertex-ai/docs)
- [Documentation de l'Agent Development Kit (ADK)](https://google.github.io/adk-docs/)
- [Codelabs Google Cloud AI](https://codelabs.developers.google.com/?cat=AI)
- [Documentation de l'API Gemini](https://ai.google.dev/docs)

## Contribuer

Une faute de frappe repérée ? Une suggestion ? Les pull requests (PR) et les issues sont les bienvenues. Consultez [CONTRIBUTING.md](./CONTRIBUTING.md) pour connaître les lignes directrices.

## Licence

Ce projet est distribué sous licence Apache 2.0 - consultez le fichier [LICENSE](./LICENSE) pour plus de détails.
