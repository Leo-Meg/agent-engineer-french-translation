# Leçon 2 : comment pensent les agents - les LLM comme moteur de raisonnement

## Introduction

Dans la leçon 1, nous avons dit qu'un agent est constitué de trois parties : un modèle (le cerveau), des outils (les mains), et une couche d'orchestration (la boucle de contrôle). Dans cette leçon, nous nous concentrons sur le cerveau.

Le modèle de langage est le composant le plus important d'un agent. C'est la partie qui lit l'objectif de l'utilisateur, raisonne sur ce qu'il faut faire, décide quels outils appeler, interprète les résultats, et génère les réponses finales. Tout le reste dans le système d'agent existe pour soutenir ou étendre ce que le modèle peut faire.

Comprendre le fonctionnement des LLM (grands modèles de langage) - même à haut niveau - fera de vous un bien meilleur constructeur d'agents. Vous saurez pourquoi certains prompts (invites) fonctionnent et d'autres non. Vous comprendrez pourquoi les agents déraillent parfois. Et vous serez capable de prendre des décisions éclairées sur le choix du modèle, la conception des prompts, et l'architecture du système.

---

## Les LLM comme cerveau de l'agent

Un grand modèle de langage est un réseau de neurones entraîné sur d'énormes quantités de données textuelles. À la base, il ne fait qu'une seule chose : **prédire le prochain token** (jeton - grosso modo, le mot suivant ou un fragment de mot), en fonction de tout ce qui précède.

Cela semble simple, mais les capacités émergentes issues de cet entraînement sont remarquables :

- **Compréhension** : comprendre des questions et des instructions complexes
- **Raisonnement** : résoudre des problèmes de logique à plusieurs étapes
- **Planification** : décomposer un objectif en sous-tâches
- **Génération de code** : écrire et déboguer des logiciels
- **Sélection d'outils** : décider quelle fonction appeler et avec quels paramètres
- **Synthèse** : condenser de longs documents en points clés
- **Traduction** : convertir entre langues, formats et représentations

### Ce que les LLM font bien

| Capacité | Exemple dans le contexte d'un agent |
|---|---|
| Compréhension du langage naturel | Analyser la demande d'un utilisateur : « Annule ma commande la plus récente » |
| Raisonnement et planification | Décider : « D'abord je dois trouver la commande, puis vérifier si elle est annulable, puis l'annuler » |
| Sélection d'outils | Choisir d'appeler `lookup_order(user_id, sort="recent")` |
| Mise en forme de la sortie | Renvoyer une réponse JSON propre ou un message convivial |
| Interprétation des erreurs | Lire une erreur d'API et décider de retenter avec des paramètres différents |
| Synthèse de contexte | Combiner les résultats de plusieurs appels d'outils en une réponse cohérente |

### Ce que les LLM ne peuvent pas faire (sans aide)

| Limitation | Pourquoi c'est important |
|---|---|
| Pas d'accès aux données en temps réel | Les connaissances du modèle ont une date limite d'entraînement |
| Pas de garantie de calcul | Les LLM peuvent se tromper en mathématiques - ils prédisent des tokens, ils ne calculent pas |
| Pas de mémoire persistante | Chaque conversation repart de zéro, sauf si vous implémentez une mémoire |
| Aucune capacité d'action | Sans outils, le modèle ne peut que générer du texte |
| Risque d'hallucination | Les modèles peuvent générer une information plausible mais incorrecte |
| Limites de la context window (fenêtre de contexte) | Il existe une quantité maximale de texte que le modèle peut traiter à la fois |

C'est pour cela que les agents existent. Les outils compensent l'incapacité du modèle à agir et à accéder à des données en direct. L'orchestration compense son manque de mémoire persistante et sa tendance à dévier de sa trajectoire.

---

## Comment les modèles de langage traitent l'information

Vous n'avez pas besoin de comprendre en détail l'architecture Transformer pour construire des agents, mais comprendre trois concepts clés vous aidera à écrire de meilleurs prompts et à concevoir de meilleurs systèmes.

### Les tokens : l'unité de langage

Les LLM ne lisent ni des caractères ni des mots. Ils lisent des **tokens** - des fragments de texte que le modèle a appris à reconnaître pendant son entraînement. Un token peut être un mot entier, une partie de mot, ou un signe de ponctuation.

**Exemples :**

| Texte | Tokens approximatifs |
|---|---|
| « Hello » | 1 token |
| « Hello, world! » | 3 tokens |
| « ChatGPT is amazing » | 4 tokens |
| Une fonction de code typique (20 lignes) | 100-300 tokens |
| Une page complète de texte en anglais | ~500-700 tokens |

**Pourquoi c'est important pour les agents :**

- **Facturation** : la plupart des API facturent au token (entrée + sortie). Les workflows d'agents utilisent beaucoup plus de tokens que de simples prompts, car chaque itération de la boucle envoie le contexte complet.
- **Vitesse** : plus de tokens = plus de temps de génération. Gardez vos descriptions d'outils concises.
- **Limites de contexte** : il existe un nombre maximum de tokens que le modèle peut traiter en un seul appel. Si le contexte accumulé par votre agent dépasse cette limite, vous perdez de l'information.

### Les context windows : la mémoire de travail du modèle

La **context window** est le nombre total de tokens qu'un modèle peut prendre en compte à la fois. Voyez cela comme le bureau du modèle : tout ce dont il a besoin pour travailler doit tenir sur ce bureau.

| Modèle | Context window |
|---|---|
| Gemini 2.5 Pro | 1 000 000 tokens |
| Gemini 2.0 Flash | 1 000 000 tokens |
| GPT-4o | 128 000 tokens |
| Claude 3.5 Sonnet | 200 000 tokens |

**Ce qui entre dans la context window pendant un appel d'agent :**

```
+------------------------------------------+
| Instructions système ("Tu es un...")      |  ~200-500 tokens
+------------------------------------------+
| Définitions des outils (noms,             |  ~500-2000 tokens
| descriptions, schémas de paramètres)      |
+------------------------------------------+
| Historique de la conversation             |  Variable
+------------------------------------------+
| Appels d'outils et résultats précédents   |  Variable (peut croître vite)
+------------------------------------------+
| Message actuel de l'utilisateur           |  Variable
+------------------------------------------+
| = Le total doit tenir dans la context     |
|   window                                  |
+------------------------------------------+
```

**Pourquoi c'est important pour les agents :**

À mesure qu'un agent exécute plusieurs étapes, le contexte grandit à chaque appel d'outil et à chaque résultat. Un workflow d'agent à 5 étapes peut accumuler des milliers de tokens de résultats d'outils. Si vous n'y prenez pas garde, vous pouvez épuiser la context window en plein milieu d'une tâche.

Stratégies pour gérer cela :
- **Résumer** les résultats intermédiaires plutôt que de conserver les données brutes
- **Tronquer** les longues sorties d'outils pour ne garder que les parties pertinentes
- **Utiliser des modèles avec de grandes context windows** pour les tâches complexes à plusieurs étapes
- **Implémenter une sliding window (fenêtre glissante)** qui élimine le contexte le plus ancien et le moins pertinent

### L'attention : comment le modèle se concentre

Le **mécanisme d'attention** est ce qui permet au modèle de déterminer quelles parties du contexte sont pertinentes pour la décision en cours. Pour décider quel token générer ensuite, le modèle attribue des poids différents aux différentes parties de l'entrée.

Voyez cela comme la lecture d'un long document en surlignant les passages importants. Le modèle « surligne » les tokens les plus pertinents par rapport à ce qu'il essaie de faire à cet instant.

**Pourquoi c'est important pour les agents :**

- **Placez les informations importantes là où le modèle peut les trouver.** Les modèles ont tendance à accorder plus d'attention au début et à la fin du contexte. Les instructions critiques doivent se trouver dans le system prompt (au début) ou à proximité de la requête de l'utilisateur (à la fin).
- **Soyez précis et clair.** Des instructions vagues forcent le modèle à deviner ce qui compte. Des instructions précises facilitent le travail du mécanisme d'attention pour se rattacher à la bonne information.
- **La structure aide.** Des titres clairs, des listes numérotées, et une mise en forme cohérente aident le modèle à analyser le contenu et à porter son attention sur ce qui compte.

---

## Stratégies de raisonnement

La façon dont un agent « pense » à un problème dépend fortement de la manière dont vous formulez votre prompt au modèle. Différentes stratégies de raisonnement produisent des résultats très différents, en particulier pour les tâches complexes.

### Chain-of-Thought (CoT)

**Ce que c'est :** inciter le modèle à réfléchir à un problème étape par étape avant de donner une réponse finale.

**Comment ça marche :** au lieu de sauter directement à une réponse, le modèle génère des étapes de raisonnement intermédiaires. Cela améliore considérablement la précision sur les tâches qui demandent de la logique, des mathématiques, ou une analyse à plusieurs étapes.

**Exemple sans CoT :**
```
Prompt : « Si un serveur traite 100 requêtes/seconde et que nous avons 3
          serveurs, 40 % du trafic étant dirigé vers le serveur 1, combien
          de requêtes/seconde le serveur 1 traite-t-il ? »

Réponse du modèle : « 120 requêtes par seconde »  (faux)
```

**Exemple avec CoT :**
```
Prompt : « Réfléchis étape par étape. Si un serveur traite 100
          requêtes/seconde et que nous avons 3 serveurs, 40 % du trafic
          étant dirigé vers le serveur 1, combien de requêtes/seconde le
          serveur 1 traite-t-il ? »

Réponse du modèle :
« Étape 1 : la capacité totale est de 3 serveurs x 100 req/s = 300 req/s
  Étape 2 : le serveur 1 reçoit 40 % du trafic total
  Étape 3 : 40 % de 300 = 120 req/s
  Étape 4 : le serveur 1 peut traiter 100 req/s mais en reçoit 120
  Réponse : le serveur 1 reçoit 120 req/s mais ne peut en traiter que 100,
            il est donc surchargé de 20 req/s »
```

L'approche étape par étape a permis de détecter la situation de surcharge que la réponse directe avait manquée.

**Quand utiliser le CoT pour les agents :**
- Décisions complexes de sélection d'outils (« Parmi ces 5 outils, lequel est utile ici ? »)
- Planification multi-étapes (« Quelle séquence d'actions permet d'atteindre cet objectif ? »)
- Diagnostic d'erreurs (« L'outil a renvoyé une erreur - qu'est-ce qui n'a pas fonctionné et que dois-je essayer ensuite ? »)

### Tree-of-Thoughts (ToT)

**Ce que c'est :** une extension du Chain-of-Thought où le modèle explore plusieurs chemins de raisonnement, les évalue, et choisit le meilleur.

**Comment ça marche :** au lieu d'une seule chaîne de raisonnement, le modèle génère plusieurs approches possibles, note ou critique chacune d'elles, et poursuit avec le chemin le plus prometteur.

```
Objectif : « Optimiser cette requête de base de données lente »

Chemin A : « Ajouter un index sur la colonne de la clause WHERE »
  -> Évaluation : « Probablement efficace, faible risque, facile à mettre en oeuvre »

Chemin B : « Réécrire sous forme de vue matérialisée »
  -> Évaluation : « Pourrait aider, mais ajoute de la complexité et de la maintenance »

Chemin C : « Dénormaliser la structure de la table »
  -> Évaluation : « Pourrait fonctionner mais risque élevé, affecte d'autres requêtes »

Décision : commencer par le chemin A, essayer le chemin B si A est insuffisant
```

**Quand utiliser le ToT pour les agents :**
- Quand plusieurs approches valides existent et que vous voulez que le modèle prenne en compte les compromis
- Débogage complexe où la cause racine est incertaine
- Décisions d'architecture qui nécessitent d'évaluer des alternatives

**Compromis :** le ToT utilise plus de tokens et prend plus de temps. Réservez-le aux décisions où le coût d'un mauvais choix est élevé.

### Décomposition étape par étape

**Ce que c'est :** décomposer un objectif complexe en une séquence de sous-tâches plus simples avant d'en exécuter aucune.

**Comment ça marche :** l'agent crée d'abord un plan, puis exécute chaque étape du plan, en vérifiant sa progression au fur et à mesure.

```
Objectif utilisateur : « Mettre en place le monitoring pour notre nouvel endpoint API »

Plan :
1. Vérifier quels outils de monitoring sont actuellement configurés
2. Déterminer quelles métriques comptent pour cet endpoint (latence, taux d'erreur, débit)
3. Créer le tableau de bord de monitoring
4. Configurer les seuils d'alerte
5. Tester que les alertes se déclenchent correctement
6. Documenter la configuration du monitoring

Exécution : [se déroule étape par étape, chaque étape pouvant utiliser des outils]
```

**Quand utiliser la décomposition pour les agents :**
- Tâches multi-étapes où l'ordre compte
- Tâches où vous voulez que l'agent soit transparent sur son approche
- Workflows complexes qui bénéficient de checkpoints (points de contrôle)

---

## Choix du modèle : sélectionner le bon modèle pour la tâche

Toutes les tâches n'ont pas besoin du modèle le plus puissant. Choisir le bon modèle est une décision d'ingénierie qui équilibre capacité, coût, vitesse et fiabilité.

### Le spectre des modèles

```
Plus léger / Plus rapide / Moins cher        Plus lourd / Plus intelligent / Plus coûteux
|----------------------------------------------------------|
Gemini Flash          Gemini Pro          Gemini 2.5 Pro
(tâches simples)      (équilibré)         (raisonnement complexe)
```

### Quand utiliser quoi

| Type de tâche | Palier recommandé | Pourquoi |
|---|---|---|
| Classification (« Est-ce un spam ? ») | Léger (Flash) | Décision simple, pas besoin de raisonnement complexe |
| Extraction de données (« Extraire la date de cet e-mail ») | Léger (Flash) | Reconnaissance de motifs, sortie bien définie |
| Synthèse | Léger à moyen | Dépend de la longueur et de la complexité de la source |
| Raisonnement multi-étapes | Moyen à lourd (Pro) | Nécessite des chaînes logiques soutenues |
| Génération de code complexe | Lourd (2.5 Pro) | Nécessite une compréhension fine des patterns et des cas limites |
| Utilisation d'outils agentique | Moyen à lourd | La sélection d'outils et l'interprétation des résultats demandent un raisonnement solide |
| Écriture créative | Moyen | Bons résultats sans recourir aux modèles les plus lourds |

### Le model routing : utiliser différents modèles pour différentes étapes

Un système d'agent sophistiqué n'utilise pas le même modèle à chaque étape. C'est ce qu'on appelle le **model routing** (routage de modèles) - diriger différentes parties du workflow vers différents modèles en fonction de leur complexité.

**Exemple d'architecture :**

```
Arrivée de la requête utilisateur
    |
    v
[Modèle léger : classifier l'intention]  --> "order_status"
    |
    v
[Modèle léger : extraire les paramètres]  --> order_id : 12345
    |
    v
[Appel d'outil : consulter la commande]  --> données de statut
    |
    v
[Modèle léger : formater la réponse]  --> "Votre commande #12345 a été expédiée le 15 mars"
```

Dans ce flux, chaque étape utilise un modèle rapide et bon marché, car aucune des étapes individuelles ne nécessite un raisonnement lourd. Le coût total et la latence sont bien plus faibles que si l'on utilisait un modèle de pointe pour l'intégralité de l'interaction.

**Comparez avec une tâche plus difficile :**

```
Utilisateur : "Relis cette pull request et suggère des améliorations"
    |
    v
[Modèle lourd : analyser les changements de code, raisonner
 sur les patterns, identifier les bugs, suggérer des améliorations]
    |
    v
[Renvoyer une relecture détaillée]
```

Cette tâche nécessite un raisonnement approfondi, elle justifie donc un modèle plus performant.

### Les options de modèles Google Cloud

Google Cloud donne accès aux modèles Gemini via Vertex AI :

- **Gemini 2.0 Flash** - rapide et efficace pour la plupart des tâches d'agent. Grande context window (1 M de tokens). Bon équilibre entre capacité et vitesse.
- **Gemini 2.5 Pro** - raisonnement de premier ordre pour les tâches complexes. À utiliser quand la tâche nécessite une analyse approfondie, une logique multi-étapes complexe, ou une compréhension nuancée.
- **Gemini 2.0 Flash Lite** - l'option la plus rapide et la plus économique pour les tâches simples comme la classification et l'extraction.

> **Pour en savoir plus :** [Documentation des modèles Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/models)

> **Pour en savoir plus :** [Documentation de l'API Gemini](https://ai.google.dev/gemini-api/docs)

---

## Le rôle des instructions système

Les instructions système sont la « fiche de poste » de l'agent. Elles indiquent au modèle qui il est, ce qu'il peut faire, comment il doit se comporter, et ce qu'il ne doit pas faire.

### Ce que contiennent les instructions système

Une instruction système bien écrite pour un agent inclut généralement :

1. **Définition du rôle** : ce qu'est l'agent et qui il sert
2. **Capacités** : quels outils sont disponibles et quand les utiliser
3. **Contraintes** : ce que l'agent ne doit pas faire
4. **Format de sortie** : comment structurer les réponses
5. **Gestion des erreurs** : que faire quand les choses tournent mal
6. **Personnalité/ton** : comment communiquer (si pertinent)

### Exemple : agent de support client

```
Vous êtes un agent de support client pour Acme Corp, un revendeur en ligne
d'appareils électroniques.

Votre rôle :
- Aider les clients avec leurs demandes de commande, retours et questions produits
- Être amical, professionnel et concis

Outils disponibles :
- lookup_order(order_id) : renvoie le statut de la commande, les articles
  et les informations de livraison
- initiate_return(order_id, reason) : démarre une procédure de retour
- search_products(query) : recherche dans le catalogue produits
- escalate_to_human(reason) : transfère vers un agent humain

Consignes :
- Toujours vérifier l'identité du client avant d'accéder aux informations
  de commande
- Si vous ne pouvez pas résoudre un problème en 3 tentatives, escaladez
  vers un agent humain
- Ne jamais inventer d'informations de commande - toujours utiliser l'outil
  de consultation
- Ne pas proposer de remises ou de remboursements au-delà de la politique
  standard
- Garder des réponses de moins de 3 paragraphes

Quand vous rencontrez une erreur provenant d'un outil :
- Si c'est une erreur temporaire (timeout, 500), réessayez une fois
- Si c'est une erreur permanente (non trouvé, non autorisé), expliquez le
  problème au client
- Si vous n'êtes pas sûr, escaladez vers un agent humain
```

### Conseils pour écrire des instructions système efficaces

| À faire | À ne pas faire |
|---|---|
| Être précis sur l'utilisation des outils | Laisser la sélection d'outils ambiguë |
| Définir des limites claires | Supposer que le modèle connaît vos règles métier |
| Inclure des consignes de gestion des erreurs | Espérer que le modèle se débrouille seul face aux erreurs |
| Spécifier le format de sortie | Laisser le modèle choisir son propre format à chaque fois |
| Utiliser des exemples concrets | Écrire des instructions abstraites et vagues |
| Garder des instructions concises | Écrire un essai de 10 pages (gaspille le contexte) |

### L'ordre compte

Les modèles accordent plus d'attention aux instructions situées au début et à la fin du system prompt. Structurez vos instructions système ainsi :

```
1. Règles les plus critiques (identité, contraintes de sécurité)  <-- Début
2. Consignes d'utilisation des outils
3. Format de sortie
4. Exemples
5. Gestion des cas limites
6. Rappel des règles les plus critiques                           <-- Fin
```

Cela tire parti des effets de « primacy and recency » (primauté et récence) dans l'attention du modèle.

---

## Température, échantillonnage et comportement de l'agent

Quand un modèle de langage génère du texte, il ne se contente pas de choisir le token suivant le plus probable. Il échantillonne à partir d'une distribution de probabilité sur tous les tokens possibles. Les paramètres qui contrôlent cet échantillonnage ont un impact important sur le comportement de l'agent.

### La température

La **température** contrôle le degré d'aléatoire des sorties du modèle.

- **Température 0 (ou très basse)** : le modèle choisit presque toujours le token le plus probable. Les sorties sont déterministes et ciblées.
- **Température 1** : le modèle échantillonne proportionnellement aux probabilités des tokens. Les sorties sont plus variées et plus créatives.
- **Température > 1** : le modèle devient de plus en plus aléatoire. Les sorties deviennent imprévisibles.

**Analogie visuelle :**

```
Température 0 :   "La capitale de la France est Paris."
                  (Toujours la même réponse)

Température 0.7 : "La capitale de la France est Paris, une ville connue pour..."
                  (Légère variation dans le développement)

Température 1.5 : "La capitale de la France trouve ses racines historiques dans..."
                  (Plus créatif, potentiellement hors sujet)
```

### Quelle température utiliser pour les agents

| Tâche de l'agent | Température recommandée | Pourquoi |
|---|---|---|
| Sélection d'outils | 0 - 0,2 | Vous voulez des appels d'outils déterministes et corrects |
| Extraction de données | 0 | Réponses exactes, pas besoin de créativité |
| Génération de code | 0 - 0,3 | La justesse compte plus que la variété |
| Planification | 0,2 - 0,5 | Un peu de flexibilité aide à explorer les options |
| Écriture créative | 0,7 - 1,0 | La variété et l'originalité sont valorisées |
| Brainstorming | 0,8 - 1,0 | On veut des idées diverses |

**Pour la plupart des cas d'usage d'agents, gardez une température basse (0 à 0,3).** Les agents doivent prendre des décisions fiables concernant l'utilisation des outils, l'extraction de paramètres, et le raisonnement. Une température élevée introduit de l'aléatoire là où vous voulez de la cohérence.

### L'échantillonnage Top-K et Top-P

Ce sont des contrôles supplémentaires sur la façon dont le modèle sélectionne les tokens.

**Top-K :** ne considérer que les K tokens les plus probables. Si K=50, le modèle ignore tous les tokens en dehors des 50 meilleurs candidats.

**Top-P (nucleus sampling, échantillonnage par noyau) :** ne considérer que les tokens dont la probabilité cumulée atteint P. Si P=0.9, le modèle considère le plus petit ensemble de tokens dont la probabilité totale atteint 90 %.

```
Probabilités des tokens : [0.4, 0.25, 0.15, 0.08, 0.05, 0.03, 0.02, ...]

Top-K=3 :   ne considérer que [0.4, 0.25, 0.15]
Top-P=0.8 : ne considérer que [0.4, 0.25, 0.15] (cumul = 0.8)
Top-P=0.9 : ne considérer que [0.4, 0.25, 0.15, 0.08] (cumul = 0.88... arrondi)
```

**Pour les agents :** utilisez des réglages conservateurs. Un Top-P autour de 0,9 et des valeurs de Top-K modérées constituent des valeurs par défaut raisonnables. Les réglages par défaut du modèle conviennent généralement pour le travail avec des agents - la température est le paramètre que vous aurez le plus souvent besoin d'ajuster.

### Comment l'échantillonnage affecte la boucle de l'agent

Prenons un agent qui doit décider quel outil appeler. Avec une température basse, il choisira systématiquement le même outil (généralement le bon) pour une situation donnée. Avec une température élevée, il pourrait choisir des outils différents d'une exécution à l'autre, ce qui entraîne un comportement incohérent.

```
Utilisateur : "Quel temps fait-il à Tokyo ?"

Température basse (0) :
  -> L'agent pense : "J'ai besoin de l'outil météo"
  -> Appelle : get_weather(city="Tokyo")
  -> Cohérent, prévisible

Température élevée (1.2) :
  -> Exécution 1 : appelle get_weather(city="Tokyo")
  -> Exécution 2 : appelle web_search("prévisions météo Tokyo")
  -> Exécution 3 : essaie de répondre à partir des données d'entraînement
     (aucun appel d'outil)
  -> Incohérent, difficile à déboguer
```

Pour les agents en production, un comportement déterministe est presque toujours ce que vous voulez.

---

## ELI5 : comment fonctionne le cerveau du LLM

### Pensez au LLM comme à un chef cuisinier

Imaginez que vous dirigiez la cuisine d'un restaurant, et que le LLM soit votre chef cuisinier.

**L'entraînement du chef (l'entraînement du modèle) :**
Le chef a passé des années à étudier des milliers de livres de cuisine, à regarder des émissions culinaires, et à s'entraîner sur des recettes. Il n'a pas mémorisé chaque recette mot pour mot, mais il a développé une intuition profonde sur les saveurs qui se marient bien, les techniques qui fonctionnent pour tels ingrédients, et comment improviser quand il manque quelque chose.

**Les tokens sont comme des ingrédients :**
Le chef ne pense pas en termes de plats complets tout à la fois. Il pense en termes d'ingrédients individuels et d'étapes. « D'abord l'oignon, puis l'ail, puis les tomates... » Chaque choix d'ingrédient influence le suivant. C'est ainsi que fonctionne la prédiction de tokens - chaque token est choisi en fonction de tous les tokens qui le précèdent.

**La context window, c'est comme l'espace du plan de travail :**
Le chef ne peut travailler qu'avec ce qui tient sur le plan de travail de la cuisine. Si le plan de travail est immense (1 million de tokens), il peut avoir beaucoup d'ingrédients, de recettes et de préparations visibles en même temps. S'il est petit, il doit ranger des choses pour faire de la place, et pourrait oublier ce qu'il était en train de faire.

**La température, c'est comme l'humeur du chef :**
- Température basse : le chef est concentré et méthodique. Il suit la recette à la lettre. Chaque fois que vous commandez le même plat, il a le même goût.
- Température élevée : le chef se sent créatif. Il improvise, substitue des ingrédients, essaie de nouvelles choses. Parfois le résultat est formidable, parfois il est étrange.

**Les instructions système, c'est comme le concept du restaurant :**
« Vous êtes un bistrot français. Vous utilisez des techniques traditionnelles. Vous ne servez pas de sushis. » Cela façonne chaque décision du chef, sans qu'il soit nécessaire de le répéter pour chaque plat.

**Les outils, ce sont comme les équipements de cuisine :**
Les connaissances du chef seules ne cuisinent pas la nourriture. Il a besoin d'un four, d'une cuisinière, de couteaux, et d'instruments de mesure. De la même manière, le raisonnement seul du LLM ne consulte pas de données et n'appelle pas d'API. Il a besoin d'outils.

**La boucle de l'agent, c'est comme le défi d'une émission de cuisine :**
Le chef reçoit un défi (« Prépare un repas de trois plats pour quelqu'un qui est intolérant au gluten »). Il planifie son approche, commence à cuisiner, goûte au fur et à mesure, ajuste l'assaisonnement, dresse l'assiette, et évalue le résultat. Si la sauce tranche, il dépanne la situation et s'adapte. Cette boucle planifier-agir-observer-ajuster est exactement ce que fait un agent.

---

## Pour résumer : comment le choix du modèle affecte la qualité de l'agent

Voici un exemple concret de la façon dont ces concepts se combinent dans un scénario d'agent réel.

### Scénario : un agent de triage de bugs

Votre équipe souhaite un agent qui lit les nouveaux rapports de bugs, les catégorise par gravité, les assigne à la bonne équipe, et rédige un premier plan d'investigation.

**Décision de choix du modèle :**

| Étape | Choix du modèle | Justification |
|---|---|---|
| Classifier la gravité (P0-P3) | Flash (léger) | Classification simple avec des critères clairs |
| Assigner à une équipe | Flash (léger) | Décision de type recherche, fondée sur le composant |
| Rédiger le plan d'investigation | Pro (lourd) | Nécessite de comprendre le bug, les systèmes associés, et de suggérer des étapes de diagnostic |

**Décisions de température :**

| Étape | Température | Justification |
|---|---|---|
| Classifier la gravité | 0 | Doit être déterministe - un même bug doit toujours recevoir la même gravité |
| Assigner à une équipe | 0 | Doit être cohérent - un même composant doit toujours être routé vers la même équipe |
| Rédiger le plan d'investigation | 0,3 | Un peu de flexibilité aide à générer des pistes d'investigation plus utiles et variées |

**Extrait d'instruction système :**

```
Vous êtes un agent de triage de bugs pour l'équipe Platform Engineering.

Classification de gravité :
- P0 : le service est en panne ou une perte de données est en cours
- P1 : une fonctionnalité majeure est cassée, aucune solution de contournement
- P2 : une fonctionnalité est dégradée mais une solution de contournement existe
- P3 : problème mineur, cosmétique, ou demande d'amélioration

Routage vers les équipes :
- Problèmes d'authentification/connexion -> équipe Identity
- Erreurs d'API -> équipe Platform
- Problèmes d'interface -> équipe Frontend
- Base de données/performance -> équipe Infrastructure

Pour rédiger un plan d'investigation :
- Commencer par la cause racine la plus probable
- Lister 3 à 5 étapes de diagnostic par ordre de priorité
- Inclure les requêtes de logs ou les liens de tableaux de bord pertinents
- Noter tout déploiement récent qui pourrait être lié
```

Cet exemple montre comment comprendre les capacités du modèle, régler des paramètres appropriés, et écrire des instructions claires contribuent tous à un agent fiable.

---

## Erreurs courantes lorsqu'on travaille avec des LLM dans des agents

### 1. faire confiance au modèle pour les mathématiques

Les LLM prédisent des tokens, ils ne calculent pas d'équations. Pour tout calcul qui compte, utilisez un outil d'exécution de code.

```
Mauvais : "Calcule le coût total de 47 articles à 23,99 $ chacun"
      -> Le modèle pourrait répondre 1 127,53 $ (la bonne réponse est
         1 127,53 $, mais c'est un coup de chance - il se trompe souvent)

Bon : faites appeler par l'agent un outil de calcul ou un outil
      d'exécution de code
      -> calculate("47 * 23.99") -> 1 127,53 $ (résultat garanti correct)
```

### 2. supposer une mémoire parfaite

Le modèle ne « se souvient » que de ce qui se trouve dans sa context window actuelle. Si l'information de l'étape 1 est tronquée d'ici l'étape 10, le modèle ne s'en souviendra pas.

### 3. trop dépendre d'un seul modèle

Utiliser un modèle de pointe à chaque étape gaspille de l'argent et ajoute de la latence. Utilisez le model routing pour faire correspondre la capacité du modèle à la complexité de la tâche.

### 4. négliger le system prompt

Un system prompt bien conçu peut faire la différence entre un agent qui fonctionne 50 % du temps et un autre qui fonctionne 95 % du temps. Investissez du temps à écrire et à itérer sur vos instructions système.

### 5. ne pas tenir compte des hallucinations

Les LLM génèrent avec assurance des informations plausibles mais incorrectes. Pour tout fait qui compte, ancrez la réponse de l'agent dans des résultats d'outils plutôt que dans les données d'entraînement du modèle.

---

## Points clés à retenir

1. **Le LLM est le moteur de raisonnement** de votre agent. Comprendre ses capacités et ses limites est fondamental pour construire de bons agents.

2. **Les tokens, les context windows et l'attention** sont les trois concepts clés. Les tokens déterminent le coût et la vitesse. Les context windows déterminent la quantité d'information avec laquelle le modèle peut travailler. L'attention détermine sur quoi le modèle se concentre.

3. **Les stratégies de raisonnement comptent.** Le Chain-of-Thought, le Tree-of-Thoughts, et la décomposition étape par étape peuvent considérablement améliorer la performance des agents sur des tâches complexes.

4. **Choisissez le bon modèle pour chaque étape.** Utilisez des modèles plus légers pour les tâches simples et des modèles plus lourds pour le raisonnement complexe. Le model routing réduit le coût et la latence.

5. **Les instructions système sont votre principal levier pour contrôler le comportement de l'agent.** Rédigez-les avec soin, soyez précis, et itérez en fonction des tests.

6. **Gardez une température basse pour les agents.** Un comportement déterministe est presque toujours préférable pour les systèmes d'agents en production.

---

## Et maintenant ?

Le cerveau, c'est important, mais un agent qui ne fait que penser n'est qu'un chatbot. Dans la prochaine leçon, nous donnons des mains à notre agent : des outils qui lui permettent d'interagir avec le monde, d'appeler des API, de rechercher sur le web, et d'exécuter du code.

[Suivant : Leçon 3 - Les outils : donner des mains aux agents -->](../03-tools-giving-agents-hands/README.md)
