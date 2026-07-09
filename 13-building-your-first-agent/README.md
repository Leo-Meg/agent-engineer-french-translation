# Leçon 13 : construire votre premier agent avec ADK

## Introduction

Vous êtes arrivé au bout des fondamentaux. Vous comprenez ce que sont les agents, comment ils pensent, comment ils utilisent des outils, comment ils se souviennent des choses, et comment évaluer s'ils font du bon travail. Il est maintenant temps d'en construire un.

Dans cette leçon, nous allons parcourir la construction d'un agent pratique en utilisant l'Agent Development Kit (ADK) de Google. À la fin, vous comprendrez comment mettre en place un projet, définir un agent, lui donner des outils, le tester en local, et même construire une petite équipe d'agents qui travaillent ensemble.

Nous allons garder cela conceptuel et renvoyer vers le quickstart officiel pour les exemples de code exacts - ainsi vous avez toujours une syntaxe à jour, et nous pouvons nous concentrer sur les idées qui comptent.

---

## Ce que nous construisons

Notre objectif est un agent pratique capable de :

- Répondre à des questions sur un sujet en utilisant Google Search
- Appeler un outil personnalisé que vous définissez (comme consulter la météo ou des informations produit)
- S'exécuter en local afin que vous puissiez tester et itérer rapidement

Voyez cela comme un « hello world » pour les agents - suffisamment simple pour être compris en une seule session, mais suffisamment réel pour montrer comment tout s'articule.

### Prérequis

Avant de commencer, assurez-vous d'avoir :

- **Python 3.9+** installé
- **ADK installé** (`pip install google-adk`)
- **Un projet Google Cloud** avec l'API Gemini activée
- **Une clé API** ou des identifiants par défaut de l'application configurés

> **Configuration :** suivez le guide de configuration officiel sur [ADK Getting Started](https://google.github.io/adk-docs/get-started/) pour des instructions d'installation détaillées.

---

## ELI5 : que faisons-nous réellement ?

Construire votre premier agent, c'est comme assembler un kit Lego. Vous avez une plaque de base (la structure du projet), un cerveau de figurine (le modèle de langage), quelques accessoires-outils (recherche, fonctions personnalisées), et un manuel d'instructions (les instructions système) qui indique à la figurine comment se comporter.

Chaque pièce s'assemble d'une manière spécifique. Le cerveau décide quoi faire. Les outils lui permettent de faire des choses. Les instructions le gardent concentré. Et la plaque de base maintient tout en place pour que ça ne s'effondre pas.

Ce qui est formidable, c'est qu'une fois que vous comprenez comment les pièces s'assemblent, vous pouvez les échanger, en ajouter d'autres, ou les réarranger pour construire quelque chose de complètement différent.

---

## Parcours étape par étape

Parcourons le processus de construction de votre premier agent. Nous couvrirons les concepts à chaque étape et vous renverrons vers le quickstart officiel pour le code exact.

### Étape 1 : mettre en place la structure du projet

ADK attend une disposition de projet spécifique. Dans sa forme la plus simple, vous avez besoin d'un dossier pour votre agent avec deux fichiers à l'intérieur :

```
my_first_agent/
    __init__.py
    agent.py
```

Le nom du dossier devient le nom de module de votre agent. Le fichier `__init__.py` indique à Python qu'il s'agit d'un package et exporte votre agent. Le fichier `agent.py` est là où vous définissez l'agent lui-même.

Cette structure compte car l'outillage ADK (comme `adk web` et `adk eval`) découvre votre agent en recherchant ce pattern. Gardez-la propre et cohérente dès le départ.

> **Astuce :** vous pouvez aussi utiliser `adk create my_first_agent` pour générer automatiquement cette structure.

### Étape 2 : définir votre agent

Chaque agent ADK a besoin d'au minimum trois choses :

1. **Un nom** - un identifiant unique pour votre agent
2. **Un modèle** - quel modèle Gemini utiliser pour le raisonnement
3. **Des instructions système** - le prompt qui indique à l'agent qui il est et comment se comporter

Voici à quoi cela ressemble conceptuellement :

```python
from google.adk.agents import Agent

my_agent = Agent(
    name="my_first_agent",
    model="gemini-2.5-flash",
    instruction="Tu es un assistant utile qui repond aux questions "
                "de facon claire et concise.",
)
```

Le paramètre `model` détermine quel modèle Gemini gère le raisonnement. Pour l'apprentissage et le prototypage, `gemini-2.5-flash` est un excellent choix - il est rapide et économique. Pour des tâches de raisonnement plus complexes, vous pourriez passer à `gemini-2.5-pro`.

Le paramètre `instruction` est le prompt système de votre agent. C'est là que vous définissez sa personnalité, ses capacités, et ses limites. Nous verrons comment écrire de bonnes instructions plus loin dans cette leçon.

### Étape 3 : créer un outil de fonction personnalisé

Les outils sont ce qui différencie un agent d'un chatbot. Donnons à notre agent une fonction personnalisée qu'il peut appeler.

Un outil dans ADK est simplement une fonction Python avec une docstring claire. La docstring compte, car ADK l'utilise pour indiquer au modèle ce que fait l'outil et quand l'utiliser.

```python
def get_weather(city: str) -> dict:
    """Recupere la meteo actuelle pour une ville donnee.

    Args:
        city: Le nom de la ville pour laquelle obtenir la meteo.

    Returns:
        Un dictionnaire contenant l'information meteo.
    """
    # Dans un vrai agent, ceci appellerait une API meteo
    # Pour l'instant, renvoie des donnees fictives
    return {
        "city": city,
        "temperature": "22C",
        "conditions": "Ensoleille",
    }
```

Éléments clés à remarquer :

- **Les indications de type** (`city: str`, `-> dict`) aident le modèle à comprendre quels paramètres passer et à quoi s'attendre en retour
- **La docstring** est la façon dont le modèle apprend ce que fait cet outil - écrivez-la comme si vous expliquiez la fonction à un collègue
- **Le nom de la fonction** devrait être descriptif - le modèle l'utilise pour décider quand appeler l'outil

Vous attachez ensuite l'outil à votre agent :

```python
my_agent = Agent(
    name="my_first_agent",
    model="gemini-2.5-flash",
    instruction="Tu es un assistant utile. Utilise l'outil get_weather "
                "quand on te pose une question sur la meteo.",
    tools=[get_weather],
)
```

### Étape 4 : ajouter un outil intégré (google search)

ADK est livré avec plusieurs outils intégrés. L'un des plus utiles est l'ancrage Google Search, qui permet à votre agent de rechercher sur le web des informations actuelles.

```python
from google.adk.tools import google_search

my_agent = Agent(
    name="my_first_agent",
    model="gemini-2.5-flash",
    instruction="Tu es un assistant utile. Utilise la recherche pour "
                "l'actualite et get_weather pour la meteo.",
    tools=[get_weather, google_search],
)
```

Maintenant votre agent peut à la fois rechercher sur le web et vérifier la météo. Le modèle décide quel outil utiliser en fonction de la question de l'utilisateur. Demandez les actualités et il recherche. Demandez la météo et il appelle votre fonction personnalisée.

### Étape 5 : exécuter et tester en local

ADK inclut un serveur de développement local qui vous donne une interface web pour interagir avec votre agent :

```bash
adk web
```

Cela démarre un serveur local (généralement à `http://localhost:8000`) avec une interface de chat. Vous pouvez :

- Envoyer des messages à votre agent
- Voir quels outils il appelle et pourquoi
- Inspecter l'historique complet de la conversation
- Déboguer les problèmes en temps réel

Vous pouvez aussi tester depuis la ligne de commande :

```bash
adk run my_first_agent
```

C'est votre boucle de retour la plus rapide. Faites un changement, rafraîchissez, testez. Répétez jusqu'à ce que l'agent se comporte comme vous le souhaitez.

### Étape 6 : évaluer avec adk eval

Une fois votre agent fonctionnel, vous voulez vous assurer qu'il continue de fonctionner à mesure que vous faites des changements. ADK inclut un framework d'évaluation pour cela :

```bash
adk eval my_first_agent eval_data.json
```

L'évaluation vous permet de définir des cas de test - des paires d'entrées et de comportements attendus - et de vérifier automatiquement si votre agent les gère correctement. C'est l'équivalent, pour les agents, des tests unitaires.

Nous avons couvert les concepts d'évaluation en profondeur dans la leçon 9. L'idée clé ici est de commencer à écrire des cas d'eval tôt, même pour votre premier agent. Cela vous évite des régressions plus tard.

> **Quickstart complet :** suivez le code complet sur [ADK Quickstart](https://google.github.io/adk-docs/get-started/quickstart/)

---

## Les types d'agents dans ADK

ADK fournit quatre types d'agents, chacun conçu pour un type de tâche différent. Choisir le bon type, c'est comme choisir la bonne structure de données - cela façonne tout ce qui suit.

### LlmAgent (le type par défaut)

C'est le type d'agent que vous utiliserez le plus souvent. Il utilise un modèle de langage pour prendre des décisions sur la prochaine action.

**Quand l'utiliser :**
- La tâche nécessite du raisonnement et du jugement
- L'agent doit décider quels outils appeler en fonction du contexte
- L'interaction utilisateur est conversationnelle

**Comment ça marche :** le modèle reçoit le message de l'utilisateur, les outils disponibles, et l'historique de la conversation. Il décide s'il faut appeler un outil, poser une question de clarification, ou répondre directement. C'est la boucle ReAct en action.

```python
from google.adk.agents import Agent

researcher = Agent(
    name="researcher",
    model="gemini-2.5-flash",
    instruction="Tu recherches des sujets en profondeur en utilisant "
                "la recherche.",
    tools=[google_search],
)
```

### SequentialAgent (des étapes dans l'ordre)

Un SequentialAgent exécute une liste fixe de sous-agents les uns après les autres, comme un pipeline.

**Quand l'utiliser :**
- La tâche a des étapes claires et ordonnées
- Chaque étape dépend de la sortie de la précédente
- Vous voulez une exécution prévisible et reproductible

**Exemple :** un pipeline de création de contenu où un agent fait la recherche, le suivant écrit un brouillon, et le troisième relit pour la grammaire et le style.

```python
from google.adk.agents import SequentialAgent

pipeline = SequentialAgent(
    name="content_pipeline",
    sub_agents=[researcher, writer, editor],
)
```

**Analogie :** pensez à un SequentialAgent comme à une chaîne de montage dans une usine. Chaque poste fait un travail et transmet le résultat au poste suivant.

### ParallelAgent (des étapes en même temps)

Un ParallelAgent exécute plusieurs sous-agents simultanément et collecte leurs résultats.

**Quand l'utiliser :**
- Vous avez des sous-tâches indépendantes qui ne dépendent pas les unes des autres
- La vitesse compte et vous voulez réduire le temps réel écoulé
- Vous avez besoin de plusieurs perspectives sur la même entrée

**Exemple :** évaluer un morceau de code en lançant un relecteur sécurité, un relecteur performance, et un relecteur style, tous en même temps.

```python
from google.adk.agents import ParallelAgent

review_team = ParallelAgent(
    name="code_review",
    sub_agents=[security_reviewer, perf_reviewer, style_reviewer],
)
```

**Analogie :** pensez à un ParallelAgent comme à un brainstorming d'équipe où chacun travaille sur sa partie simultanément puis présente ses conclusions.

### LoopAgent (répéter jusqu'à terminé)

Un LoopAgent exécute ses sous-agents en cycle jusqu'à ce qu'une condition d'arrêt soit remplie.

**Quand l'utiliser :**
- La tâche nécessite un raffinement itératif
- Vous voulez continuer à améliorer la sortie jusqu'à ce qu'elle atteigne un seuil de qualité
- Le nombre d'itérations n'est pas connu à l'avance

**Exemple :** un agent de rédaction qui rédige du contenu, l'évalue par rapport à des critères, et le révise jusqu'à ce que le score de qualité dépasse un seuil.

```python
from google.adk.agents import LoopAgent

refiner = LoopAgent(
    name="iterative_writer",
    sub_agents=[drafter, evaluator, reviser],
    max_iterations=5,
)
```

**Analogie :** pensez à un LoopAgent comme à un cycle de relecture de code - vous écrivez, recevez un retour, révisez, et répétez jusqu'à ce que le relecteur approuve.

### Choisir le bon type d'agent

| Type d'agent | À utiliser quand | Exemple |
|---|---|---|
| LlmAgent | Un raisonnement flexible est nécessaire | Chatbot, assistant de recherche |
| SequentialAgent | Un pipeline fixe d'étapes | ETL, création de contenu |
| ParallelAgent | Des tâches parallèles indépendantes | Systèmes multi-relecteurs |
| LoopAgent | Raffinement itératif | Boucles d'amélioration de qualité |

Vous pouvez aussi combiner ces types. Un SequentialAgent pourrait avoir un LlmAgent comme l'une de ses étapes. Un LoopAgent pourrait contenir un ParallelAgent en son sein. Cette composabilité est l'une des forces d'ADK.

> **Pour en savoir plus :** [Les types d'agents ADK](https://google.github.io/adk-docs/agents/)

---

## Construire un agent multi-outils

La plupart des agents réels ont besoin de plus d'un outil. Voyons comment combiner plusieurs capacités.

### Combiner recherche, exécution de code, et outils personnalisés

```python
from google.adk.agents import Agent
from google.adk.tools import google_search

def look_up_product(product_id: str) -> dict:
    """Recherche les details d'un produit par ID.

    Args:
        product_id: L'identifiant unique du produit.

    Returns:
        Les details du produit incluant nom, prix et disponibilite.
    """
    # Appelez votre base de donnees/API produit ici
    return {"id": product_id, "name": "Widget Pro", "price": 29.99}

def calculate_discount(price: float, discount_percent: float) -> float:
    """Calcule le prix apres remise.

    Args:
        price: Le prix original.
        discount_percent: Le pourcentage de remise (par ex., 20 pour 20%).

    Returns:
        Le prix apres remise.
    """
    return price * (1 - discount_percent / 100)

shopping_agent = Agent(
    name="shopping_assistant",
    model="gemini-2.5-flash",
    instruction="""Tu es un assistant d'achat. Tu peux :
    - Rechercher sur le web des avis et comparaisons de produits
    - Rechercher des produits specifiques dans notre catalogue par ID
    - Calculer des remises sur les prix

    Recherche toujours le produit d'abord avant de calculer des remises.""",
    tools=[google_search, look_up_product, calculate_discount],
)
```

La clé des agents multi-outils est une instruction claire sur quand utiliser chaque outil. Sans guidance, le modèle pourrait rechercher sur le web alors qu'il devrait interroger votre base de données, ou l'inverse.

### Principes de conception d'outils

Quand vous construisez des outils pour votre agent, suivez ces recommandations :

1. **Un outil, un travail.** Chaque outil devrait bien faire une seule chose. Ne créez pas de méga-outil qui gère cinq opérations différentes.

2. **Des noms et docstrings descriptifs.** Le modèle choisit les outils en fonction de leurs noms et descriptions. `get_order_status` est meilleur que `fetch_data`.

3. **Des types de paramètres clairs.** Utilisez les indications de type. Le modèle doit savoir s'il faut passer une chaîne, un nombre, ou un objet.

4. **Une gestion des erreurs avec élégance.** Renvoyez des messages d'erreur utiles plutôt que de planter. Le modèle peut souvent récupérer s'il comprend ce qui n'a pas fonctionné.

5. **Déterministe quand c'est possible.** Les outils qui renvoient des résultats cohérents pour les mêmes entrées sont plus faciles à raisonner pour le modèle.

---

## Construire une équipe d'agents

Parfois, un seul agent ne peut pas tout gérer. Vous pourriez avoir besoin de spécialistes - un agent pour la recherche, un autre pour la rédaction, un troisième pour la vérification des faits. ADK vous permet de construire des équipes d'agents avec un agent racine qui délègue à des sous-agents.

### Le pattern de l'agent racine

```python
from google.adk.agents import Agent

# Agents specialistes
researcher = Agent(
    name="researcher",
    model="gemini-2.5-flash",
    instruction="Tu recherches des sujets en utilisant la recherche web. "
                "Renvoie des conclusions factuelles.",
    tools=[google_search],
)

writer = Agent(
    name="writer",
    model="gemini-2.5-flash",
    instruction="Tu ecris un contenu clair et engageant base sur les "
                "resultats de recherche.",
)

fact_checker = Agent(
    name="fact_checker",
    model="gemini-2.5-flash",
    instruction="Tu verifies les affirmations en recherchant des preuves "
                "a l'appui.",
    tools=[google_search],
)

# Agent racine qui orchestre
coordinator = Agent(
    name="content_team",
    model="gemini-2.5-pro",
    instruction="""Tu coordonnes une equipe de creation de contenu. Pour
    toute demande de contenu :
    1. Demande au researcher de rassembler l'information
    2. Demande au writer de creer du contenu base sur la recherche
    3. Demande au fact_checker de verifier les affirmations cles

    Delegue les taches aux membres de ton equipe et synthetise leur
    travail.""",
    sub_agents=[researcher, writer, fact_checker],
)
```

### Quand utiliser des sous-agents contre plusieurs outils

| Approche | Idéal pour | Compromis |
|---|---|---|
| Un agent, de nombreux outils | Les tâches où une seule chaîne de raisonnement fonctionne | Plus simple, mais l'agent pourrait peiner avec trop d'outils |
| Plusieurs sous-agents | Les tâches qui bénéficient de la spécialisation | Plus flexible, mais ajoute un surcoût de coordination |

**Règle générale :** si votre agent unique a plus de 10 à 15 outils et commence à se perdre sur lequel utiliser, envisagez de le diviser en sous-agents avec des ensembles d'outils ciblés.

### La communication entre agents

Les sous-agents dans ADK partagent un état de session, qui agit comme un espace de travail partagé. L'agent racine peut passer du contexte aux sous-agents, et les sous-agents peuvent stocker des résultats auxquels d'autres agents peuvent accéder.

Voyez cela comme un document partagé dans un projet d'équipe. Le coordinateur écrit le brief, le researcher ajoute ses conclusions, et le writer utilise ces conclusions pour rédiger le contenu.

---

## Conseils pour écrire de bonnes instructions système

L'instruction système (le prompt) est le facteur le plus important dans le comportement de votre agent. Voici des recommandations pratiques.

### Soyez précis sur le rôle

**Faible :**
```
Tu es un assistant utile.
```

**Meilleur :**
```
Tu es un agent de support client pour Acme Corp. Tu aides les clients
avec le suivi de commandes, les retours, et les questions produits.
Tu as acces a la base de donnees de commandes et peux rechercher des
commandes par ID ou par e-mail.
```

### Fixez des limites claires

Indiquez à l'agent ce qu'il doit et ne doit pas faire :

```
Tu gères UNIQUEMENT les questions sur les commandes, les retours, et les
produits. Pour les questions de facturation, dis au client de contacter
billing@acme.com.
Ne fais jamais de promesses sur les dates de livraison, sauf si le
systeme de suivi les confirme.
Ne partage jamais d'informations internes de tarification ou de cout.
```

### Fournissez des exemples

Montrez à l'agent comment vous voulez qu'il se comporte :

```
Quand un client demande le statut de sa commande, suis ce pattern :
1. Demande son ID de commande s'il ne l'a pas fourni
2. Recherche la commande avec l'outil get_order
3. Resume le statut en langage clair
4. Si la commande est en retard, excuse-toi et propose d'escalader

Exemple :
Client : "Ou est ma commande ?"
Toi : "Je serais ravi de vous aider a suivre votre commande. Pourriez-vous
me communiquer votre ID de commande ? Il devrait commencer par ORD- et
vous le trouverez dans votre e-mail de confirmation."
```

### Définissez le format de sortie

Si vous voulez des réponses structurées, dites-le :

```
Reponds toujours dans ce format :
- Commence par un resume en une phrase
- Fournis les details sous forme de puces
- Termine par une prochaine etape suggeree
```

### Erreurs d'instruction courantes

| Erreur | Pourquoi c'est un problème | Correctif |
|---|---|---|
| Trop vague (« sois utile ») | Le modèle n'a aucune guidance spécifique | Définir le rôle, le périmètre, et le comportement |
| Trop long (2000+ mots) | Les instructions clés se perdent dans le bruit | Rester ciblé, utiliser une structure |
| Aucune guidance sur les outils | Le modèle devine quand utiliser les outils | Expliquer quand chaque outil est approprié |
| Aucune gestion des erreurs | L'agent ne sait pas quoi faire quand les choses échouent | Ajouter des instructions de repli |
| Aucune limite | L'agent essaie de tout gérer | Définir ce qui est hors périmètre |

---

## Erreurs courantes des débutants

### 1. commencer trop complexe

Vous n'avez pas besoin d'un système multi-agent avec 10 outils dès le premier jour. Commencez avec un agent et un outil. Faites-le fonctionner parfaitement avant d'ajouter de la complexité.

### 2. négliger l'instruction système

Beaucoup de débutants se concentrent sur les outils et le code mais écrivent une instruction système d'une seule ligne. L'instruction est votre levier principal pour contrôler le comportement de l'agent. Investissez-y du temps.

### 3. ne pas tester les appels d'outils

Votre agent n'est fiable que dans la mesure où ses outils le sont. Testez chaque fonction d'outil indépendamment avant de la connecter à l'agent. Si `get_weather("Londres")` lève une exception, votre agent le fera aussi.

### 4. oublier de gérer les erreurs

Les outils échouent. Les API expirent. Les données manquent. Votre agent a besoin d'instructions sur quoi faire quand les choses tournent mal. Ajoutez la gestion des erreurs à la fois dans votre code d'outil et dans vos instructions système.

### 5. utiliser le mauvais modèle

Toutes les tâches n'ont pas besoin du modèle le plus puissant. Pour le routage simple et l'appel d'outils, un modèle plus léger comme Flash fonctionne bien et économise de l'argent. Réservez les modèles plus grands aux tâches qui nécessitent réellement un raisonnement complexe.

### 6. sauter l'évaluation

Si vous n'avez pas de cas d'eval, vous ne savez pas si vos changements améliorent ou cassent les choses. Écrivez au moins 5 à 10 cas de test avant de commencer à itérer sur les prompts.

### 7. mettre des secrets dans le code

Ne codez jamais en dur des clés d'API ou des identifiants dans le code de votre agent. Utilisez des variables d'environnement ou un gestionnaire de secrets. C'est une pratique standard d'ingénierie logicielle, mais il est facile de l'oublier pendant le prototypage.

---

## Pour tout assembler

Voici un modèle mental pour l'ensemble du processus de construction d'un agent ADK :

```
1. Definir l'objectif
   "Que doit accomplir cet agent ?"
        |
        v
2. Choisir le type d'agent
   LlmAgent ? Sequential ? Parallel ? Loop ?
        |
        v
3. Ecrire l'instruction systeme
   Role, perimetre, limites, exemples
        |
        v
4. Definir les outils
   Quelles actions l'agent doit-il pouvoir entreprendre ?
        |
        v
5. Assembler le tout
   Agent + modele + instruction + outils
        |
        v
6. Tester en local
   adk web / adk run
        |
        v
7. Ecrire des cas d'eval
   Entree -> comportement attendu
        |
        v
8. Iterer
   Ameliorer les instructions, ajouter des outils, affiner
        |
        v
9. Deployer
   Agent Engine ou votre propre infrastructure
```

Chaque étape alimente la suivante. Et les étapes 6 à 8 forment une boucle - vous les parcourrez plusieurs fois avant de passer à l'étape 9.

---

## Référence rapide : commandes CLI d'ADK

| Commande | Ce qu'elle fait |
|---|---|
| `pip install google-adk` | Installer ADK |
| `adk create <name>` | Générer la structure d'un nouveau projet d'agent |
| `adk web` | Démarrer le serveur de développement local avec interface web |
| `adk run <agent>` | Exécuter un agent depuis la ligne de commande |
| `adk eval <agent> <data>` | Exécuter des cas d'évaluation sur votre agent |
| `adk deploy` | Déployer votre agent vers Agent Engine |

---

## Pour aller plus loin

- **Premiers pas avec ADK :** [https://google.github.io/adk-docs/get-started/](https://google.github.io/adk-docs/get-started/)
- **Tutoriel quickstart complet :** [https://google.github.io/adk-docs/get-started/quickstart/](https://google.github.io/adk-docs/get-started/quickstart/)
- **Les types d'agents en détail :** [https://google.github.io/adk-docs/agents/](https://google.github.io/adk-docs/agents/)
- **Référence des outils :** [https://google.github.io/adk-docs/tools/](https://google.github.io/adk-docs/tools/)

---

## Points clés à retenir

1. **Un agent ADK a besoin de trois choses :** un nom, un modèle, et des instructions système. Les outils sont optionnels mais sont ce qui rend les agents véritablement utiles.

2. **Commencez avec LlmAgent.** Il gère la plupart des cas d'usage. Ne recourez à SequentialAgent, ParallelAgent, ou LoopAgent que lorsque vous avez une raison structurelle claire.

3. **Les instructions système sont votre principal levier de contrôle.** Soyez précis sur le rôle, fixez des limites, fournissez des exemples, et incluez des directives de gestion des erreurs.

4. **Testez tôt et souvent.** Utilisez `adk web` pour les tests interactifs et `adk eval` pour les vérifications de régression automatisées.

5. **Construisez de façon incrémentale.** Un agent, un outil, un cas d'eval. Faites fonctionner chaque élément avant d'ajouter le suivant.

---

## Et maintenant ?

Maintenant que vous savez construire un agent, vous devez comprendre comment les agents communiquent avec le monde plus large. Dans la prochaine leçon, nous explorerons deux protocoles importants - MCP et A2A - qui permettent aux agents de parler à des outils et entre eux via des standards ouverts.

[Suivant : Leçon 14 - Protocoles d'agents : MCP et A2A -->](../14-agent-protocols-mcp-and-a2a/README.md)
