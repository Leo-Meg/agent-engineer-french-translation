# Leçon 16 : MCP en détail - connecter les agents au monde

## Introduction

Dans la [leçon 14](../14-agent-protocols-mcp-and-a2a/README.md), nous avons présenté MCP (Model Context Protocol) et A2A à haut niveau. Cette leçon approfondit spécifiquement MCP - comment il fonctionne réellement en coulisses, quand il apporte une vraie valeur, quand des approches plus simples sont préférables, et comment penser la sécurité.

Nous abordons aussi l'une des questions les plus débattues dans la communauté de l'ingénierie IA : quand devriez-vous utiliser des serveurs MCP plutôt que de simplement laisser votre agent utiliser des outils CLI directement ?

### ELI5 : pensez à MCP comme à une multiprise avec des dispositifs de sécurité

Votre ordinateur portable peut se brancher directement sur une prise murale. Cela fonctionne très bien à la maison. Mais dans un bureau avec 50 appareils, vous voulez une multiprise avec protection contre les surtensions, des interrupteurs individuels, et un disjoncteur. MCP est cette multiprise - il ajoute une couche de gestion entre l'agent et les outils qu'il utilise. Que vous ayez besoin de cette couche dépend du nombre d'outils que vous avez, de qui les utilise, et du niveau de contrôle dont vous avez besoin.

> **Point clé à retenir :** MCP est un protocole puissant pour connecter les agents à des outils et données externes, mais il comporte de réels compromis en termes de coût et de complexité. Comprendre quand MCP apporte de la valeur, par rapport à quand des approches plus simples fonctionnent mieux, est une compétence essentielle pour les constructeurs d'agents.

---

## L'architecture MCP - comment ça marche réellement

MCP suit une architecture client-serveur avec trois rôles :

### Les trois rôles

```
+------------------+     +------------------+     +------------------+
|                  |     |                  |     |                  |
|   Hote MCP       |     |   Client MCP     |     |   Serveur MCP    |
|   (Votre app)    |---->|   (gestionnaire  |---->|   (fournisseur   |
|                  |     |    de protocole) |     |    d'outils)     |
|                  |     |                  |     |                  |
+------------------+     +------------------+     +------------------+
```

- **L'hôte** - l'application où l'agent s'exécute (Claude Desktop, un IDE, votre application personnalisée). Il crée et gère les clients MCP.
- **Le client** - gère la communication du protocole. Maintient une connexion 1:1 avec un seul serveur MCP. Gère la négociation de capacités et le routage des messages.
- **Le serveur** - expose des outils, des ressources, et des prompts au client. Chaque serveur enveloppe généralement un service spécifique (une base de données, une API, un système de fichiers).

### La communication

Tous les messages utilisent JSON-RPC 2.0. Le protocole prend en charge deux mécanismes de transport :

| Transport | Cas d'usage | Comment ça marche |
|-----------|----------|-------------|
| **stdio** | Outils locaux | Le serveur s'exécute comme un sous-processus, communique via stdin/stdout. Simple, rapide, aucun surcoût réseau. |
| **Streamable HTTP** | Outils distants | Utilise un seul endpoint HTTP avec communication bidirectionnelle. Prend en charge le déploiement serverless (Lambda, Cloud Functions). |

Note : le transport SSE (Server-Sent Events) original a été déprécié dans la révision de spécification de mars 2025. SSE était unidirectionnel et nécessitait deux endpoints. Streamable HTTP l'a remplacé par une conception bidirectionnelle à endpoint unique.

### Les primitives MCP

Les serveurs MCP peuvent exposer trois types de capacités :

| Primitive | Ce que c'est | Qui la contrôle | Exemple |
|-----------|-----------|----------------|---------|
| **Tools (outils)** | Des fonctions que l'agent peut appeler | Le modèle décide quand les utiliser | `query_database`, `send_email`, `create_file` |
| **Resources (ressources)** | Des données que l'agent peut lire | L'application ou l'utilisateur les sélectionne | Schémas de base de données, contenu de fichiers, documentation d'API |
| **Prompts** | Des modèles de prompt réutilisables | L'utilisateur les invoque | « Résume cette base de code », « Relis cette PR » |

En pratique, les Tools sont de loin la primitive la plus largement utilisée. Fin 2025, 99% des clients MCP prennent en charge les Tools, tandis que les Resources et les Prompts ont un taux d'adoption d'environ 30 à 35%.

---

## Le débat MCP contre CLI

C'est l'un des sujets les plus activement débattus dans l'ingénierie IA en ce moment. La question centrale : si un agent IA peut exécuter des commandes shell, pourquoi a-t-il besoin de MCP ?

### L'argument en faveur du CLI

De nombreux serveurs MCP sont de simples wrappers autour d'outils qui ont déjà d'excellents CLI. Le serveur MCP GitHub réimplémente des fonctionnalités disponibles via `gh`. Le serveur MCP Docker enveloppe les commandes `docker`. Le serveur MCP Kubernetes enveloppe `kubectl`.

Les LLM savent déjà utiliser ces CLI. Ils ont été entraînés sur des millions de pages de manuel, de réponses Stack Overflow, et de dépôts GitHub. Quand un agent utilise `gh pr list`, il utilise une connaissance qu'il possède déjà. Quand il utilise un serveur MCP, il doit d'abord charger le schéma de l'outil dans sa context window.

Les chiffres sont frappants :

| Métrique | Approche CLI | Approche MCP |
|--------|-------------|-------------|
| **Coût en tokens (requête simple)** | ~1 400 tokens | ~44 000 tokens (32 fois plus) |
| **Coût d'initialisation** | Proche de zéro | Peut atteindre 50 000+ tokens pour le chargement de schéma |
| **Fiabilité (benchmark)** | 100% | 72% |
| **Configuration requise** | Aucune (les outils sont déjà installés) | Installer et configurer le serveur MCP |

La différence de coût en tokens vient du fait que MCP doit charger des schémas d'outils complets (noms, descriptions, types de paramètres, types de retour) dans la context window. Un serveur MCP de base de données avec 106 outils a consommé 54 600 tokens juste pour s'initialiser - avant même que le moindre travail réel ne commence.

### L'argument en faveur de MCP

Les propriétés qui rendent MCP coûteux sont les mêmes propriétés qui le rendent gouvernable :

**La sécurité et l'authentification.** Les outils CLI s'exécutent avec les permissions ambiantes de l'utilisateur. Si l'agent peut exécuter `rm -rf /`, il le fera s'il le décide. MCP fournit une frontière de permission. La spécification impose OAuth 2.1 avec PKCE pour les serveurs basés sur HTTP, vous donnant une authentification standardisée, une rotation de tokens, et une révocation.

**Les environnements multi-utilisateurs.** Quand un agent agit en votre nom, la sécurité ambiante du CLI convient. Quand un agent agit pour le compte d'autres personnes - lisant les dépôts de clients, écrivant dans leur Jira, envoyant des messages sur leur Slack - vous avez besoin d'une authentification par utilisateur, de permissions limitées, et de pistes d'audit. MCP fournit un cadre pour cela.

**La découverte d'outils.** Les serveurs MCP annoncent leurs capacités via des schémas. Un agent peut découvrir quels outils sont disponibles à l'exécution sans en être informé à l'avance. Cela compte quand les outils changent ou quand différents utilisateurs ont accès à différents outils.

**Les entrées/sorties structurées.** Les outils MCP ont des entrées et des sorties typées. La sortie CLI est du texte non structuré que l'agent doit analyser. Pour des outils simples, cela convient, mais pour des API complexes avec des réponses JSON imbriquées, une sortie structurée est plus fiable.

### Quand utiliser quoi

| Situation | Approche recommandée | Pourquoi |
|-----------|---------------------|-----|
| Développeur travaillant en local | CLI | Aucune configuration, l'agent connaît déjà les outils, l'option la moins chère |
| Outils bien connus (git, docker, kubectl, jq) | CLI | Le LLM a de solides données d'entraînement, une analyse fiable |
| Agent mono-utilisateur | CLI | Les permissions ambiantes sont acceptables |
| Multi-utilisateurs / multi-tenant | MCP | Besoin d'authentification par utilisateur et de permissions limitées |
| Entreprise avec des exigences d'audit | MCP | Besoin de journalisation structurée et de contrôle d'accès |
| Ensemble d'outils étroit à haute fréquence | MCP | Le coût du schéma s'amortit sur de nombreux appels |
| Large surface d'outils, usage occasionnel | CLI | Éviter de payer le coût du schéma pour des outils rarement utilisés |
| API interne personnalisée sans CLI | MCP | Aucun CLI existant à exploiter |
| Outils qui changent fréquemment | MCP | La découverte dynamique gère les changements automatiquement |

### La réponse pratique

La plupart des systèmes en production utilisent les deux. Claude Code, par exemple, a un outil Bash pour l'accès CLI direct et prend aussi en charge les serveurs MCP. La décision se prend par intégration, pas à l'échelle du système entier.

Un choix par défaut raisonnable : **commencez avec le CLI. Ajoutez MCP quand vous rencontrez une limitation spécifique que MCP résout** - généralement l'authentification, le multi-tenant, ou la découverte d'outils structurée.

### mcp2cli : combler le fossé

Un outil intéressant appelé [mcp2cli](https://github.com/knowsuchagency/mcp2cli) convertit n'importe quel serveur MCP en CLI à l'exécution. Plutôt que de charger tous les schémas d'outils à l'avance, l'agent interroge `--list` et `--help` uniquement quand c'est nécessaire. Cela a montré une réduction de 96 à 99% des tokens dans des benchmarks, tout en conservant l'API structurée de MCP en dessous.

---

## La sécurité MCP - ce qui peut mal tourner

MCP introduit une nouvelle surface d'attaque. La fondation OWASP a publié une liste des risques de sécurité [MCP Top 10](https://owasp.org/www-project-mcp-top-10/). Voici ceux qui comptent le plus pour les constructeurs d'agents :

### 1. L'empoisonnement d'outils (tool poisoning)

Un serveur MCP malveillant ou compromis peut renvoyer des résultats manipulés. Si votre agent fait confiance à la sortie d'un outil sans vérification, il peut être amené à entreprendre des actions nuisibles.

**Atténuation :** validez les sorties d'outils. Utilisez plusieurs sources pour les décisions critiques. Implémentez un filtrage de sortie.

### 2. L'ombrage d'outils (tool shadowing)

Un outil malveillant imite un outil légitime. Si un agent a accès à deux outils avec des noms similaires - disons `query_database` d'un serveur de confiance et `query_db` d'un serveur non fiable - il pourrait utiliser le mauvais.

**Atténuation :** contrôlez à quels serveurs MCP votre agent peut se connecter. Relisez les noms et descriptions d'outils. Utilisez des listes blanches pour l'accès aux outils.

### 3. Les permissions excessives

MCP n'a pas de limitation de périmètre native. Un serveur MCP de base de données pourrait exposer à la fois des opérations de lecture et d'écriture. Si votre agent n'a besoin que de lire, il peut quand même écrire.

**Atténuation :** construisez ou utilisez des serveurs MCP qui n'exposent que les opérations dont vous avez besoin. Implémentez des contrôles d'accès côté serveur. Utilisez une passerelle API (comme Apigee) devant les serveurs MCP pour une gestion fine des permissions.

### 4. La saturation de la context window

Trop d'outils MCP dégradent la performance de l'agent. Chaque définition d'outil consomme des tokens de contexte. Un serveur avec plus de 100 outils peut épuiser une portion significative de la context window avant même que le moindre travail réel ne commence.

**Atténuation :** gardez le nombre d'outils par serveur raisonnable (moins de 20). Utilisez plusieurs serveurs spécialisés plutôt qu'un seul gros serveur. Envisagez un chargement paresseux (lazy loading) des schémas d'outils.

### 5. L'exposition de secrets

L'analyse de 5 200 serveurs MCP open source a révélé que plus de la moitié s'appuient sur des clés d'API statiques à longue durée de vie. Seulement environ 8,5% utilisent une authentification moderne comme OAuth.

**Atténuation :** utilisez des identifiants à durée de vie courte et limités en périmètre. Stockez les secrets dans un gestionnaire de secrets (comme Google Cloud Secret Manager), pas dans des variables d'environnement ou des fichiers de configuration. Faites tourner les identifiants régulièrement.

### Checklist de sécurité pour les déploiements MCP

- [ ] Auditer à quels serveurs MCP votre agent se connecte
- [ ] Relire les schémas d'outils à la recherche de permissions trop larges
- [ ] Utiliser OAuth 2.1 pour les serveurs MCP distants
- [ ] Stocker les secrets dans un gestionnaire de secrets, pas dans des variables d'environnement
- [ ] Valider et assainir les sorties d'outils avant d'agir sur leur base
- [ ] Limiter le nombre d'outils par serveur
- [ ] Journaliser toutes les invocations d'outils pour les pistes d'audit
- [ ] Tester avec des entrées adverses (empoisonnement d'outils, injection de prompt via les résultats d'outils)
- [ ] Utiliser une passerelle API pour les déploiements MCP d'entreprise
- [ ] Faire tourner les serveurs MCP dans des environnements en sandbox quand c'est possible

---

## MCP sur Google Cloud

Google Cloud fournit plusieurs points d'intégration pour MCP :

### ADK et MCP

L'Agent Development Kit (ADK) de Google dispose d'une prise en charge intégrée des outils MCP. Vous pouvez vous connecter à n'importe quel serveur MCP et utiliser ses outils au sein de votre agent ADK.

Pour les détails sur la configuration des outils MCP dans ADK, consultez la [documentation des outils MCP d'ADK](https://google.github.io/adk-docs/tools/mcp-tools/).

### Apigee comme passerelle MCP

Pour les déploiements d'entreprise, [Apigee](https://cloud.google.com/apigee) peut servir de passerelle API et d'agent pour MCP. Cela ajoute :

- La limitation de débit et la gestion de quotas
- Des politiques d'authentification et d'autorisation
- Des analyses et du monitoring
- Un registre d'outils et une découverte
- La gestion du trafic à travers plusieurs serveurs MCP

C'est particulièrement utile quand vous avez de nombreuses équipes déployant des serveurs MCP et avez besoin d'une gouvernance centralisée.

### Model Armor

[Model Armor](https://cloud.google.com/security/products/model-armor) peut filtrer et valider les entrées et les sorties circulant à travers les appels d'outils MCP, ajoutant une protection contre l'injection de prompt et l'exfiltration de données via les interactions d'outils.

---

## Construire un serveur MCP - décisions clés

Si vous décidez de construire un serveur MCP pour votre service, voici les décisions clés :

### Le choix du transport

| Question | stdio | Streamable HTTP |
|----------|-------|----------------|
| Le serveur est-il local à l'agent ? | Oui | L'un ou l'autre |
| Avez-vous besoin d'un accès distant ? | Non | Oui |
| Avez-vous besoin d'un déploiement serverless ? | Non | Oui |
| La latence est-elle critique ? | Oui | Moins |

### La granularité des outils

Préférez des outils à grain fin plutôt qu'à gros grain :

- Bon : `get_user_by_id`, `list_users`, `create_user`, `update_user_email`
- Mauvais : `manage_users` (un seul outil qui fait tout selon un paramètre de mode)

Des outils à grain fin donnent au LLM des choix plus clairs et produisent de meilleurs résultats. Mais gardez le nombre total gérable - 5 à 20 outils par serveur est une bonne fourchette.

### Le nommage et les descriptions

Les noms et descriptions d'outils sont la façon principale dont le LLM décide quel outil utiliser. Investissez du temps pour les rendre clairs :

- Le nom devrait décrire l'action : `search_documents_by_topic` et non `search`
- La description devrait expliquer quand l'utiliser, ce qu'il renvoie, et toute contrainte importante
- Les descriptions de paramètres devraient inclure les types, les plages valides, et des exemples
- Les messages d'erreur devraient aider le LLM à récupérer : « Utilisateur non trouvé. Essayez de rechercher par e-mail plutôt que par ID. »

### La conception de la sortie

Gardez les sorties d'outils concises. La sortie entre dans la context window de l'agent, et les grandes réponses mangent le budget.

- Ne renvoyez que ce dont l'agent a besoin pour prendre sa prochaine décision
- Paginez les grands ensembles de résultats
- Résumez plutôt que de déverser des données brutes
- Utilisez des formats structurés (JSON) pour une sortie analysable par machine

---

## Pour tout assembler - un arbre de décision

Quand vous décidez comment connecter votre agent à un service externe :

```
Avez-vous besoin de vous connecter a un service externe ?
|
+-- Existe-t-il un CLI bien connu pour cela ? (git, docker, aws,
|   gcloud, kubectl)
|   |
|   +-- Oui : l'agent a-t-il besoin d'une authentification
|   |   multi-utilisateur ou de pistes d'audit ?
|   |   |
|   |   +-- Non : utilisez le CLI directement
|   |   +-- Oui : utilisez MCP avec OAuth
|   |
|   +-- Non : continuez ci-dessous
|
+-- Existe-t-il un serveur MCP existant pour cela ?
|   |
|   +-- Oui : est-il activement maintenu et provient-il d'une
|   |   source de confiance ?
|   |   |
|   |   +-- Oui : utilisez le serveur MCP
|   |   +-- Non : envisagez de construire le votre ou d'utiliser
|   |       le CLI/l'API directement
|   |
|   +-- Non : continuez ci-dessous
|
+-- Le service a-t-il une API REST ?
    |
    +-- Oui : construisez un serveur MCP ou utilisez les outils
    |   OpenAPI d'ADK
    +-- Non : construisez un outil de fonction personnalise ou un
        serveur MCP
```

---

## Points clés à retenir

- MCP fournit une intégration d'outils structurée avec des schémas, de l'authentification, et de la découverte - mais à un coût en tokens
- Les outils CLI sont moins chers et souvent plus fiables pour les outils de développement bien connus
- La décision se prend par intégration, pas à l'échelle du système entier - la plupart des agents en production utilisent à la fois MCP et le CLI
- Commencez avec le CLI par défaut ; ajoutez MCP quand vous avez besoin d'authentification, de multi-tenant, ou de découverte d'outils
- La sécurité MCP nécessite une attention active - auditez les serveurs, limitez les permissions, validez les sorties
- Sur Google Cloud, ADK prend en charge MCP nativement, et Apigee peut servir de passerelle MCP d'entreprise
- Gardez les serveurs MCP ciblés : 5 à 20 outils bien décrits par serveur, des sorties concises, des messages d'erreur clairs

---

## Pour aller plus loin

- [Spécification MCP](https://modelcontextprotocol.io/)
- [Outils MCP d'ADK](https://google.github.io/adk-docs/tools/mcp-tools/)
- [OWASP MCP Top 10](https://owasp.org/www-project-mcp-top-10/)
- [Agentic AI Foundation (AAIF)](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)
- [mcp2cli - relier MCP au CLI](https://github.com/knowsuchagency/mcp2cli)
- [Google Cloud Apigee](https://cloud.google.com/apigee)

---

[Leçon précédente : AGENTS.md](../15-agents-md/README.md) | [Leçon suivante : Agent skills ->](../17-agent-skills/README.md)
