# Leçon 3 : les outils - donner des mains aux agents

## Introduction

Dans la leçon 1, nous avons établi qu'un agent est un cerveau (LLM) plus des mains (des outils) plus une boucle de contrôle (l'orchestration). Dans la leçon 2, nous avons exploré le cerveau. Nous allons maintenant donner des mains à l'agent.

Sans outils, un LLM ne peut que générer du texte. Il peut raisonner sur une question, mais il ne peut pas consulter la réponse dans une base de données. Il peut planifier un déploiement, mais il ne peut pas réellement exécuter le script de déploiement. Il peut rédiger un e-mail, mais il ne peut pas l'envoyer.

Les outils sont ce qui comble le fossé entre penser et agir. Ils sont le mécanisme par lequel un agent interagit avec le monde extérieur - lire des données, appeler des API, exécuter du code, et entreprendre des actions.

Cette leçon couvre ce que sont les outils, les différents types d'outils, comment le function calling fonctionne en coulisses, les bonnes pratiques pour concevoir des outils, et les pièges courants qui font trébucher même les ingénieurs expérimentés.

---

## Pourquoi les agents ont besoin d'outils

Considérez ce scénario. Un utilisateur demande à votre agent : « Combien de bugs ouverts me sont assignés ? »

**Sans outils :**
Le LLM n'a aucun accès à votre outil de suivi de bugs. Il pourrait dire « Je n'ai pas accès à votre outil de suivi de bugs » (si vous avez de la chance) ou il pourrait halluciner un nombre (si vous n'en avez pas).

**Avec des outils :**
L'agent pense « Je dois consulter l'outil de suivi de bugs. » Il appelle l'outil `get_bugs(assignee="current_user", status="open")`. L'outil interroge votre instance Jira/Linear/GitHub Issues et renvoie `[{id: 1234, title: "Login timeout"}, {id: 1235, title: "CSS overflow on mobile"}]`. L'agent compte les résultats et répond : « Vous avez 2 bugs ouverts : Login timeout (#1234) et CSS overflow on mobile (#1235). »

Les outils transforment l'agent d'un partenaire de conversation intelligent en un assistant capable qui peut réellement accomplir des choses.

### Ce que les outils permettent

| Sans outils | Avec des outils |
|---|---|
| Répondre à partir des données d'entraînement (potentiellement obsolètes) | Répondre à partir de données en direct |
| Décrire ce qu'il faudrait faire | Le faire réellement |
| Raisonner sur du code | Exécuter du code et en observer le résultat |
| Deviner l'état actuel | Interroger et observer l'état réel |
| Rédiger un message | Envoyer le message |

---

## Types d'outils

Les outils se déclinent sous plusieurs formes. Comprendre les différents types vous aide à choisir la bonne approche pour votre agent.

### 1. les outils de fonction (function tools, fonctions personnalisées)

Ce sont des fonctions que vous définissez et que l'agent peut appeler. Vous contrôlez l'implémentation, les entrées, les sorties, et la gestion des erreurs.

**Exemples :**
- `search_database(query, filters)` - interroge votre base de données interne
- `create_ticket(title, description, priority)` - crée un ticket de support
- `send_notification(user_id, message)` - envoie une notification push
- `get_weather(city)` - récupère la météo depuis une API tierce
- `run_test_suite(test_path)` - exécute des tests et renvoie les résultats

Les outils de fonction sont le type le plus courant. C'est le moyen principal d'étendre les capacités d'un agent pour l'adapter à votre cas d'usage spécifique.

### 2. les outils intégrés (built-in tools, fournis par la plateforme)

Ce sont des outils fournis par la plateforme ou le framework que vous utilisez. Ils sont préconstruits et prêts à l'emploi.

**Outils intégrés de Google Cloud / Gemini :**

| Outil | Ce qu'il fait |
|---|---|
| **Google Search** | Ancre les réponses du modèle dans des résultats de recherche web en temps réel. Réduit les hallucinations en fournissant des informations à jour. |
| **Code Execution** | Exécute du code Python dans un environnement sandbox (bac à sable). Utile pour les calculs, l'analyse de données, et la génération de graphiques. |
| **URL Context** | Récupère et traite le contenu d'une URL donnée. |

**Quand utiliser des outils intégrés :**
- Quand la fonctionnalité dont vous avez besoin existe déjà comme outil intégré
- Quand vous voulez que la plateforme gère l'exécution, le sandboxing, et la mise à l'échelle
- Quand vous voulez éviter de maintenir votre propre implémentation de fonctionnalités courantes

**Quand construire des outils de fonction personnalisés à la place :**
- Quand vous devez accéder à vos propres systèmes et données
- Quand vous avez besoin d'une logique personnalisée ou de règles métier
- Quand les outils intégrés ne couvrent pas votre cas d'usage

### 3. les outils agents (agent-as-tool, agent en tant qu'outil)

C'est un pattern puissant où un agent utilise un autre agent comme outil. L'agent appelant délègue une sous-tâche à un agent spécialisé, récupère le résultat, et poursuit son propre workflow.

**Exemple :**

```
Agent principal (planificateur de voyage)
    |
    +-- appelle --> Agent de recherche de vols
    |                (spécialisé dans la recherche de vols)
    |
    +-- appelle --> Agent hôtel
    |                (spécialisé dans les réservations d'hôtel)
    |
    +-- appelle --> Agent activités
                     (spécialisé dans les activités locales)
```

L'agent principal coordonne la planification globale du voyage. Chaque sous-agent est un expert dans son domaine, avec ses propres outils et instructions. L'agent principal n'a pas besoin de savoir comment fonctionne la recherche de vols - il demande simplement à l'agent de recherche de vols et récupère les résultats.

**Quand utiliser des outils agents :**
- Quand une sous-tâche est suffisamment complexe pour justifier son propre agent avec des outils et des instructions spécialisés
- Quand vous voulez séparer les responsabilités et simplifier chaque agent
- Quand différentes sous-tâches bénéficient de modèles ou de configurations différents

---

## Le function calling expliqué étape par étape

Le function calling est le mécanisme central qui permet l'utilisation d'outils. Voici exactement comment cela fonctionne, étape par étape.

### Le déroulement

```
Étape 1 : vous définissez des outils et les envoyez au modèle avec le message de l'utilisateur
Étape 2 : le modèle décide s'il doit appeler un outil (et lequel)
Étape 3 : le modèle renvoie un appel d'outil structuré (nom de la fonction + arguments)
Étape 4 : VOTRE CODE exécute la fonction (le modèle ne l'exécute jamais)
Étape 5 : vous renvoyez le résultat de la fonction au modèle
Étape 6 : le modèle utilise le résultat pour générer une réponse (ou appeler un autre outil)
```

C'est essentiel à comprendre : **le modèle n'exécute pas les outils**. Il propose des appels d'outils. Votre code les exécute. Le modèle ne touche jamais directement votre base de données, vos API, ou vos systèmes. Vous gardez toujours le contrôle.

### Exemple étape par étape

Parcourons un exemple concret avec un agent météo.

**Étape 1 : définir l'outil et envoyer la requête**

```python
tools = [
    {
        "name": "get_weather",
        "description": "Récupère la météo actuelle pour une ville donnée. Renvoie la température, les conditions et l'humidité.",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "Le nom de la ville, par ex. 'Londres' ou 'San Francisco'"
                },
                "units": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "Unités de température"
                }
            },
            "required": ["city"]
        }
    }
]

# Envoyer au modèle avec le message de l'utilisateur
response = model.generate(
    messages=[{"role": "user", "content": "Quel temps fait-il à Tokyo ?"}],
    tools=tools
)
```

**Étapes 2-3 : le modèle renvoie un appel d'outil**

Le modèle ne répond pas avec du texte. Il renvoie plutôt un appel d'outil structuré :

```json
{
    "tool_call": {
        "name": "get_weather",
        "arguments": {
            "city": "Tokyo",
            "units": "celsius"
        }
    }
}
```

Le modèle a déterminé :
- Qu'il a besoin de l'outil météo (pas d'un autre outil)
- Que la ville est « Tokyo » (extraite du message de l'utilisateur)
- Que Celsius est probablement approprié pour Tokyo (déduit du contexte)

**Étape 4 : votre code exécute la fonction**

```python
# Ceci est VOTRE code - vous contrôlez ce qui se passe ici
def get_weather(city, units="celsius"):
    # Appelle une vraie API météo
    response = weather_api.get(city=city, units=units)
    return {
        "city": city,
        "temperature": response.temp,
        "conditions": response.conditions,
        "humidity": response.humidity
    }

# Exécute l'appel d'outil
result = get_weather(**tool_call.arguments)
# result = {"city": "Tokyo", "temperature": 18, "conditions": "Partiellement nuageux", "humidity": "65%"}
```

**Étapes 5-6 : renvoyer le résultat et obtenir une réponse**

```python
response = model.generate(
    messages=[
        {"role": "user", "content": "Quel temps fait-il à Tokyo ?"},
        {"role": "assistant", "tool_call": tool_call},
        {"role": "tool", "content": json.dumps(result)}
    ],
    tools=tools
)
# Le modèle répond : "Il fait 18 degrés Celsius à Tokyo, avec un temps
#                     partiellement nuageux et 65 % d'humidité."
```

### Appels d'outils multiples

Parfois, le modèle a besoin d'appeler plusieurs outils pour répondre à une question. Cela peut se produire de deux façons :

**Séquentielle (l'un après l'autre) :**
```
Utilisateur : "Compare la météo à Tokyo et à Londres"

Étape 1 : le modèle appelle get_weather(city="Tokyo")
Étape 2 : vous exécutez, renvoyez le résultat
Étape 3 : le modèle appelle get_weather(city="London")
Étape 4 : vous exécutez, renvoyez le résultat
Étape 5 : le modèle synthétise les deux résultats en une comparaison
```

**Parallèle (en même temps) :**
Certains modèles prennent en charge les appels d'outils parallèles, où le modèle propose plusieurs appels en une seule réponse. Pour la même comparaison météo, le modèle pourrait renvoyer à la fois `get_weather("Tokyo")` et `get_weather("London")` en une seule fois. Vous exécutez les deux, renvoyez les deux résultats, et le modèle les synthétise. C'est plus rapide, car vous économisez un aller-retour.

### La boucle complète de l'agent avec des outils

Dans un agent réel, les appels d'outils se produisent à l'intérieur d'une boucle : envoyer le message de l'utilisateur et les outils au modèle, vérifier si la réponse est une réponse textuelle (terminé) ou un appel d'outil (l'exécuter, ajouter le résultat à la conversation, et reboucler). Cela continue jusqu'à ce que le modèle décide qu'il dispose de suffisamment d'information pour répondre.

---

## Bonnes pratiques de conception d'outils

La façon dont vous concevez vos outils a un impact direct sur la performance de votre agent. Le modèle doit comprendre vos outils pour les utiliser correctement. Voici les principes les plus importants.

### 1. utiliser des noms clairs et descriptifs

Le nom de l'outil est la première chose que voit le modèle. Il doit immédiatement indiquer ce que fait l'outil.

| Mauvais nom | Bon nom | Pourquoi |
|---|---|---|
| `do_thing` | `search_knowledge_base` | Précise ce qu'il recherche |
| `api_call` | `create_support_ticket` | Décrit l'action et sa cible |
| `process` | `validate_email_address` | Clair sur ce qu'il traite et comment |
| `get_data` | `get_order_by_id` | Précise quelle donnée et comment la trouver |
| `run` | `execute_sql_query` | Explicite sur ce qui s'exécute |

### 2. responsabilité unique

Chaque outil doit bien faire une seule chose. Tout comme les fonctions dans votre code, les outils doivent avoir un objectif unique et clair.

**Mauvais : un seul outil qui fait tout**
```json
{
    "name": "manage_orders",
    "description": "Crée, lit, met à jour ou supprime des commandes",
    "parameters": {
        "action": {"type": "string", "enum": ["create", "read", "update", "delete"]},
        "order_id": {"type": "string"},
        "order_data": {"type": "object"}
    }
}
```

Le modèle doit maintenant déterminer à la fois quelle action utiliser ET quels paramètres chaque action nécessite. Cela entraîne des erreurs.

**Bon : des outils séparés pour chaque action**
```json
[
    {
        "name": "get_order",
        "description": "Recherche une commande par son ID. Renvoie le statut, les articles et les informations de livraison.",
        "parameters": {
            "order_id": {"type": "string", "description": "L'ID de la commande, par ex. 'ORD-12345'"}
        }
    },
    {
        "name": "create_order",
        "description": "Crée une nouvelle commande avec les articles spécifiés.",
        "parameters": {
            "items": {"type": "array", "description": "Liste des ID d'articles et des quantités"},
            "shipping_address": {"type": "string"}
        }
    },
    {
        "name": "cancel_order",
        "description": "Annule une commande existante. Fonctionne uniquement pour les commandes pas encore expédiées.",
        "parameters": {
            "order_id": {"type": "string"},
            "reason": {"type": "string", "description": "Raison de l'annulation"}
        }
    }
]
```

Chaque outil a un objectif clair avec des paramètres spécifiques. Le modèle peut facilement choisir le bon.

### 3. écrire des paramètres descriptifs

Les descriptions de paramètres indiquent au modèle quoi passer. Soyez précis sur le format, les contraintes, et les valeurs par défaut.

**Mauvais :**
```json
{
    "name": "search",
    "parameters": {
        "q": {"type": "string"},
        "n": {"type": "integer"}
    }
}
```

Le modèle doit deviner ce que signifient `q` et `n`.

**Bon :**
```json
{
    "name": "search_products",
    "parameters": {
        "query": {
            "type": "string",
            "description": "Requête de recherche pour le nom ou la description du produit, par ex. 'casque sans fil'"
        },
        "max_results": {
            "type": "integer",
            "description": "Nombre maximum de résultats à renvoyer. Plage : 1-50. Par défaut : 10"
        }
    }
}
```

### 4. renvoyer une sortie concise et utile

Les résultats des outils retournent dans la context window du modèle. Renvoyer trop de données gaspille des tokens et peut dérouter le modèle.

**Mauvais : renvoyer la réponse API brute avec tout**
```json
{
    "status": 200,
    "headers": {"content-type": "application/json", "x-request-id": "abc123", ...},
    "data": {
        "order": {
            "id": "ORD-12345",
            "internal_id": "a1b2c3d4-e5f6-7890",
            "created_at": "2024-03-15T10:30:00Z",
            "updated_at": "2024-03-15T14:22:00Z",
            "customer_id": "CUST-789",
            "customer_hash": "sha256:abcdef...",
            "status": "shipped",
            "status_code": 3,
            "status_history": [...50 entrées...],
            "items": [...détails complets des produits avec tous les champs...],
            "shipping": {...détails complets du transporteur...},
            "billing": {...détails complets de paiement...},
            "metadata": {...données de suivi internes...}
        }
    }
}
```

**Bon : renvoyer uniquement ce dont l'agent a besoin**
```json
{
    "order_id": "ORD-12345",
    "status": "shipped",
    "items": ["Casque sans fil x1", "Câble USB-C x2"],
    "tracking_number": "1Z999AA10123456784",
    "estimated_delivery": "20 mars 2026"
}
```

### 5. inclure des messages d'erreur clairs

Quand un appel d'outil échoue, le message d'erreur doit aider le modèle à comprendre ce qui n'a pas fonctionné et quoi essayer ensuite.

**Mauvaise erreur :**
```json
{"error": "Échec"}
```

**Bonne erreur :**
```json
{
    "error": "ORDER_NOT_FOUND",
    "message": "Aucune commande trouvée avec l'ID 'ORD-99999'. Veuillez vérifier l'ID de la commande et réessayer. Les ID de commande suivent le format ORD-XXXXX."
}
```

Le bon message d'erreur donne au modèle suffisamment d'information pour soit corriger son approche (peut-être a-t-il utilisé le mauvais format d'ID), soit informer clairement l'utilisateur.

### 6. documenter le comportement de l'outil dans la description

La description de l'outil est l'occasion d'indiquer au modèle quand et comment utiliser l'outil. Incluez :

- **Ce que fait l'outil** (action principale)
- **Quand l'utiliser** (conditions et déclencheurs)
- **Ce qu'il renvoie** (format de sortie)
- **Les limitations** (ce qu'il ne peut pas faire)

**Exemple :**
```json
{
    "name": "search_knowledge_base",
    "description": "Recherche dans la base de connaissances de l'entreprise les articles et la documentation pertinents. Utilisez cet outil quand l'utilisateur pose une question sur les politiques de l'entreprise, les fonctionnalités d'un produit, ou les processus internes. Renvoie les 5 articles les plus pertinents avec leurs titres, résumés et liens. Ne recherche pas sur des sites externes - utilisez web_search pour cela."
}
```

---

## Le problème d'intégration N x M

À mesure que les systèmes d'agents grandissent, l'intégration des outils devient un défi de taille. Voici pourquoi.

### Le problème

Imaginez que vous ayez N applications IA différentes (assistant de codage, agent de recherche, bot de support) et M services différents (Slack, Jira, GitHub, Salesforce). Sans standardisation, chaque application nécessite du code personnalisé pour s'intégrer à chaque service - cela représente N x M intégrations à construire et à maintenir.

### Pourquoi la standardisation compte

Des standards comme le **Model Context Protocol (MCP, protocole de contexte de modèle)** visent à résoudre ce problème en créant une interface commune entre les applications IA et les fournisseurs d'outils. Chaque application implémente le protocole une seule fois. Chaque fournisseur d'outils implémente le protocole une seule fois. Les nouvelles applications fonctionnent automatiquement avec tous les outils existants, et inversement.

**Voyez cela comme l'USB.** Avant l'USB, chaque appareil avait son propre câble propriétaire. Après l'USB, n'importe quel appareil pouvait se connecter à n'importe quel ordinateur. Les standards d'outils visent à devenir l'« USB des outils IA ».

Quand vous choisissez un framework ou construisez des outils, réfléchissez à si d'autres agents peuvent réutiliser vos outils (construisez-les comme des serveurs MCP pour la portabilité) et si votre agent peut utiliser des outils provenant d'autres sources (prenez en charge MCP pour une large compatibilité). Les standards sont encore en évolution, évaluez donc en fonction de votre calendrier.

---

## Pièges courants dans la conception d'outils

### 1. trop d'outils surchargent le contexte

Chaque définition d'outil occupe de l'espace dans la context window du modèle. Plus important encore, le modèle doit raisonner pour choisir quel outil utiliser parmi l'ensemble complet. Trop d'outils entraîne :

- **Une paralysie de la décision** : le modèle a du mal à choisir le bon outil
- **Une mauvaise sélection d'outil** : avec de nombreux outils similaires, le modèle choisit le mauvais
- **Un gaspillage de contexte** : les définitions d'outils prennent la place de l'historique de conversation utile

**Combien d'outils est trop d'outils ?**

Il n'y a pas de règle stricte, mais voici quelques recommandations :

| Nombre d'outils | Recommandation |
|---|---|
| 1-5 | Convient à la plupart des modèles. Aucun traitement particulier nécessaire. |
| 5-15 | Fonctionne bien avec des noms et des descriptions clairs et distincts. |
| 15-30 | Envisagez de regrouper les outils apparentés ou d'utiliser une étape de sélection d'outils. |
| 30+ | Probablement trop. Utilisez une approche en deux étapes : d'abord sélectionner une catégorie, puis choisir un outil spécifique. |

**L'approche en deux étapes pour les grands ensembles d'outils :**

```
Étape 1 : le modèle choisit une catégorie
  "L'utilisateur veut gérer sa commande -> Catégorie : gestion des commandes"

Étape 2 : le modèle ne voit que les outils de cette catégorie
  [get_order, create_order, cancel_order, update_shipping]
  -> Le modèle choisit : get_order
```

Cela permet de garder chaque décision individuelle gérable.

### 2. des descriptions d'outils vagues

Si le modèle ne peut pas déterminer quand utiliser un outil, il l'utilisera incorrectement, ou pas du tout.

**Mauvais :**
```json
{
    "name": "lookup",
    "description": "Recherche des informations"
}
```

Des informations sur quoi ? Quand le modèle devrait-il appeler cet outil plutôt qu'un autre ? Sous quel format arrive la réponse ?

**Bon :**
```json
{
    "name": "lookup_employee",
    "description": "Recherche un employé par nom ou par ID d'employé. Renvoie son département, son rôle, son e-mail et son manager. À utiliser quand l'utilisateur pose une question sur une personne spécifique de l'entreprise."
}
```

### 3. des wrappers d'API superficiels (qui exposent des détails d'implémentation)

Une erreur courante consiste à envelopper un endpoint (point de terminaison) d'API brut en tant qu'outil, sans aucune abstraction. Un outil générique `api_request(method, url, headers, body)` force le modèle à connaître la structure de vos URL, les en-têtes d'authentification, et le format des requêtes - il se trompera fréquemment sur ces éléments.

Construisez plutôt des outils conçus sur mesure comme `get_customer_orders(customer_email, status_filter)`, qui masquent les détails HTTP derrière une interface propre. Votre code gère l'authentification, la construction des URL, et l'analyse des réponses. Le modèle se contente de dire ce qu'il veut.

### 4. des informations d'erreur manquantes

Quand un appel d'outil échoue et que le message d'erreur est inutile, l'agent se retrouve bloqué dans une boucle de nouvelles tentatives, ou abandonne complètement.

**Modes d'échec courants :**

| Type d'erreur | Mauvaise réponse | Bonne réponse |
|---|---|---|
| Non trouvé | `{"error": true}` | `{"error": "CUSTOMER_NOT_FOUND", "message": "Aucun client avec l'e-mail 'jn@example.com'. Vouliez-vous dire 'jane@example.com' ?"}` |
| Entrée invalide | `500 Internal Server Error` | `{"error": "INVALID_DATE_FORMAT", "message": "Date attendue au format AAAA-MM-JJ, reçu '03/15/2024'"}` |
| Limite de débit atteinte | `{"error": "fail"}` | `{"error": "RATE_LIMITED", "message": "Trop de requêtes. Réessayez dans 30 secondes."}` |
| Échec d'authentification | `null` | `{"error": "UNAUTHORIZED", "message": "Clé API expirée. Cet outil est temporairement indisponible."}` |

### 5. renvoyer trop de données

Les réponses d'outils volumineuses accaparent la context window et peuvent dérouter le modèle avec des détails non pertinents. Stratégies pour gérer cela :

- **Filtrer à la source** : ne récupérez de la base de données/API que ce dont vous avez besoin
- **Sélectionner les champs pertinents** : ne renvoyez pas chaque champ - choisissez ce dont l'agent a besoin
- **Paginer** : renvoyez un sous-ensemble avec la possibilité d'en obtenir davantage
- **Résumer** : pour les réponses textuelles volumineuses, résumez avant de renvoyer
- **Tronquer** : plafonnez les champs de texte longs (par ex., `message[:200]`)

### 6. ne pas gérer les timeouts des outils

Les API externes peuvent être lentes ou ne pas répondre. Configurez toujours des timeouts (délais d'expiration) sur les appels HTTP, et renvoyez des messages d'erreur descriptifs en cas d'échec, plutôt que de laisser l'agent bloqué indéfiniment.

---

## Liste de vérification pour la conception d'outils

Utilisez cette liste de vérification (checklist) quand vous concevez des outils pour votre agent :

```
[ ] Le nom décrit clairement l'action (verbe + nom)
[ ] La description explique quoi, quand, et ce qui est renvoyé
[ ] Chaque outil a une responsabilité unique
[ ] Les paramètres ont des noms descriptifs (pas d'abréviations)
[ ] Les paramètres incluent des descriptions avec des exemples de format
[ ] Les paramètres obligatoires et optionnels sont clairement indiqués
[ ] La sortie est concise - uniquement les champs dont l'agent a besoin
[ ] Les messages d'erreur sont descriptifs et exploitables
[ ] Des timeouts sont implémentés pour les appels externes
[ ] L'authentification est gérée dans le code de l'outil, pas par le modèle
[ ] La taille de la sortie est bornée (troncature, pagination, ou synthèse)
[ ] L'outil est testé indépendamment avant d'être connecté à l'agent
```

---

## ELI5 : que sont les outils ?

### Les outils, c'est comme les applications sur un téléphone

Pensez à votre smartphone. Le téléphone lui-même est intelligent - il a un processeur puissant, un bel écran, et un bon système d'exploitation. Mais sans applications, il ne peut pas faire grand-chose.

- Vous voulez vérifier la météo ? Il vous faut l'**application météo**.
- Vous voulez envoyer un message ? Il vous faut l'**application de messagerie**.
- Vous voulez vous rendre quelque part ? Il vous faut l'**application de cartes**.
- Vous voulez prendre une photo ? Il vous faut l'**application appareil photo**.

Chaque application donne au téléphone une nouvelle capacité. Le téléphone lui-même ne sait pas prévoir la météo - il sait seulement ouvrir l'application météo, demander les prévisions, et afficher le résultat.

**Un LLM, c'est comme le téléphone sans applications.** Il est intelligent, mais il ne peut pas consulter la météo réelle, envoyer de vrais messages, ou chercher un vrai itinéraire. Il peut seulement parler de ces choses en se basant sur ce qu'il a appris pendant son entraînement.

**Les outils, c'est comme installer des applications sur le téléphone.** Chaque outil donne à l'agent une nouvelle capacité :
- `get_weather` est l'application météo
- `send_email` est l'application e-mail
- `search_database` est comme une application de recherche spécialisée pour vos données
- `run_code` est comme une application de codage

Et tout comme avec les applications :
- Le téléphone (le modèle) décide quelle application ouvrir en fonction de ce que vous demandez
- L'application (l'outil) fait le travail réel
- Le téléphone vous affiche le résultat

Quand vous concevez des outils, vous construisez en réalité l'app store (magasin d'applications) de votre agent. De bonnes applications avec des noms clairs et des fonctionnalités utiles rendent le téléphone plus capable. De mauvaises applications avec des noms confus et un comportement bogué le rendent frustrant à utiliser.

---

## Les outils sur Google Cloud

Google Cloud propose plusieurs façons de donner des outils aux agents :

### Le function calling sur Vertex AI

Vertex AI prend en charge le function calling avec les modèles Gemini. Vous définissez vos outils comme des déclarations de fonctions, et le modèle génère des appels de fonction structurés quand c'est approprié.

> **Pour en savoir plus :** [Function calling sur Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/function-calling)

### Les outils de l'Agent Development Kit (ADK)

L'Agent Development Kit (ADK) fournit une manière structurée de définir et de gérer les outils de vos agents. ADK prend en charge :

- **Les outils de fonction** : envelopper n'importe quelle fonction Python en tant qu'outil d'agent
- **Les outils intégrés** : Google Search, Code Execution, et plus encore
- **Les outils agents** : utiliser un autre agent ADK comme outil
- **Les outils tiers** : intégration avec les outils LangChain, les outils CrewAI, et les serveurs MCP

ADK gère le format de définition des outils, l'exécution, et le passage des résultats, afin que vous puissiez vous concentrer sur la logique de l'outil plutôt que sur la plomberie technique.

> **Pour en savoir plus :** [Documentation des outils ADK](https://google.github.io/adk-docs/tools/)

### Les outils d'ancrage intégrés

L'ancrage par Google Search est disponible comme outil intégré pour les modèles Gemini sur Vertex AI. Une fois activé, le modèle peut rechercher sur le web pour ancrer ses réponses dans des informations à jour - aucun code d'outil personnalisé n'est requis.

---

## Pour tout assembler : un exemple pratique

Concevons l'ensemble d'outils d'un agent DevOps simple qui aide les ingénieurs à investiguer les problèmes de production.

### L'objectif de l'agent

Aider les ingénieurs d'astreinte à diagnostiquer et à répondre aux alertes de production.

### Conception de l'ensemble d'outils

| Outil | Objectif | Quand l'utiliser |
|---|---|---|
| `get_alert_details` | Récupère les détails d'une alerte spécifique | Quand un ingénieur pose une question sur une alerte |
| `query_metrics` | Récupère les métriques en série temporelle pour un service | Quand on investigue des problèmes de performance |
| `search_logs` | Recherche dans les logs applicatifs par service et plage horaire | Quand on cherche des erreurs ou des motifs |
| `list_recent_deployments` | Affiche les déploiements récents pour un service | Quand on vérifie si un déploiement a causé le problème |
| `get_runbook` | Récupère le runbook (procédure d'intervention) pour un type d'alerte spécifique | Quand l'agent doit suggérer des étapes de remédiation |
| `create_incident` | Crée un ticket d'incident | Quand le problème est confirmé et doit être suivi |

Remarquez comment chaque outil suit nos bonnes pratiques : des noms clairs en verbe+nom, des descriptions qui expliquent le quoi/quand/ce qui est renvoyé, une responsabilité unique, et des paramètres descriptifs avec des exemples.

### Exemple d'interaction

```
Ingénieur : "L'alerte ALT-9876 vient de se déclencher. Que se passe-t-il ?"

L'agent pense : "Je dois d'abord consulter cette alerte."
-> Appelle : get_alert_details(alert_id="ALT-9876")
<- Renvoie : {service: "auth-service", type: "high_latency", severity: "P1",
              description: "latence p99 > 2s pendant 5 minutes"}

L'agent pense : "Latence élevée sur auth-service. Vérifions les métriques et
les déploiements récents."
-> Appelle : query_metrics(service="auth-service", metric="latency_p99", time_range="1h")
-> Appelle : list_recent_deployments(service="auth-service", limit=5)
<- Les métriques montrent un pic de latence à partir de 14h05
<- Le déploiement #456 a eu lieu à 14h02

L'agent pense : "La latence a explosé 3 minutes après un déploiement. Vérifions
les logs d'erreurs."
-> Appelle : search_logs(service="auth-service", query="timeout OR slow", level="error", time_range="1h")
<- Les logs montrent des erreurs "Database connection pool exhausted" à partir de 14h05

L'agent pense : "Le déploiement a probablement modifié quelque chose qui épuise
le pool de connexions à la base de données. Récupérons le runbook pour m'orienter."
-> Appelle : get_runbook(alert_type="high_latency")
<- Runbook : Étape 1 : vérifier les déploiements récents. Étape 2 : comparer
les changements de configuration...

L'agent répond :
"Voici ce que j'ai trouvé pour ALT-9876 :

**Problème :** la latence p99 de auth-service a dépassé 2 secondes à partir de 14h05.

**Cause probable :** le déploiement #456 (à 14h02) semble avoir déclenché un
épuisement du pool de connexions à la base de données. Les logs d'erreurs montrent
des erreurs « Database connection pool exhausted » à partir de 3 minutes après
le déploiement.

**Actions recommandées :**
1. Revoir les changements du déploiement #456, en particulier les changements
   de configuration de base de données
2. Envisager un rollback du déploiement #456 si la latence affecte les utilisateurs
3. Vérifier le paramètre de taille du pool de connexions - il pourrait avoir
   besoin d'être augmenté

Souhaitez-vous que je crée un ticket d'incident pour cela ?"
```

Cet exemple montre comment des outils bien conçus, avec des noms clairs, de bonnes descriptions, et des sorties concises, permettent à l'agent d'investiguer efficacement un problème à travers plusieurs étapes.

---

## Points clés à retenir

1. **Les outils comblent le fossé entre penser et agir.** Sans outils, un agent n'est qu'un chatbot. Avec des outils, il peut interagir avec des systèmes et des données réels.

2. **Le modèle n'exécute jamais directement les outils.** Il propose des appels d'outils. Votre code les exécute. Vous gardez toujours le contrôle de ce qui se passe réellement.

3. **Trois types d'outils** : les outils de fonction (personnalisés), les outils intégrés (fournis par la plateforme), et les outils agents (utiliser un autre agent comme outil). Commencez par des outils de fonction pour votre cas d'usage spécifique.

4. **La conception d'outils, c'est du design UX - pour le modèle.** Des noms clairs, des paramètres descriptifs, des sorties concises, et des messages d'erreur utiles font la différence entre un outil que le modèle utilise bien et un outil avec lequel il peine.

5. **Surveillez le nombre de vos outils.** Plus il y a d'outils, plus le contexte utilisé est important et plus les décisions du modèle sont difficiles. Gardez un ensemble d'outils centré sur ce dont l'agent a réellement besoin.

6. **La standardisation réduit la charge d'intégration.** Des protocoles comme MCP visent à résoudre le problème N x M en créant une interface commune entre les applications IA et les outils.

---

## Et maintenant ?

Maintenant que nous comprenons le cerveau (le LLM) et les mains (les outils), la prochaine leçon les réunit avec la couche d'orchestration - la boucle de contrôle qui gère comment un agent pense, agit, observe, et recommence jusqu'à ce que le travail soit fait.

[Suivant : Leçon 4 - Design patterns agentiques -->](../04-agentic-design-patterns/README.md)
