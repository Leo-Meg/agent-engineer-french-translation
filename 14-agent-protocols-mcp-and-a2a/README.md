# Leçon 14 : protocoles d'agents - MCP et A2A

## Introduction

Vous savez construire un agent. Vous savez lui donner des outils. Vous pouvez même construire une équipe d'agents qui travaillent ensemble. Mais que se passe-t-il quand votre agent doit utiliser un outil construit par quelqu'un d'autre ? Ou quand votre agent doit collaborer avec un agent construit par une équipe différente, dans une entreprise différente, utilisant un framework différent ?

Sans standards, vous finissez par écrire du code d'intégration personnalisé pour chaque combinaison. Cela ne passe pas à l'échelle.

Cette leçon couvre deux protocoles ouverts qui résolvent ce problème : le **Model Context Protocol (MCP)** pour connecter les agents à des outils et des données, et l'**Agent-to-Agent Protocol (A2A)** pour permettre aux agents de collaborer à travers les organisations et les fournisseurs. Ensemble, ils forment la couche de communication de l'écosystème d'agents moderne.

---

## Pourquoi les protocoles comptent

### Le problème d'intégration N x M

Imaginez que vous ayez 5 frameworks d'agents et 10 outils. Sans protocole standard, chaque framework a besoin d'un connecteur personnalisé pour chaque outil. Cela fait 5 x 10 = 50 intégrations personnalisées.

Ajoutez maintenant 5 outils supplémentaires. Vous avez besoin de 25 intégrations de plus. Ajoutez un autre framework et vous en avez besoin de 15 de plus. Le coût grandit de façon multiplicative.

```
Sans protocole standard :

  Agent A ---personnalise---> Outil 1
  Agent A ---personnalise---> Outil 2
  Agent A ---personnalise---> Outil 3
  Agent B ---personnalise---> Outil 1
  Agent B ---personnalise---> Outil 2
  Agent B ---personnalise---> Outil 3
  ...
  (N agents x M outils = N*M integrations)


Avec un protocole standard :

  Agent A ---\                   /--> Outil 1
  Agent B -----> [Protocole] ---+--> Outil 2
  Agent C ---/                   \--> Outil 3
  (N + M integrations)
```

C'est le même problème que l'USB a résolu pour le matériel informatique. Avant l'USB, chaque appareil avait son connecteur propriétaire. Les imprimantes avaient besoin de ports parallèles. Les claviers avaient besoin de PS/2. Les appareils photo avaient besoin de câbles série. L'USB a donné à tout le monde une interface commune, et l'écosystème a explosé.

Les protocoles font la même chose pour les agents.

### Deux types de communication

Les agents doivent communiquer de deux façons fondamentalement différentes :

1. **Agent vers outil :** « Appelle cette fonction avec ces paramètres et donne-moi le résultat. » C'est structuré, spécifique, et synchrone. MCP gère cela.

2. **Agent vers agent :** « J'ai besoin d'aide pour cet objectif. Détermine comment l'accomplir et informe-moi quand c'est fait. » C'est ouvert, orienté objectif, et potentiellement asynchrone. A2A gère cela.

Comprendre cette distinction est essentiel pour comprendre pourquoi nous avons besoin de deux protocoles, pas d'un seul.

---

## Model Context Protocol (MCP)

### Qu'est-ce que MCP

MCP est un standard ouvert pour connecter des modèles de langage et des agents à des outils et sources de données externes. Créé à l'origine par Anthropic et maintenant largement adopté à travers l'industrie, MCP fournit une interface universelle entre les applications IA et les services auxquels elles ont besoin d'accéder.

Voyez MCP comme un **adaptateur universel** pour les outils IA. Tout comme un port USB vous permet de brancher n'importe quel périphérique USB sur n'importe quel ordinateur, MCP permet à n'importe quel agent d'utiliser n'importe quel outil compatible MCP sans code d'intégration personnalisé.

### L'architecture : hôte, client, serveur

MCP utilise une architecture en trois parties :

```
+------------------+
|      Hote        |    (Votre application IA - IDE, chatbot, agent)
|  +------------+  |
|  |   Client   |  |    (Client MCP - gere les connexions aux serveurs)
|  +-----+------+  |
+---------|---------+
          |
     Protocole MCP
     (JSON-RPC)
          |
+---------|---------+
|     Serveur       |    (Serveur MCP - enveloppe un outil ou une source
|  +------------+   |     de donnees)
|  | Outil/     |   |
|  | Donnees    |   |    (La capacite reelle - base de donnees, API,
|  +------------+   |     systeme de fichiers)
+-------------------+
```

**L'hôte :** l'application avec laquelle l'utilisateur interagit. Cela pourrait être un IDE comme VS Code, une interface de chat, ou votre application d'agent. L'hôte contient un ou plusieurs clients MCP.

**Le client :** le client MCP vit à l'intérieur de l'hôte et gère les connexions vers les serveurs MCP. Il gère la négociation de protocole, le routage des messages, et le cycle de vie des connexions. Un seul client peut se connecter à plusieurs serveurs.

**Le serveur :** un serveur MCP enveloppe un outil ou une source de données spécifique et l'expose via le protocole MCP. Il existe des serveurs pour les bases de données, les systèmes de fichiers, les API, les produits SaaS, et plus encore. N'importe qui peut construire un serveur MCP.

### Primitives clés

MCP définit trois primitives centrales que les serveurs peuvent exposer :

| Primitive | Ce que c'est | Direction | Exemple |
|---|---|---|---|
| **Tools (outils)** | Des fonctions que le modèle peut appeler | Le modèle invoque, le serveur exécute | `search_database(query)`, `send_email(to, body)` |
| **Resources (ressources)** | Des données que le modèle peut lire | Le modèle demande, le serveur fournit | Contenu de fichier, enregistrements de base de données, réponses d'API |
| **Prompts** | Des interactions modèles | Le serveur fournit, l'utilisateur sélectionne | « Résume ce document », « Débogue cette erreur » |

Les **tools** sont la primitive la plus couramment utilisée. Elles fonctionnent tout comme les outils de fonction que nous avons couverts dans la leçon 3, mais avec une interface standardisée qui fonctionne à travers n'importe quel agent compatible MCP.

Les **resources** fournissent du contexte au modèle sans nécessiter d'appel de fonction. Voyez-les comme des sources de données en lecture seule que le modèle peut référencer.

Les **prompts** sont des modèles d'interaction préconstruits qu'un serveur peut proposer. Ils aident les utilisateurs à découvrir ce que le serveur peut faire.

### L'analogie de l'adaptateur universel

Sans MCP, connecter un agent à une nouvelle source de données ressemble à ceci :

1. Lire la documentation d'API de la source de données
2. Écrire du code d'authentification
3. Écrire la gestion des requêtes/réponses
4. Écrire la gestion des erreurs
5. Définir le schéma d'outil pour votre framework spécifique
6. Tester l'intégration

Avec MCP, ça ressemble à ceci :

1. Installer le serveur MCP pour cette source de données
2. Connecter votre agent à celui-ci
3. Terminé

Le serveur MCP gère l'authentification, le formatage des requêtes, la gestion des erreurs, et la définition de schéma. Votre agent a juste besoin de parler MCP.

### Les avantages de MCP

- **Écrire une fois, utiliser partout.** Un outil construit comme serveur MCP fonctionne avec n'importe quel agent compatible MCP - ADK, Claude, Cursor, ou tout autre hôte.

- **Découverte dynamique d'outils.** Les agents peuvent découvrir quels outils sont disponibles à l'exécution, plutôt que d'avoir tout codé en dur. Connectez-vous à un nouveau serveur MCP et votre agent gagne automatiquement de nouvelles capacités.

- **Effet de levier de l'écosystème.** Il existe des centaines de serveurs MCP construits par la communauté pour des services populaires. Besoin de vous connecter à GitHub ? Slack ? Une base de données PostgreSQL ? Il existe probablement déjà un serveur MCP pour cela.

- **Séparation des responsabilités.** Les constructeurs d'outils se concentrent sur leur outil. Les constructeurs d'agents se concentrent sur leur agent. Le protocole gère l'interface entre les deux.

### Utiliser des outils MCP dans ADK

ADK dispose d'une prise en charge intégrée de MCP. Vous pouvez vous connecter à n'importe quel serveur MCP et utiliser ses outils comme s'ils étaient des outils ADK natifs.

```python
from google.adk.agents import Agent
from google.adk.tools.mcp_tool import MCPToolset, SseServerParams

# Se connecter a un serveur MCP
mcp_tools = MCPToolset(
    connection_params=SseServerParams(
        url="http://localhost:3000/mcp",
    )
)

agent = Agent(
    name="mcp_agent",
    model="gemini-2.0-flash",
    instruction="Tu es un assistant utile ayant acces a des outils externes.",
    tools=[mcp_tools],
)
```

L'agent découvre les outils disponibles depuis le serveur MCP à l'exécution. Si le serveur expose un outil `search_database` et un outil `create_ticket`, votre agent peut utiliser les deux sans code supplémentaire.

> **Pour en savoir plus :** [Les outils MCP dans ADK](https://google.github.io/adk-docs/tools/mcp-tools/)

### Limitations et considérations de sécurité

MCP est puissant, mais il vient avec des compromis que vous devriez comprendre :

**L'ombrage d'outils (tool shadowing).** Si deux serveurs MCP exposent des outils avec des noms ou des descriptions similaires, le modèle pourrait être confus sur lequel appeler. Soyez délibéré sur les serveurs auxquels vous vous connectez et vérifiez les conflits de nommage.

**La saturation de la context window.** Chaque serveur MCP connecté ajoute des définitions d'outils à la context window. Connectez-vous à trop de serveurs et vous mangez l'espace disponible pour la conversation réelle. Chaque définition d'outil consomme généralement 100 à 500 tokens.

**Aucune limitation native de périmètre.** MCP n'a pas de contrôles de permission granulaires intégrés. Si votre agent se connecte à un serveur MCP de base de données, il peut potentiellement accéder à toutes les données que ce serveur expose. Vous devez gérer l'autorisation au niveau du serveur ou via des guardrails.

**Confiance et chaîne d'approvisionnement.** Les serveurs MCP construits par la communauté sont du code tiers que votre agent exécute. Traitez-les avec la même prudence que vous traiteriez n'importe quelle dépendance open source. Relisez le code, vérifiez le mainteneur, et exécutez-les dans des environnements en sandbox.

**La latence.** Chaque appel d'outil MCP implique une communication réseau avec le serveur MCP. Pour les applications sensibles au temps, tenez compte de ce surcoût.

| Considération | Risque | Atténuation |
|---|---|---|
| Ombrage d'outils | Le modèle appelle le mauvais outil | Auditer les noms d'outils, limiter les serveurs connectés |
| Saturation de contexte | Qualité de raisonnement réduite | Ne connecter que les serveurs nécessaires |
| Aucune limite de périmètre | Accès aux données trop large | Authentification côté serveur, guardrails |
| Chaîne d'approvisionnement | Serveurs malveillants ou buggés | Relecture de code, sandboxing |
| Latence | Réponses d'outils lentes | Serveurs locaux, mise en cache |

---

## Agent-to-Agent Protocol (A2A)

### Qu'est-ce que A2A ?

A2A est un protocole ouvert développé par Google pour permettre aux agents de découvrir d'autres agents, de communiquer avec eux, et de leur déléguer des tâches - même des agents construits par des équipes différentes utilisant des frameworks différents dans des organisations différentes.

Alors que MCP gère le problème « l'agent parle à l'outil », A2A gère le problème « l'agent parle à l'agent ».

### L'analogie du réseau professionnel

Pensez à la façon dont les professionnels collaborent dans le monde réel. Quand vous avez besoin d'un conseil juridique, vous ne devenez pas avocat. Vous trouvez un avocat qualifié, expliquez ce dont vous avez besoin, et il s'en occupe.

Comment trouvez-vous cet avocat ?

1. **Découverte :** vous le cherchez - peut-être via un annuaire, une recommandation, ou un réseau professionnel
2. **Vérification de capacité :** vous consultez son profil pour voir s'il traite votre type d'affaire
3. **Engagement :** vous décrivez votre situation et ce dont vous avez besoin
4. **Délégation :** il s'en va travailler dessus, en vous envoyant des mises à jour
5. **Livraison :** il revient avec le résultat

A2A fonctionne de la même manière pour les agents :

1. **Découverte :** votre agent trouve d'autres agents via des Agent Cards
2. **Vérification de capacité :** il lit la carte pour voir ce que l'agent peut faire
3. **Engagement :** il envoie une tâche avec une description de ce qui doit être fait
4. **Délégation :** l'agent distant travaille dessus, en envoyant des mises à jour de statut
5. **Livraison :** l'agent distant renvoie le résultat terminé

### Concepts clés

#### Les Agent Cards

Une Agent Card est comme une carte de visite pour un agent. C'est un document JSON standardisé qui décrit ce que l'agent peut faire, comment communiquer avec lui, et quelle authentification il requiert.

```json
{
  "name": "Travel Booking Agent",
  "description": "Reserve des vols et des hotels selon les besoins de voyage",
  "url": "https://travel-agent.example.com/a2a",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true
  },
  "skills": [
    {
      "id": "book_flight",
      "name": "Book Flight",
      "description": "Recherche et reserve des vols entre villes"
    },
    {
      "id": "book_hotel",
      "name": "Book Hotel",
      "description": "Trouve et reserve des chambres d'hotel"
    }
  ],
  "authentication": {
    "schemes": ["oauth2"]
  }
}
```

Les Agent Cards sont hébergées à une URL bien connue (généralement `/.well-known/agent.json`), ce qui rend la découverte simple. Votre agent peut vérifier un endpoint connu pour voir ce qu'un autre agent propose.

#### Les tâches (tasks)

Une tâche est l'unité de travail fondamentale dans A2A. Quand un agent veut qu'un autre agent fasse quelque chose, il crée une tâche :

- **La création de tâche :** l'agent appelant envoie un message décrivant ce qui doit être fait
- **Le cycle de vie de la tâche :** la tâche traverse des états - soumise, en cours, entrée requise, terminée, ou échouée
- **Les mises à jour de tâche :** l'agent qui travaille peut envoyer des mises à jour de progression afin que l'appelant sache ce qui se passe
- **L'achèvement de tâche :** l'agent qui travaille renvoie les résultats sous forme d'artifacts

#### Les artifacts

Les artifacts sont les sorties d'une tâche. Ils peuvent être du texte, des fichiers, des données structurées, ou tout autre contenu que l'agent qui travaille produit.

#### Les files d'événements

A2A prend en charge la communication en temps réel via les Server-Sent Events (SSE). Cela permet aux agents de diffuser des mises à jour de progression plutôt que d'attendre que la tâche entière soit terminée. C'est particulièrement important pour les tâches de longue durée où l'agent appelant (ou un humain) veut voir la progression intermédiaire.

### Comment A2A se compare aux appels d'API directs

Vous vous demandez peut-être : pourquoi ne pas simplement appeler directement l'API d'un autre agent ? Vous le pourriez, mais vous feriez face au même problème N x M dont nous avons discuté plus tôt. A2A vous donne :

- **Une découverte standardisée** - trouver des agents sans connaître leur API spécifique
- **Un cycle de vie de tâche commun** - chaque agent gère les tâches de la même manière
- **Le streaming par défaut** - des mises à jour en temps réel sans code WebSocket personnalisé
- **La compatibilité inter-frameworks** - votre agent ADK peut fonctionner avec un agent LangChain
- **Des standards d'authentification** - un modèle de sécurité cohérent entre les agents

### Quand utiliser A2A contre MCP

C'est l'une des distinctions les plus importantes à comprendre :

| Aspect | MCP | A2A |
|---|---|---|
| **Qui parle** | Agent à outil | Agent à agent |
| **Style de communication** | « Fais cette chose précise » | « Atteins cet objectif » |
| **Complexité de la requête** | Un seul appel de fonction | Une tâche ouverte |
| **Intelligence de l'autre côté** | Outil (aucun raisonnement) | Agent (a du raisonnement) |
| **Exemple** | « Interroge cette base de données » | « Recherche ce sujet et rédige un rapport » |
| **Analogie** | Utiliser une calculatrice | Engager un consultant |

**MCP est pour les outils.** Vous savez exactement quelle fonction vous voulez appeler et quels paramètres passer. L'outil s'exécute et renvoie un résultat. Il n'y a aucun raisonnement de l'autre côté.

**A2A est pour les agents.** Vous décrivez un objectif et laissez l'autre agent déterminer comment l'accomplir. L'autre agent a son propre raisonnement, ses propres outils, et sa propre approche.

**Un exemple pratique :** supposons que vous construisiez un agent de planification de voyage.

- Vous utiliseriez **MCP** pour vous connecter à une API de recherche de vols (un outil qui prend une ville de départ, une ville d'arrivée, et une date, et renvoie des vols)
- Vous utiliseriez **A2A** pour déléguer à un agent de réservation d'hôtel capable de comprendre des préférences comme « quelque part de calme près du lieu de la conférence » et de déterminer les meilleures options par lui-même

> **Pour en savoir plus :** [A2A dans ADK](https://google.github.io/adk-docs/a2a/) et la [spécification du protocole A2A](https://a2a-protocol.org/latest/)

---

## Comment MCP et A2A fonctionnent ensemble

MCP et A2A ne sont pas des standards concurrents. Ils opèrent à des couches différentes et se complètent.

```
+---------------------------------------------+
|              Votre agent                      |
|                                              |
|  "J'ai besoin de reserver un voyage a Tokyo" |
|                                              |
|  +-------------------+  +----------------+   |
|  | Client MCP         |  | Client A2A     |   |
|  | (parle aux outils) |  | (parle aux     |   |
|  |                    |  |  autres agents) |   |
|  +--------+----------+  +-------+--------+   |
+-----------|-----------------------|----------+
            |                       |
    +-------v--------+     +-------v--------+
    | Serveurs MCP    |     | Agents distants |
    |                 |     |                |
    | - API vols      |     | - Agent hotel  |
    | - API meteo     |     | - Agent budget |
    | - Calendrier    |     | - Agent avis   |
    +--+---------+----+     +---+--------+---+
       |         |              |        |
       v         v              v        v
    [Donnees  [Donnees      [Reservation [Analyse
     vols]     meteo]        hotel]      budget]
```

### L'architecture en couches

Pensez-y comme des couches :

1. **La couche outil (MCP) :** votre agent se connecte à des sources de données et des API spécifiques via des serveurs MCP. Cela lui donne accès à des capacités brutes - rechercher dans des bases de données, appeler des API, lire des fichiers.

2. **La couche agent (A2A) :** votre agent collabore avec d'autres agents qui ont leurs propres outils, raisonnement, et expertise. Cela lui donne accès à des capacités de plus haut niveau - des tâches nécessitant du jugement, de la planification, et une exécution multi-étapes.

3. **La couche d'orchestration (votre agent) :** votre agent décide quand utiliser un outil directement (MCP) et quand déléguer à un autre agent (A2A) selon la tâche en cours.

### Un scénario concret

Un utilisateur demande à votre agent de voyage : « Planifie un voyage de 3 jours à Tokyo le mois prochain avec un budget de 3000 $. »

Votre agent pourrait :

1. **Appel MCP :** vérifier le calendrier de l'utilisateur pour les dates disponibles (serveur MCP calendrier)
2. **Appel MCP :** obtenir les prix de vols actuels pour ces dates (serveur MCP API de vols)
3. **Délégation A2A :** demander à un agent de réservation d'hôtel de trouver un logement près de Shibuya à moins de 200 $/nuit
4. **Délégation A2A :** demander à un agent d'activités locales de suggérer un itinéraire de 3 jours
5. **Appel MCP :** vérifier les prévisions météo pour Tokyo pendant ces dates (serveur MCP météo)
6. **Raisonnement :** combiner tous les résultats, vérifier par rapport au budget, et présenter un plan

Remarquez le pattern : MCP pour la récupération de données spécifiques, A2A pour les tâches nécessitant l'expertise et le jugement d'un autre agent.

---

## ELI5 : comprendre MCP et A2A

### MCP, c'est comme un adaptateur électrique

Vous savez comment différents pays ont des prises électriques différentes ? Si vous voyagez des États-Unis vers le Royaume-Uni, le chargeur de votre ordinateur portable ne rentrera pas dans la prise. Vous avez besoin d'un adaptateur.

MCP est cet adaptateur pour les agents IA. Chaque outil avait l'habitude d'avoir sa propre prise propriétaire (intégration d'API personnalisée). MCP donne à tout le monde une prise universelle. Branchez n'importe quel outil sur MCP, et n'importe quel agent peut l'utiliser.

L'outil lui-même ne devient pas plus intelligent. Un adaptateur électrique ne rend pas votre ordinateur portable plus rapide. Mais il rend votre ordinateur portable utilisable dans des endroits où il ne pouvait pas fonctionner auparavant. Il en va de même pour MCP - il rend les outils accessibles à des agents qui ne pouvaient pas les atteindre avant.

### A2A, c'est comme un appel téléphonique entre collègues

Maintenant, imaginez que vous travaillez sur un gros projet dans une entreprise. Vous gérez l'ingénierie, mais vous avez besoin de supports marketing. Vous n'apprenez pas le marketing vous-même. Vous appelez votre collègue du service marketing.

Vous dites : « Nous lançons la nouvelle API mardi prochain. Peux-tu préparer un article de blog de lancement et un plan pour les réseaux sociaux ? »

Votre collègue répond : « Bien sûr, je vais rédiger quelque chose et t'envoyer des mises à jour au fur et à mesure. »

A2A, c'est cet appel téléphonique. Un agent (vous) appelle un autre agent (le marketing) avec un objectif. L'autre agent utilise ses propres compétences et outils pour l'accomplir. Il envoie des mises à jour en cours de route. Et il livre le travail terminé une fois fait.

Vous n'aviez pas besoin de savoir quels outils le marketing utilise. Vous n'aviez pas besoin de comprendre leur processus. Vous aviez juste besoin de décrire ce que vous vouliez et de leur faire confiance pour trouver comment faire.

### Pourquoi nous avons besoin des deux

Pour revenir à l'analogie des collègues :

- **MCP** est comme les outils sur votre bureau - votre clavier, votre écran, votre éditeur de code. Vous les utilisez directement.
- **A2A** est comme vos collègues - vous leur déléguez du travail et ils utilisent leurs propres outils.

Vous avez besoin des deux. Certaines choses, vous les faites vous-même avec vos outils. D'autres choses, vous demandez à un spécialiste de les gérer.

---

## Considérations de sécurité pour les deux protocoles

### La sécurité MCP

- **L'authentification du serveur :** vérifiez l'identité des serveurs MCP avant de vous connecter. Utilisez TLS pour toute communication.
- **Le moindre privilège :** ne vous connectez qu'aux serveurs MCP dont votre agent a réellement besoin. Chaque serveur supplémentaire augmente votre surface d'attaque.
- **La validation des entrées :** les serveurs MCP devraient valider tous les paramètres qu'ils reçoivent. Ne présumez pas que le modèle enverra toujours des entrées bien formées.
- **La journalisation d'audit :** journalisez tous les appels d'outils MCP pour le débogage et la revue de sécurité.

### La sécurité A2A

- **La vérification d'agent :** avant de déléguer des tâches, vérifiez l'identité de l'agent distant via son Agent Card et son schéma d'authentification.
- **La minimisation des données :** ne partagez que l'information dont l'agent distant a besoin pour accomplir la tâche. N'envoyez pas tout votre contexte.
- **La validation des résultats :** traitez les résultats provenant d'agents distants avec un scepticisme approprié. Vérifiez les sorties critiques avant d'agir sur leur base.
- **Le contrôle d'accès :** définissez quels agents peuvent accéder à quelles capacités de votre agent.

### La défense en profondeur

Les deux protocoles bénéficient d'une approche de sécurité en couches :

1. **La sécurité du transport :** TLS partout
2. **L'authentification :** vérifier les identités des deux côtés
3. **L'autorisation :** limiter ce que chaque connexion peut faire
4. **Le monitoring :** surveiller les patterns inhabituels
5. **Les guardrails :** valider les entrées et les sorties à chaque frontière

---

## L'état actuel de l'écosystème

### L'écosystème MCP

MCP a connu une adoption rapide depuis son introduction. L'écosystème inclut :

- **Des centaines de serveurs MCP** pour des services populaires (bases de données, plateformes cloud, outils SaaS, outils de développement)
- **Une prise en charge dans les principales plateformes IA**, incluant Claude, ADK, VS Code, et bien d'autres
- **Une communauté grandissante** de contributeurs construisant et maintenant des serveurs

### L'écosystème A2A

A2A est plus récent et l'écosystème est encore en développement :

- **Une prise en charge ADK** à la fois pour créer des agents compatibles A2A et se connecter à des agents A2A distants
- **Des implémentations de référence** démontrant des patterns courants
- **Un intérêt grandissant** de la part des organisations construisant des systèmes multi-agents

### À quoi s'attendre

Les deux protocoles évoluent activement. Attendez-vous à voir :

- Plus de serveurs MCP pour les outils et services d'entreprise
- Plus de frameworks d'agents adoptant la prise en charge d'A2A
- Un meilleur outillage pour découvrir, tester, et surveiller les connexions de protocole
- Une standardisation des patterns de sécurité et des bonnes pratiques

---

## Conseils pratiques

### Pour débuter avec MCP

1. **Commencez avec des serveurs officiels.** Utilisez des serveurs MCP bien maintenus provenant de sources fiables avant d'essayer ceux de la communauté.
2. **Testez d'abord en local.** Faites tourner les serveurs MCP en local pendant le développement avant de pointer vers des serveurs distants.
3. **Surveillez l'usage de tokens.** Chaque serveur MCP connecté ajoute des définitions d'outils à votre contexte. Suivez la quantité d'espace de contexte que vos outils consomment.
4. **Verrouillez la version de vos serveurs.** Les serveurs MCP sont des dépendances logicielles. Verrouillez les versions pour éviter les surprises.

### Pour débuter avec A2A

1. **Commencez avec des agents que vous contrôlez.** Construisez deux agents vous-même et entraînez-vous à la communication A2A avant de vous connecter à des agents externes.
2. **Définissez des contrats clairs.** Soyez précis sur les tâches que vous attendez qu'un agent distant gère et sur les sorties que vous attendez.
3. **Gérez les échecs avec élégance.** Les agents distants peuvent être lents, indisponibles, ou renvoyer des résultats inattendus. Construisez une logique de nouvelle tentative et de repli.
4. **Journalisez tout.** La communication multi-agents est difficile à déboguer. Une journalisation détaillée est essentielle.

---

## Points clés à retenir

1. **Les protocoles résolvent le problème d'intégration N x M.** Sans standards, chaque combinaison agent-outil et agent-agent nécessite du code personnalisé. MCP et A2A remplacent cela par des interfaces universelles.

2. **MCP connecte les agents aux outils.** C'est un adaptateur universel qui permet à n'importe quel agent d'utiliser n'importe quel outil compatible MCP. Pensez à l'USB pour l'IA.

3. **A2A connecte les agents aux agents.** Il permet aux agents de découvrir d'autres agents, de communiquer avec eux, et de leur déléguer des tâches à travers les organisations et les frameworks.

4. **MCP et A2A se complètent.** MCP opère au niveau de la couche outil (appels de fonction spécifiques). A2A opère au niveau de la couche agent (tâches orientées objectif). Utilisez les deux ensemble pour une flexibilité maximale.

5. **La sécurité nécessite de l'attention aux deux couches.** Vérifiez les identités, minimisez le partage de données, validez les résultats, et journalisez tout.

---

## Pour aller plus loin

- **Les outils MCP dans ADK :** [https://google.github.io/adk-docs/tools/mcp-tools/](https://google.github.io/adk-docs/tools/mcp-tools/)
- **A2A dans ADK :** [https://google.github.io/adk-docs/a2a/](https://google.github.io/adk-docs/a2a/)
- **Spécification du protocole A2A :** [https://a2a-protocol.org/latest/](https://a2a-protocol.org/latest/)
- **Spécification MCP :** [https://modelcontextprotocol.io](https://modelcontextprotocol.io)

---

## Et maintenant ?

Vous avez couvert les fondamentaux et les briques de base. Dans les leçons suivantes, nous approfondirons des sujets spécifiques essentiels au développement d'agents en conditions réelles.

[Suivant : Leçon 15 - AGENTS.md -->](../15-agents-md/README.md)
