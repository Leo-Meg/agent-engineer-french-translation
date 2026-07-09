# Leçon 12 : premiers pas avec Vertex AI et ADK

## Introduction

Au cours des onze premières leçons, nous avons construit des bases solides sur le fonctionnement des agents - raisonnement, outils, mémoire, planification, systèmes multi-agents, RAG, évaluation, sécurité, et opérations de production. Tous ces concepts sont indépendants de toute plateforme.

Nous passons maintenant à la pratique. Cette leçon relie ces concepts aux outils et services spécifiques disponibles sur Google Cloud. L'objectif est de vous donner une vue claire de ce qui existe, de la façon dont les pièces s'assemblent, et du service à utiliser selon le type d'agent que vous construisez.

### ELI5 : pensez à la stack Google Cloud AI comme à un atelier

Imaginez que vous installiez un atelier de menuiserie. Vous avez besoin de matières premières (bois, métal), d'outils électriques (scies, perceuses), d'un établi (une surface stable pour construire), d'équipement de sécurité (lunettes, gants), et d'un espace pour exposer ou vendre vos produits finis.

La stack Google Cloud AI fonctionne de manière similaire :

- **Les modèles Gemini** sont vos matières premières - l'intelligence qui alimente tout
- **ADK (Agent Development Kit)** est votre panoplie d'outils électriques - le framework que vous utilisez pour construire des agents
- **La plateforme Vertex AI** est votre établi - hébergement de modèles, évaluation, et infrastructure de déploiement
- **Agent Engine** est une salle d'exposition managée - il fait tourner vos agents en production sans que vous ayez à gérer des serveurs
- **Model Armor** est votre équipement de sécurité - guardrails et filtrage de contenu
- **Vertex AI Search et RAG Engine** sont votre bibliothèque de référence - donnant aux agents l'accès à vos données

Vous pouvez utiliser ces éléments indépendamment ou ensemble. Tous les projets n'ont pas besoin de tous les outils.

> **Point clé à retenir :** Google Cloud fournit une stack complète pour construire des agents, des modèles jusqu'à l'exécution managée. Comprendre quel élément fait quoi vous aide à choisir le bon outil pour chaque tâche.

---

## La stack Google Cloud AI pour les agents

Voici une carte des composants majeurs et de leurs relations :

```
+------------------------------------------------------------------+
|                   Votre application d'agent                       |
+------------------------------------------------------------------+
|                                                                    |
|  +--------------------+    +----------------------------------+   |
|  |  Agent Development |    |  Agent Engine                    |   |
|  |  Kit (ADK)         |    |  (execution managee)             |   |
|  |                    |    |                                  |   |
|  |  - Construire des  |    |  - Deployer et executer les      |   |
|  |    agents          |--->|    agents                        |   |
|  |  - Definir des     |    |  - Gestion de session            |   |
|  |    outils          |    |  - Mise a l'echelle et monitoring|   |
|  |  - Orchestration   |    |                                  |   |
|  +--------------------+    +----------------------------------+   |
|           |                            |                          |
|           v                            v                          |
|  +----------------------------------------------------+          |
|  |              Plateforme Vertex AI                    |          |
|  |                                                     |          |
|  |  - Hebergement de modeles (Gemini, modeles          |          |
|  |    partenaires)                                     |          |
|  |  - Outils d'evaluation                              |          |
|  |  - Mise en cache de contexte                        |          |
|  |  - Ancrage avec Google Search                       |          |
|  +----------------------------------------------------+          |
|           |                            |                          |
|           v                            v                          |
|  +----------------------+    +------------------------+           |
|  | Vertex AI Search &   |    |  Model Armor           |           |
|  | RAG Engine           |    |  (Securite & Guardrails)|          |
|  +----------------------+    +------------------------+           |
|                                                                    |
+------------------------------------------------------------------+
|                     Modeles Gemini                                 |
|  Pro (raisonnement complexe) | Flash (equilibre) | Flash-Lite (volume) |
+------------------------------------------------------------------+
```

Parcourons chaque composant.

---

## Les modèles Gemini

Gemini est la famille de modèles d'IA multimodaux de Google. Pour le développement d'agents, vous travaillerez principalement avec trois paliers :

| Modèle | Idéal pour | Caractéristiques |
|-------|----------|----------------|
| **Gemini Pro** | Raisonnement complexe, planification multi-étapes, décisions nuancées | Capacité la plus élevée, latence plus élevée, coût plus élevé |
| **Gemini Flash** | Tâches équilibrées - utilisation d'outils, synthèse, conversation | Bonne capacité, rapide, coût modéré |
| **Gemini Flash-Lite** | Tâches simples à haut volume - classification, routage, extraction | Rapide, coût le plus bas, adapté aux cas d'usage à haut débit |

### Choisir le bon modèle

Pensez au choix du modèle comme au choix du bon véhicule pour un voyage :

- **Flash-Lite** est un vélo - rapide, bon marché, idéal pour les courts trajets (classification simple, détection d'intention, extraction basique)
- **Flash** est une voiture - polyvalente, bonne pour la plupart des trajets (tâches générales d'agent, utilisation d'outils, RAG, conversation)
- **Pro** est un camion - puissant, gère les charges lourdes (raisonnement multi-étapes complexe, documents longs, planification difficile)

La plupart des agents en production utilisent plusieurs paliers de modèles via le **model routing** (couvert dans la leçon 11). Utilisez Flash-Lite pour les étapes simples, Flash pour la logique centrale, et Pro seulement quand la tâche le nécessite réellement.

### Les capacités multimodales

Les modèles Gemini peuvent traiter du texte, des images, de l'audio, et de la vidéo. Cela signifie que vos agents peuvent :

- Analyser des images téléversées par les utilisateurs (photos de produits, captures d'écran, documents)
- Traiter des entrées audio (commandes vocales, enregistrements de réunions)
- Comprendre du contenu vidéo (tutoriels, démonstrations)
- Travailler nativement avec des PDF et d'autres formats de documents

C'est un avantage significatif par rapport aux modèles texte uniquement, car cela vous permet de construire des agents qui interagissent avec le monde réel de manières plus riches.

Pour les détails et capacités des modèles, consultez la [documentation des modèles Gemini](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/models).

---

## La plateforme Vertex AI

Vertex AI est la plateforme de machine learning de Google Cloud. Pour les développeurs d'agents, les capacités les plus pertinentes sont :

### L'hébergement de modèles et l'accès API

Vertex AI fournit un accès API aux modèles Gemini ainsi qu'à des modèles partenaires et open source. Vous obtenez :

- **Des endpoints managés** - aucune infrastructure à gérer
- **La mise en cache de contexte** - mettre en cache de longs prompts système ou documents pour réduire le coût et la latence sur les appels répétés
- **L'ancrage avec Google Search** - laisser le modèle vérifier et compléter ses connaissances avec des résultats de recherche web
- **La prédiction par lots** - traiter efficacement de grands volumes de requêtes

### Les outils d'évaluation

Vertex AI inclut des capacités d'évaluation qui s'alignent avec ce que nous avons couvert dans la leçon 9 :

- **AutoSxS (Auto Side-by-Side)** - comparer automatiquement deux versions de modèle
- **L'évaluation ponctuelle (pointwise)** - noter des réponses individuelles sur des dimensions de qualité
- **Des métriques personnalisées** - définir vos propres critères d'évaluation
- **L'évaluation en masse (bulk)** - exécuter des evals sur de grands ensembles de données

Ces outils s'intègrent avec votre pipeline CI/CD pour le déploiement conditionné par l'évaluation (leçon 11).

### Le Model Garden

Le Vertex AI Model Garden fournit un accès à un large éventail de modèles au-delà de Gemini - incluant des modèles open source et des modèles d'entreprises partenaires. C'est utile quand vous avez besoin de modèles spécialisés pour des tâches particulières ou que vous voulez comparer différentes options.

Pour une vue d'ensemble complète de la plateforme, consultez la [documentation Vertex AI](https://cloud.google.com/vertex-ai/docs).

---

## Agent Development Kit (ADK)

ADK est le framework open source de Google, axé sur le code, pour construire des agents IA. Si Vertex AI est l'établi, ADK est la panoplie d'outils électriques que vous utilisez pour réellement construire les choses.

### Caractéristiques clés

| Fonctionnalité | Détail |
|---------|--------|
| **Open source** | Disponible sur GitHub, sous licence MIT |
| **Multi-langage** | Python, TypeScript/JavaScript, Go, Java |
| **Indépendant du modèle** | Fonctionne avec Gemini, mais prend aussi en charge d'autres LLM |
| **Indépendant du déploiement** | S'exécute en local, sur Agent Engine, sur Cloud Run, sur n'importe quelle plateforme de conteneurs |
| **Structuré mais flexible** | Fournit une structure sans vous enfermer |

### Pourquoi un framework ?

Vous pourriez construire des agents en appelant directement l'API Gemini - en écrivant votre propre boucle d'appel d'outils, en gérant l'état de la conversation, et en traitant l'orchestration. ADK vous évite de réinventer ces patterns courants :

- L'enregistrement et l'exécution d'outils
- La gestion de l'état de conversation
- Les boucles d'orchestration multi-tours
- La coordination multi-agents
- La gestion de session et de mémoire
- Les hooks de callback pour les guardrails et la journalisation

Voyez cela comme la différence entre écrire des gestionnaires HTTP bruts et utiliser un framework web. Vous pouvez faire les deux, mais le framework gère le code répétitif afin que vous puissiez vous concentrer sur la logique unique de votre agent.

### Les concepts centraux d'ADK

ADK organise les agents en trois catégories :

#### 1. Les agents LLM

Ce sont des agents alimentés par un modèle de langage capable de raisonner, planifier, et décider quels outils appeler. C'est le type le plus courant, et il correspond aux agents de type ReAct que nous avons couverts dans la leçon 4.

```python
from google.adk.agents import Agent
from google.adk.tools import FunctionTool

# Definir un outil
def get_weather(city: str) -> str:
    """Recupere la meteo actuelle pour une ville."""
    # En pratique, ceci appellerait une API meteo
    return f"Le temps a {city} est ensoleille, 22C."

# Creer un agent
weather_agent = Agent(
    name="weather_agent",
    model="gemini-2.0-flash",
    instruction="Tu es un assistant meteo serviable. Utilise l'outil "
                "get_weather pour repondre aux questions sur les "
                "conditions meteorologiques.",
    tools=[get_weather],
)
```

#### 2. Les agents de workflow

Ces agents suivent des patterns d'orchestration prédéfinis plutôt que de s'appuyer sur le LLM pour décider du flux. ADK fournit trois types de workflow intégrés :

| Type de workflow | Comment ça marche | Quand l'utiliser |
|--------------|-------------|----------|
| **SequentialAgent** | Exécute les sous-agents les uns après les autres dans un ordre fixe | Les étapes doivent se dérouler en séquence (par ex., valider -> traiter -> répondre) |
| **ParallelAgent** | Exécute les sous-agents simultanément | Les étapes sont indépendantes et peuvent se dérouler en même temps (par ex., rechercher dans plusieurs sources) |
| **LoopAgent** | Exécute un sous-agent de manière répétée jusqu'à ce qu'une condition soit remplie | Raffinement itératif (par ex., générer -> évaluer -> améliorer) |

```python
from google.adk.agents import SequentialAgent

# Un pipeline qui valide l'entree, la traite, et met en forme la reponse
pipeline = SequentialAgent(
    name="order_pipeline",
    sub_agents=[
        input_validator_agent,
        order_processor_agent,
        response_formatter_agent,
    ],
)
```

Les agents de workflow correspondent directement aux design patterns agentiques de la leçon 4. Sequential correspond aux patterns de pipeline. Parallel correspond au fan-out/fan-in. Loop correspond à la reflection et au raffinement itératif.

#### 3. Les agents personnalisés

Pour les patterns d'orchestration qui ne correspondent pas aux types intégrés, vous pouvez créer des agents personnalisés en dérivant la classe d'agent de base et en implémentant votre propre flux de contrôle.

### L'écosystème d'outils ADK

L'une des plus grandes forces d'ADK est son écosystème d'outils. Les outils sont la façon dont les agents interagissent avec le monde extérieur (leçon 3), et ADK fournit plusieurs façons de les définir :

| Type d'outil | Ce que c'est | Exemple |
|-----------|-----------|---------|
| **Les outils de fonction** | De simples fonctions Python/JS décorées comme des outils | Une fonction qui interroge votre base de données |
| **Les outils MCP** | Des outils provenant de serveurs Model Context Protocol | Se connecter à n'importe quel serveur d'outils compatible MCP |
| **Les outils OpenAPI** | Générés automatiquement à partir de spécifications OpenAPI/Swagger | Envelopper n'importe quelle API REST comme outil d'agent |
| **Les outils intégrés** | Intégrations préconstruites fournies par ADK | Google Search, exécution de code, RAG |

ADK inclut plus de 60 intégrations d'outils préconstruites, couvrant des besoins courants comme :

- Google Search et la navigation web
- L'exécution de code (dans un sandbox)
- Les opérations sur fichiers
- Les requêtes de base de données
- Les appels d'API
- Google Workspace (Gmail, Calendar, Drive)

```python
from google.adk.tools import FunctionTool

# Un simple outil de fonction
def search_products(query: str, max_results: int = 5) -> list[dict]:
    """Recherche dans le catalogue produits.

    Args:
        query: La chaine de requete de recherche.
        max_results: Nombre maximum de resultats a renvoyer.

    Returns:
        Une liste de produits correspondants avec nom, prix et description.
    """
    # Votre implementation ici
    return product_database.search(query, limit=max_results)

# ADK genere automatiquement le schema de l'outil a partir de la signature
# de la fonction et de la docstring, afin que le LLM sache comment
# l'appeler correctement.
```

### Les skills ADK

Les skills sont un concept plus récent d'ADK qui empaquette les capacités d'agent en unités autonomes et réutilisables. Voyez-les comme des « plugins » pour agents.

Les skills ont trois niveaux de complexité croissante :

| Niveau | Ce qu'il inclut | Exemple |
|-------|-----------------|---------|
| **L1 - Métadonnées** | Nom, description, et tags qui aident l'agent à comprendre quand utiliser la skill | « Cette skill gère la réservation de vols » |
| **L2 - Instructions** | Instructions détaillées sur comment l'agent devrait utiliser la skill | Guide étape par étape pour le flux de réservation |
| **L3 - Ressources** | Outils, sources de données, et sous-agents dont la skill a besoin | Outil d'API de recherche de vols, base de données de compagnies aériennes |

Les skills facilitent le partage et la composition de capacités d'agent à travers les équipes et les projets.

Pour la documentation complète d'ADK, consultez le [site de documentation ADK](https://google.github.io/adk-docs/).

---

## Agent Engine

Agent Engine est un service d'exécution managé sur Google Cloud pour déployer et faire tourner des agents. Si ADK est la façon dont vous construisez des agents, Agent Engine est la façon dont vous les faites tourner en production sans gérer l'infrastructure.

### Ce que fournit Agent Engine

| Capacité | Ce qu'elle fait |
|-----------|-------------|
| **Hébergement managé** | Faire tourner votre agent sans provisionner de serveurs ou de conteneurs |
| **Gestion de session** | Persistance intégrée de l'état de conversation |
| **Mise à l'échelle** | Mise à l'échelle automatique en fonction du trafic |
| **Monitoring** | Intégration avec Cloud Monitoring et Cloud Logging |
| **Sécurité** | Contrôle d'accès basé sur IAM, support de VPC Service Controls |

### Quand utiliser Agent Engine contre d'autres options de déploiement

| Option de déploiement | Idéal pour |
|-------------------|----------|
| **Agent Engine** | Les agents en production où vous voulez une infrastructure managée et ne voulez pas gérer vous-même la mise à l'échelle, la gestion de session, ou le déploiement |
| **Cloud Run** | Les agents qui nécessitent des environnements d'exécution personnalisés, des dépendances spécifiques, ou plus de contrôle sur le conteneur |
| **GKE (Kubernetes)** | Les agents qui font partie d'une architecture de microservices plus large déjà en cours d'exécution sur Kubernetes |
| **Local / auto-hébergé** | Le développement, le test, ou quand vous ne pouvez pas utiliser de services cloud |

Les agents ADK peuvent être déployés vers n'importe laquelle de ces cibles. Agent Engine est l'option la plus managée - vous lui donnez votre code d'agent et il gère le reste.

Pour les détails de déploiement, consultez la [documentation d'Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview).

---

## Vertex AI Search et RAG Engine

Dans la leçon 8, nous avons couvert comment les agents peuvent utiliser la génération augmentée par récupération (RAG) pour accéder à vos données. Vertex AI fournit des services managés pour cela :

### Vertex AI Search

Un service de recherche managé qui peut indexer et rechercher à travers :

- Des sites web
- Des documents non structurés (PDF, documents Word, HTML)
- Des données structurées (bases de données, tableurs)

Il gère le découpage (chunking), l'embedding, l'indexation, et la récupération - le pipeline RAG complet que nous avons discuté dans la leçon 8 - en tant que service managé.

### RAG Engine

RAG Engine fournit un pipeline de récupération managé spécifiquement conçu pour ancrer les réponses du LLM dans vos données :

- **L'ingestion de documents** - téléverser et traiter des documents automatiquement
- **Les stratégies de découpage** - des approches configurables pour diviser les documents
- **La recherche vectorielle** - embedding managé et recherche par similarité
- **L'intégration avec Gemini** - ancrage intégré pour les appels aux modèles Gemini

L'avantage de ces services managés est que vous n'avez pas à faire tourner votre propre base de données vectorielle, à gérer les embeddings, ou à construire des pipelines de récupération. Le compromis est moins de contrôle sur les détails.

Pour les capacités RAG, consultez la [présentation de RAG Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/rag-overview).

---

## Model Armor

Model Armor est le service de guardrails managé de Google Cloud (nous avons couvert les guardrails en détail dans la leçon 10). Il fournit :

| Fonctionnalité | Ce qu'elle fait |
|---------|-------------|
| **Le filtrage des prompts** | Détecter et bloquer les prompts nuisibles ou adverses avant qu'ils n'atteignent le modèle |
| **Le filtrage des réponses** | Filtrer les sorties du modèle à la recherche de contenu nuisible, toxique, ou inapproprié |
| **La détection d'injection de prompt** | Identifier les tentatives de contourner les instructions du modèle |
| **Des politiques configurables** | Définir vos propres seuils pour différentes catégories de contenu |
| **L'intégration** | Fonctionne avec les endpoints Vertex AI et peut être ajouté à n'importe quelle application d'IA générative |

Model Armor vous donne une défense de couche 2 prête pour la production (issue du modèle de défense en profondeur de la leçon 10) sans avoir à construire le filtrage de contenu à partir de zéro.

---

## Guide de démarrage rapide

Voici comment démarrer avec ADK et Google Cloud pour le développement d'agents.

### Prérequis

- Un compte Google Cloud (vous pouvez commencer avec le [niveau gratuit](https://cloud.google.com/free))
- Python 3.9+ (pour le SDK Python)
- Un projet Google Cloud avec la facturation activée

### Étape 1 : installer ADK

```bash
pip install google-adk
```

### Étape 2 : configurer l'authentification

Vous avez deux options pour vous authentifier :

**Option A : clé API (le plus simple pour démarrer)**
```bash
export GOOGLE_API_KEY="votre-cle-api-ici"
```

Vous pouvez obtenir une clé API depuis [Google AI Studio](https://aistudio.google.com/).

**Option B : projet Google Cloud (pour la production et les fonctionnalités Vertex AI)**
```bash
# Installer le CLI Google Cloud
# https://cloud.google.com/sdk/docs/install

# S'authentifier
gcloud auth application-default login

# Definir votre projet
gcloud config set project VOTRE_ID_PROJET
```

### Étape 3 : créer votre premier agent

Créez un fichier nommé `agent.py` :

```python
from google.adk.agents import Agent

root_agent = Agent(
    name="greeting_agent",
    model="gemini-2.0-flash",
    instruction="Tu es un assistant amical qui accueille les "
                "utilisateurs et repond aux questions de base.",
)
```

### Étape 4 : l'exécuter en local

```bash
adk web
```

Cela démarre une interface web locale où vous pouvez discuter avec votre agent et inspecter son comportement. L'interface de développement d'ADK vous montre les étapes de raisonnement de l'agent, les appels d'outils, et l'état - ce qui est inestimable pour le débogage.

### Étape 5 : ajouter des outils et de la complexité

À partir de là, vous pouvez ajouter des outils, créer des systèmes multi-agents, intégrer le RAG, et éventuellement déployer vers Agent Engine ou Cloud Run. Le [guide de démarrage ADK](https://google.github.io/adk-docs/get-started/) détaille ces étapes.

---

## Arbre de décision : quel service utiliser ?

Quand vous construisez un agent, utilisez cet arbre de décision pour choisir les bons services Google Cloud :

```
"Je veux construire..."
    |
    +-- "Un simple chatbot sans outils"
    |       --> l'API Gemini directement (aucun framework necessaire)
    |
    +-- "Un agent avec des outils et du raisonnement"
    |       --> ADK + Gemini Flash
    |       |
    |       +-- "...et j'en ai besoin en production"
    |               --> deployer vers Agent Engine ou Cloud Run
    |
    +-- "Un agent qui recherche dans mes documents"
    |       --> ADK + Vertex AI Search ou RAG Engine
    |
    +-- "Un systeme multi-agent"
    |       --> ADK (orchestration multi-agent integree)
    |
    +-- "Un agent avec des exigences de securite strictes"
    |       --> ADK + Model Armor + guardrails personnalises
    |
    +-- "Une application a haut volume et sensible aux couts"
    |       --> model routing (Flash-Lite pour les taches simples,
    |           Flash pour les taches complexes) + mise en cache
    |           de contexte
    |
    +-- "Un agent qui doit utiliser des API externes"
            --> ADK avec des outils OpenAPI ou des outils MCP
```

### Tableau de référence rapide

| J'ai besoin de... | Utilisez... |
|-----------|--------|
| Un LLM à appeler | Les modèles Gemini via Vertex AI ou AI Studio |
| Un framework pour construire des agents | L'Agent Development Kit (ADK) |
| Un hébergement d'agent managé | Agent Engine |
| Un hébergement de conteneur personnalisé | Cloud Run ou GKE |
| Une recherche de documents / RAG | Vertex AI Search ou RAG Engine |
| Des guardrails de sécurité de contenu | Model Armor |
| Une évaluation de modèle | Les outils d'évaluation Vertex AI |
| Une gestion de prompts | La gestion de prompts Vertex AI |
| L'interopérabilité avec des outils externes | Les outils MCP dans ADK |
| L'interopérabilité avec d'autres agents | Le protocole A2A (couvert dans la leçon 14) |

---

## Comment les pièces s'assemblent : un exemple complet

Voici comment un agent typique en production utilise plusieurs services Google Cloud ensemble :

```
L'utilisateur demande : "Quelle est la politique de retour pour ma
commande recente ?"

1. [Agent ADK] recoit la requete
       |
2. [Model Armor] filtre l'entree pour la securite
       |
3. [Gemini Flash] raisonne sur la requete :
       "Je dois consulter la commande et trouver la politique de retour"
       |
4. [Outil ADK : consultation de commande] appelle votre base de
       donnees de commandes
       |
5. [RAG Engine] recherche dans vos documents de politique
       la politique de retour pertinente
       |
6. [Gemini Flash] synthetise une reponse a partir des details
       de la commande et des documents de politique
       |
7. [Model Armor] filtre la sortie pour la securite
       |
8. [Agent Engine] gere l'etat de la session
       et renvoie la reponse a l'utilisateur
```

Chaque service Google Cloud gère une partie du puzzle. ADK orchestre le flux. Gemini fournit le raisonnement. RAG Engine fournit la connaissance. Model Armor fournit la sécurité. Agent Engine fournit l'exécution.

---

## Où chaque concept de leçon se rattache à Google Cloud

Voici une référence reliant les concepts des leçons précédentes à des services Google Cloud spécifiques :

| Leçon | Concept | Service Google Cloud |
|--------|---------|---------------------|
| 2 - Comment pensent les agents | Raisonnement LLM | Modèles Gemini |
| 3 - Les outils | Function calling | Outils de fonction ADK, outils MCP, outils OpenAPI |
| 4 - Design patterns | Orchestration | Agents Sequential/Parallel/Loop d'ADK |
| 5 - Mémoire | État de session, mémoire à long terme | Gestion de session ADK, Agent Engine |
| 6 - Planification | Raisonnement multi-étapes | Gemini Pro pour la planification complexe |
| 7 - Multi-agents | Coordination d'agents | Support multi-agent d'ADK |
| 8 - RAG | Récupération de connaissances | Vertex AI Search, RAG Engine |
| 9 - Évaluation | Tester les agents | Vertex AI Evaluation |
| 10 - Sécurité | Guardrails | Model Armor |
| 11 - Production | Déploiement, CI/CD | Agent Engine, Cloud Run, Agent Starter Pack |

---

## Points clés à retenir

1. **Google Cloud fournit une stack complète pour les agents.** Des modèles Gemini à la base jusqu'à Agent Engine au sommet, vous pouvez construire et déployer des systèmes d'agents complets.

2. **ADK est le framework axé sur le code.** Il est open source, multi-langage, indépendant du modèle, et indépendant du déploiement. Il gère les patterns courants (appel d'outils, orchestration, gestion d'état) afin que vous puissiez vous concentrer sur la logique de votre agent.

3. **Choisissez le bon palier de modèle.** Utilisez Flash-Lite pour les tâches simples, Flash pour la plupart du travail d'agent, et Pro pour le raisonnement complexe. Le model routing entre paliers est une stratégie clé d'optimisation des coûts.

4. **Les services managés réduisent la charge opérationnelle.** Agent Engine, RAG Engine, et Model Armor gèrent l'infrastructure et les opérations afin que vous puissiez vous concentrer sur la construction. Le compromis est moins de contrôle sur les détails d'implémentation.

5. **Tout est composable.** Vous pouvez utiliser ADK sans Agent Engine, utiliser Vertex AI sans ADK, ou utiliser la stack complète ensemble. Commencez avec ce dont vous avez besoin et ajoutez des services à mesure que vos exigences grandissent.

6. **Les agents ADK correspondent directement aux concepts de cette formation.** Les agents LLM pour le raisonnement, les agents de workflow pour les patterns d'orchestration, les outils pour l'interaction externe, et les skills pour les capacités réutilisables.

---

## Pour aller plus loin

- [Documentation ADK](https://google.github.io/adk-docs/) - Guide complet pour construire des agents avec ADK
- [Documentation Vertex AI](https://cloud.google.com/vertex-ai/docs) - La référence complète de la plateforme Vertex AI
- [Présentation d'Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview) - Exécution managée pour les agents
- [Présentation de RAG Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/rag-overview) - Génération augmentée par récupération managée
- [Agent Starter Pack](https://github.com/GoogleCloudPlatform/agent-starter-pack) - Modèles prêts pour la production avec CI/CD et observabilité intégrés
- [Google AI Studio](https://aistudio.google.com/) - Obtenir des clés API et expérimenter avec les modèles Gemini

---

Leçon suivante : [Construire votre premier agent](../13-building-your-first-agent/README.md)
