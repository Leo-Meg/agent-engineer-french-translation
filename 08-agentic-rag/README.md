# Leçon 8 : RAG agentique - une récupération plus intelligente

## Introduction

Dans les leçons précédentes, vous avez appris comment les agents utilisent des outils pour interagir avec le monde. L'un des outils les plus importants qu'un agent puisse avoir, c'est la capacité de rechercher des choses - interroger une base de connaissances, une base de données, ou récupérer des documents. C'est le fondement de la génération augmentée par récupération, ou RAG (Retrieval-Augmented Generation).

Le RAG de base suit un pattern simple : prendre la question de l'utilisateur, rechercher des documents pertinents, les insérer dans le contexte du LLM, et générer une réponse. Cela fonctionne bien pour des recherches simples. Mais cela s'effondre quand la question est complexe, quand la première recherche ne renvoie pas les bons documents, ou quand la réponse nécessite de synthétiser de l'information provenant de plusieurs sources.

Le RAG agentique corrige cela en plaçant un agent aux commandes du processus de récupération. Plutôt qu'un pipeline rigide de récupération-puis-lecture, l'agent décide quand rechercher, quoi rechercher, si les résultats sont suffisamment bons, et quand réessayer avec une approche différente. L'agent traite la récupération comme un outil qu'il utilise stratégiquement, et non comme une étape fixe qu'il suit systématiquement.

## Rappel rapide : qu'est-ce que le RAG ?

RAG signifie Retrieval-Augmented Generation (génération augmentée par récupération). L'idée est simple : les LLM ont une date limite de connaissances et ne peuvent pas tout savoir. Donc, avant de répondre à une question, vous récupérez l'information pertinente depuis une source externe et vous l'incluez dans le prompt.

**Le pipeline RAG de base :**

```
Question de l'utilisateur
     |
     v
[Recuperateur] -- recherche --> [Entrepot de documents]
     |
     v
Documents recuperes
     |
     v
[LLM] -- genere une reponse a partir de la question + des documents
     |
     v
Reponse
```

**Pourquoi le RAG compte :**
- Les LLM ont une date limite de connaissances - le RAG fournit une information à jour
- Les LLM peuvent halluciner - le RAG ancre les réponses dans des documents réels
- Les LLM ne peuvent pas accéder à des données privées - le RAG les connecte à vos bases de données
- Le RAG vous permet de mettre à jour les connaissances sans réentraîner le modèle

**Une analogie simple :** le RAG, c'est comme un examen à livre ouvert. Plutôt que de vous fier uniquement à ce que vous avez mémorisé (les données d'entraînement du LLM), vous pouvez consulter vos notes (l'entrepôt de documents) avant de rédiger votre réponse.

## Les limites du RAG de base

Le RAG de base fonctionne bien pour des questions simples et directes où la réponse existe dans un seul document. Mais il peine dans plusieurs scénarios courants :

### Problème 1 : la requête ne correspond pas aux documents

L'utilisateur pose une question avec des mots différents de ceux utilisés dans les documents. Le récupérateur recherche « comment réparer une base de données lente » mais le document pertinent parle de « techniques d'optimisation de requêtes ». L'écart sémantique fait que le récupérateur passe à côté du meilleur document.

### Problème 2 : la réponse s'étend sur plusieurs documents

L'utilisateur demande « Quelles sont les politiques de notre entreprise sur le télétravail pour les employés internationaux ? » La réponse nécessite de combiner l'information provenant de la politique de télétravail, des directives d'emploi international, et de la documentation de conformité fiscale. Le RAG de base récupère une poignée de documents en une seule fois et espère que les bons y figurent.

### Problème 3 : les premiers résultats ne sont pas assez bons

Le RAG de base récupère des documents une seule fois et s'engage à les utiliser. Si les meilleurs résultats sont médiocres ou non pertinents, le LLM génère une réponse médiocre ou incorrecte. Il n'existe aucun mécanisme pour dire « ces résultats ne sont pas utiles, laissez-moi essayer une recherche différente ».

### Problème 4 : la question nécessite une décomposition

L'utilisateur pose une question complexe comme « Compare notre chiffre d'affaires du T3 en Amérique du Nord contre l'Europe et explique les principaux facteurs de la différence. » Cela nécessite plusieurs sous-requêtes : le chiffre d'affaires du T3 pour l'Amérique du Nord, celui pour l'Europe, et une analyse des facteurs contributifs. Le RAG de base tente d'y répondre avec une seule recherche.

### Problème 5 : aucune vérification

Le RAG de base n'a aucun mécanisme d'auto-vérification. Le LLM génère une réponse basée sur les documents récupérés, quels qu'ils soient, même si ces documents sont obsolètes, non pertinents, ou contradictoires. Il n'y a aucune étape qui demande « cette réponse a-t-elle réellement du sens au vu des preuves ? »

## Ce qui rend le RAG « agentique »

Le RAG agentique donne au LLM une capacité d'action sur le processus de récupération. Plutôt que de suivre un pipeline fixe, l'agent prend des décisions à chaque étape :

### L'agent décide quand rechercher

Toutes les questions ne nécessitent pas de récupération. Un système de RAG agentique peut reconnaître quand il connaît déjà la réponse (à partir de ses données d'entraînement ou du contexte de conversation actuel) et sauter complètement la récupération. Il peut aussi reconnaître quand il a définitivement besoin d'information externe et lancer une recherche.

### L'agent reformule les requêtes

Si la recherche initiale renvoie de mauvais résultats, l'agent n'abandonne pas. Il analyse pourquoi les résultats sont mauvais et essaie une requête différente. Peut-être que la question originale était trop large, alors il la resserre. Peut-être qu'il a utilisé la mauvaise terminologie, alors il reformule.

**Exemple :**
```
Requete originale : "reparer base de donnees lente"
    Resultats : articles generiques sur les bases de donnees
    L'agent pense : "Trop vague. Soyons plus precis."
Reformulee : "optimisation de requetes PostgreSQL pour operations JOIN lentes"
    Resultats : techniques d'optimisation specifiques
    L'agent pense : "Bien mieux. Ceux-ci sont pertinents."
```

### L'agent recoupe plusieurs sources

Plutôt que de se fier à une seule recherche, l'agent peut interroger plusieurs sources, comparer les résultats, et synthétiser une réponse plus complète. Il pourrait consulter la base de connaissances interne, puis vérifier avec la documentation publique, puis recouper avec des tickets de support récents.

### L'agent s'autocorrige

Après avoir généré une réponse, l'agent la vérifie par rapport aux preuves récupérées. La réponse découle-t-elle réellement des documents ? Y a-t-il des contradictions ? Manque-t-il une information critique ? Si la réponse ne tient pas, l'agent revient en arrière et récupère plus d'information.

## La boucle du RAG agentique

Le cœur du RAG agentique est une boucle, pas un pipeline. L'agent itère à travers la récupération, l'évaluation, et le raffinement, jusqu'à obtenir une réponse satisfaisante ou avoir épuisé ses options.

```
        Question de l'utilisateur
            |
            v
    [1. Planification de la requete]
     "Qu'est-ce que je dois trouver ?"
            |
            v
    [2. Recuperer]
     Rechercher dans la/les base(s) de connaissances
            |
            v
    [3. Evaluer les resultats]
     "Ces resultats sont-ils assez bons ?"
           / \
        Non   Oui
         |      \
         v       v
    [4. Raffiner] [5. Generer la reponse]
     Reformuler       |
     la requete       v
     et retourner  [6. Verifier la reponse]
     a l'etape 2   "Cette reponse tient-elle la route ?"
                    / \
                  Non   Oui
                  |      \
                  v       v
             Retourner  Renvoyer la
             a l'etape 1  reponse a
                          l'utilisateur
```

Parcourons chaque étape en détail.

### Étape 1 : planification de la requête

L'agent analyse la question de l'utilisateur et décide d'une stratégie de récupération. Pour des questions simples, cela peut signifier une seule recherche. Pour des questions complexes, l'agent décompose la question en sous-requêtes.

**Exemple :**

L'utilisateur demande : « Comment nos scores de satisfaction client ont-ils évolué après le lancement de notre nouveau chatbot de support ? »

L'agent planifie :
1. Trouver les scores de satisfaction client avant le lancement du chatbot
2. Trouver la date de lancement du chatbot
3. Trouver les scores de satisfaction client après le lancement du chatbot
4. Chercher toute analyse ou rapport reliant les deux

### Étape 2 : récupérer

L'agent exécute les recherches planifiées. Cela peut impliquer d'interroger une base de données vectorielle, d'appeler une API de recherche, de consulter des données structurées, ou toute combinaison de ces éléments.

### Étape 3 : évaluer les résultats

C'est là que le RAG agentique diverge du RAG de base. L'agent examine les documents récupérés et porte un jugement :

- **Pertinence :** ces documents répondent-ils réellement à ma question ?
- **Exhaustivité :** ai-je toute l'information dont j'ai besoin ?
- **Fraîcheur :** ces documents sont-ils à jour ?
- **Cohérence :** les sources concordent-elles entre elles ?

### Étape 4 : raffiner (si nécessaire)

Si les résultats ne sont pas assez bons, l'agent raffine son approche :

- **Reformuler la requête :** utiliser des mots-clés différents, être plus précis ou plus général
- **Essayer une source différente :** passer de la base de connaissances générale à une base spécialisée
- **Décomposer davantage :** diviser la question en sous-questions encore plus petites
- **Élargir la recherche :** chercher des concepts liés qui pourraient mener à la réponse

### Étape 5 : générer la réponse

Une fois que l'agent dispose de preuves suffisantes, il génère une réponse ancrée dans les documents récupérés. La réponse devrait citer ses sources afin que l'utilisateur puisse vérifier.

### Étape 6 : vérifier la réponse

L'agent effectue une vérification finale :

- La réponse contredit-elle l'un des documents récupérés ?
- Toutes les affirmations de la réponse sont-elles étayées par des preuves ?
- Y a-t-il des lacunes ou des atermoiements suggérant qu'une récupération supplémentaire est nécessaire ?

Si la vérification échoue, l'agent reboucle pour rassembler plus d'information.

## Capacités clés du RAG agentique

### La planification autonome des requêtes

L'agent décompose automatiquement les questions complexes en sous-requêtes, sans avoir besoin de modèles ou de règles prédéfinis. Il utilise sa compréhension de la question pour déterminer quelle information il lui faut.

**RAG de base :** envoie la question exacte de l'utilisateur au récupérateur.
**RAG agentique :** analyse la question, identifie les besoins en information, et planifie plusieurs recherches ciblées.

### La sélection adaptative de sources

L'agent peut choisir quelles sources de connaissances interroger en fonction de la question. Une question sur la politique de l'entreprise va vers la base de données de politiques. Une question sur un client va vers le CRM. Une question sur des événements récents va vers le web.

| Type de question | Sélection de source |
|---|---|
| Politique de l'entreprise | Base de données de politiques internes |
| Information client | Système CRM |
| Documentation technique | Wiki d'ingénierie |
| Événements récents | Recherche web |
| Spécifications produit | Catalogue produit |
| Données historiques | Entrepôt de données |

### L'expansion de requête sensible au contexte

L'agent utilise le contexte de conversation et l'information précédemment récupérée pour améliorer les requêtes suivantes. Si la première recherche a renvoyé de l'information sur le « Projet Alpha », l'agent pourrait ajouter « Projet Alpha » aux requêtes suivantes pour trouver des documents liés.

### Le raisonnement multi-sauts (multi-hop)

Certaines questions nécessitent des chaînes de récupération. La réponse à la première requête informe ce qu'il faut rechercher ensuite.

**Exemple :**
```
Question : "Qui manage l'equipe qui a construit notre moteur de
           recommandation ?"

Saut 1 : rechercher "equipe moteur de recommandation"
  -> Trouve : "Le moteur de recommandation a ete construit par
     l'equipe ML Platform"

Saut 2 : rechercher "manager de l'equipe ML Platform"
  -> Trouve : "L'equipe ML Platform est managee par Sarah Chen"

Reponse : "Sarah Chen manage l'equipe ML Platform, qui a construit
          le moteur de recommandation."
```

## Mécanismes d'autocorrection

L'un des aspects les plus précieux du RAG agentique est sa capacité à reconnaître ses erreurs et à s'en remettre.

### La re-requête

Quand l'agent détecte que ses résultats initiaux sont insuffisants, il formule de nouvelles requêtes basées sur ce qu'il a appris du premier tour. Ce n'est pas une nouvelle tentative aléatoire - c'est un raffinement éclairé.

```
Requete initiale : "processus de deploiement"
Resultats : trop de documents generiques sur divers processus de deploiement
Analyse de l'agent : "Je dois etre plus precis sur quel service"
Requete raffinee : "processus de deploiement pour le microservice de paiement"
Resultats : runbook specifique au deploiement du service de paiement
```

### Les outils de diagnostic

L'agent peut utiliser des outils de diagnostic pour évaluer la qualité de sa récupération :

- **Notation de pertinence :** noter chaque document récupéré selon sa pertinence par rapport à la question
- **Vérification de couverture :** s'assurer que tous les aspects de la question sont traités
- **Détection de contradictions :** signaler quand différentes sources sont en désaccord
- **Estimation de confiance :** évaluer le degré de confiance dans la réponse au vu des preuves

### Le recours à un humain

Quand l'agent ne peut pas trouver de réponse satisfaisante après plusieurs tentatives, il escalade vers un humain plutôt que de deviner. C'est un mécanisme de sécurité critique.

```
Agent : "J'ai recherche dans notre base de connaissances des
informations sur le projet de migration de donnees 2024, mais je
n'ai pas trouve de documentation suffisante. J'ai trouve des
references dans trois documents, mais aucun ne contenait le
calendrier precis que vous avez demande. Je recommande de verifier
directement aupres de l'equipe Data Engineering."
```

## Quand utiliser le RAG de base contre le RAG agentique

Tous les cas d'usage n'ont pas besoin de l'approche agentique complète. Voici un guide pour choisir :

### Utilisez le RAG de base quand :

- Les questions sont simples et directes (« Quelle est notre politique de retour ? »)
- La réponse existe généralement dans un seul document
- La latence est critique (le RAG agentique ajoute plusieurs appels au LLM)
- Le coût est une contrainte majeure
- La base de connaissances est petite et bien organisée
- Les exigences de précision sont modérées

### Utilisez le RAG agentique quand :

- Les questions sont complexes et ouvertes (« Analyse les tendances d'attrition de nos clients »)
- Les réponses nécessitent de synthétiser plusieurs documents
- L'utilisateur attend une profondeur digne d'une recherche
- La base de connaissances est vaste, diverse, ou mal organisée
- La précision est critique et les mauvaises réponses sont coûteuses
- Les questions nécessitent souvent des clarifications ou une décomposition

### Comparaison des coûts et de la latence

| Aspect | RAG de base | RAG agentique |
|---|---|---|
| Appels au LLM par requête | 1 | 3-10+ |
| Appels de récupération par requête | 1 | 2-5+ |
| Latence typique | 1-3 secondes | 5-30 secondes |
| Coût en tokens | Faible | 3 à 10 fois plus élevé |
| Qualité de réponse pour questions simples | Bonne | Similaire (excessif) |
| Qualité de réponse pour questions complexes | Faible à correcte | Bonne à excellente |

## ELI5 : le bibliothécaire contre l'assistant de recherche

Le RAG de base, c'est comme poser une question à un bibliothécaire. Vous vous approchez du comptoir et dites : « Avez-vous un livre sur les dinosaures ? » Le bibliothécaire consulte le catalogue, trouve un livre, et vous le tend. Si le livre ne répond pas à votre question précise, tant pis - c'est tout ce que vous obtenez.

Le RAG agentique, c'est comme engager un assistant de recherche. Vous dites : « J'ai besoin de comprendre pourquoi les dinosaures ont disparu. » L'assistant de recherche se rend à la bibliothèque, sort plusieurs livres, les parcourt, réalise que l'un est obsolète, le repose, trouve un article plus récent, recoupe deux théories différentes, vérifie les citations, et revient avec une synthèse bien sourcée. S'il ne trouve pas assez d'information à la bibliothèque, il consulte des bases de données en ligne. S'il trouve des informations contradictoires, il note le désaccord et explique les deux points de vue.

Le bibliothécaire vous donne un livre. L'assistant de recherche vous donne une réponse.

## Le RAG agentique avec Google Cloud

Google Cloud fournit plusieurs briques pour implémenter le RAG agentique :

### Le RAG Engine de Vertex AI

Le [Vertex AI RAG Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/rag-overview) fournit une infrastructure managée pour les pipelines RAG. Il gère l'ingestion de documents, le découpage (chunking), l'embedding, et la récupération, afin que vous puissiez vous concentrer sur la logique agentique.

Fonctionnalités clés :
- Recherche vectorielle managée avec indexation automatique
- Plusieurs connecteurs de sources de données (Cloud Storage, Google Drive, URL web)
- Stratégies de découpage et d'embedding configurables
- Intégration avec les endpoints de modèles de Vertex AI

### Le Vertex AI Agent Engine

Le [Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview) vous permet de construire des agents qui utilisent le RAG comme l'un de leurs outils. L'agent peut décider quand rechercher, quelles sources de données interroger, et comment combiner les résultats.

### Construire la boucle

Une implémentation pratique de RAG agentique sur Google Cloud pourrait ressembler à ceci :

```
Question de l'utilisateur
    |
    v
[Vertex AI Agent Engine]
    |-- Planifie la strategie de recuperation
    |-- Appelle le RAG Engine (possiblement plusieurs fois)
    |-- Evalue les resultats
    |-- Reformule si necessaire
    |-- Genere une reponse ancree
    |
    v
Reponse verifiee avec citations
```

## Patterns d'implémentation pratiques

### Pattern 1 : décomposition de requête

Décomposer les requêtes complexes en sous-requêtes plus simples avant la récupération.

```
Utilisateur : "Compare la velocite de notre equipe d'ingenierie ce
              trimestre avec le trimestre precedent et identifie
              les goulots d'etranglement"

L'agent decompose en :
  1. "Metriques de velocite de l'equipe d'ingenierie T3 2025"
  2. "Metriques de velocite de l'equipe d'ingenierie T2 2025"
  3. "Goulots d'etranglement d'ingenierie T3 2025"
  4. "Conclusions de retrospective de sprint T3 2025"

Chaque sous-requete est recuperee independamment, puis l'agent
synthetise une analyse comparative.
```

### Pattern 2 : récupération avec vérification

Après avoir généré une réponse, vérifier chaque affirmation par rapport aux documents source.

```
Reponse generee : "Le chiffre d'affaires a augmente de 15% au T3..."
Etape de verification :
  - Affirmation : "le chiffre d'affaires a augmente de 15%"
  - Le document source indique : "le chiffre d'affaires a cru de
    15,2% en glissement annuel"
  - Verdict : etaye (arrondi mineur)

Reponse generee : "...tire principalement par la region APAC"
Etape de verification :
  - Affirmation : "tire principalement par l'APAC"
  - Documents source : aucune mention de l'APAC comme facteur
    principal
  - Verdict : non etaye - necessite une nouvelle recuperation
```

### Pattern 3 : approfondissement itératif

Commencer par une recherche large et resserrer progressivement en fonction de ce que l'on trouve.

```
Tour 1 : recherche large pour "plaintes clients T3"
  -> Trouve : les themes courants incluent les retards de
     livraison, la qualite des produits

Tour 2 : recherche ciblee pour "cause racine des retards de
         livraison T3"
  -> Trouve : problemes d'effectifs en entrepot en septembre

Tour 3 : recherche precise pour "impact des effectifs d'entrepot
         en septembre"
  -> Trouve : augmentation de 40% du temps d'execution des commandes
     due au sous-effectif

L'agent dispose maintenant d'une chaine complete allant des
symptomes a la cause racine.
```

### Pattern 4 : triangulation des sources

Interroger plusieurs sources indépendantes et chercher un accord entre elles.

```
Question : "Quelle est la date de sortie prevue pour le Projet
           Phoenix ?"

Source 1 (Suivi de projet) : "Cible : 15 mars"
Source 2 (Notes de stand-up d'equipe) : "Vise mi-mars"
Source 3 (Mise a jour executive) : "Phoenix se lance dans la
          fenetre du 15-20 mars"

Agent : "Plusieurs sources convergent vers une sortie mi-mars,
        ciblant plus precisement la fenetre du 15 au 20 mars."
```

## Pièges courants

### Piège 1 : les boucles de récupération infinies

L'agent continue de rechercher parce qu'il ne considère jamais les résultats « assez bons ». Fixez toujours un nombre maximal d'itérations de récupération (généralement 3 à 5) et faites en sorte que l'agent fournisse sa meilleure réponse avec une réserve sur sa confiance quand la limite est atteinte.

### Piège 2 : la sur-récupération

L'agent récupère trop de documents et sature la context window. Fixez des limites sur le nombre de documents par étape de récupération et sur la taille totale du contexte. Priorisez la pertinence plutôt que la quantité.

### Piège 3 : ignorer le contexte récupéré

L'agent récupère des documents mais génère ensuite une réponse basée sur ses données d'entraînement plutôt que sur l'information récupérée. Utilisez des instructions d'ancrage strictes qui indiquent à l'agent de fonder sa réponse sur les documents récupérés et de les citer explicitement.

### Piège 4 : aucune solution de repli pour l'information manquante

L'agent essaie de répondre même quand la base de connaissances ne contient pas l'information nécessaire. Entraînez l'agent à reconnaître les lacunes et à dire « je n'ai pas trouvé d'information sur X dans les sources disponibles » plutôt que d'halluciner une réponse.

## Exercice pratique

Construisez un système de RAG agentique pour un cas d'usage de documentation technique :

1. **Configuration :** choisissez un ensemble de documents techniques (vos propres documents de projet, ou un ensemble de documentation publique comme la documentation Google Cloud).

2. **Base de référence RAG de base :** implémentez un pipeline simple de récupération-puis-lecture. Testez-le avec cinq questions de complexité variée.

3. **Ajoutez des capacités agentiques :** implémentez au moins deux des éléments suivants :
   - Décomposition de requête pour les questions complexes
   - Évaluation des résultats avec re-requête
   - Récupération multi-sources
   - Vérification des réponses par rapport aux sources

4. **Comparez :** exécutez les cinq mêmes questions à travers les deux systèmes. Documentez où la version agentique produit de meilleures réponses et où elle ajoute un surcoût inutile.

## Points clés à retenir

- Le RAG de base suit un pipeline fixe de récupération-puis-lecture. Le RAG agentique place un agent aux commandes de la récupération, le laissant décider quand rechercher, quoi rechercher, et si les résultats sont suffisants.
- La boucle du RAG agentique - planifier, récupérer, évaluer, raffiner, répondre, vérifier - remplace l'approche en une seule passe par un raffinement itératif.
- Les capacités clés incluent la planification autonome des requêtes, la sélection adaptative de sources, l'expansion sensible au contexte, et le raisonnement multi-sauts.
- L'autocorrection via la re-requête, les vérifications diagnostiques, et le recours à l'humain empêche le système de s'engager sur de mauvais résultats.
- Utilisez le RAG de base pour les recherches simples où la vitesse compte. Utilisez le RAG agentique pour les questions de recherche complexes où la précision compte plus que la latence.
- Fixez des limites claires sur les itérations de récupération et la taille du contexte pour éviter les boucles infinies et les coûts incontrôlés.

## Pour aller plus loin

- [Présentation du Vertex AI RAG Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/rag-overview) - Infrastructure RAG managée sur Google Cloud
- [Présentation du Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview) - Construire des agents qui utilisent le RAG comme outil sur Vertex AI
