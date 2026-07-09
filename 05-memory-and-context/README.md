# Leçon 5 : mémoire et contexte - comment les agents se souviennent

## Ce que vous allez apprendre

- Comment fonctionne la context window (fenêtre de contexte) et pourquoi elle compte
- Le context engineering (ingénierie du contexte) comme évolution du prompt engineering (ingénierie de prompt)
- Les types de mémoire d'agent : court terme, long terme, procédurale et déclarative
- Comment les sessions donnent aux agents une continuité conversationnelle
- La différence entre la mémoire et le RAG
- Ce qu'est le context rot (dégradation du contexte) et comment le combattre
- Des stratégies pratiques pour gérer le contexte
- Les options de stockage de mémoire : bases de données vectorielles, graphes de connaissances, et approches hybrides

## Prérequis

- [Leçon 2 : Comment pensent les agents](../02-how-agents-think/README.md)
- [Leçon 4 : Design patterns agentiques](../04-agentic-design-patterns/README.md)

---

## ELI5 : pensez au contexte comme à un bureau

Imaginez que vous travailliez à un bureau. Le bureau, c'est la context window de votre agent - c'est là que vous posez tout ce sur quoi vous travaillez activement.

Le bureau a un espace limité. Vous pouvez y étaler des notes, des livres de référence, votre ordinateur portable, et une tasse de café. Mais si vous continuez à empiler des choses, le bureau finit par devenir si encombré que vous ne trouvez plus rien. Des notes importantes se retrouvent enfouies sous des papiers moins pertinents. Vous commencez à perdre le fil de ce que vous étiez en train de faire.

C'est exactement ce qui se passe avec le contexte d'un agent IA. La context window est la surface de travail de l'agent. Tout ce à quoi l'agent doit penser - la question de l'utilisateur, l'historique de la conversation, les résultats d'outils, les instructions - doit tenir sur ce bureau. Quand le bureau est plein, l'agent doit décider ce qui reste et ce qui part.

Une bonne gestion de la mémoire, c'est comme être organisé à son bureau : garder les éléments importants visibles et accessibles, tout en rangeant les informations moins critiques quelque part où vous pourrez les retrouver plus tard.

---

## La context window expliquée

### Qu'est-ce qu'une context window ?

La context window est la quantité totale de texte (mesurée en tokens) qu'un LLM peut traiter en une seule requête. Voyez-la comme la mémoire de travail du modèle - tout ce qu'il peut « voir » à la fois.

```
+------------------------------------------------------------------+
|                       CONTEXT WINDOW                             |
|                                                                  |
|  [Instructions systeme]                                          |
|  [Definitions des outils]                                        |
|  [Historique de conversation - messages utilisateur et agent]    |
|  [Documents recuperes / resultats RAG]                           |
|  [Message actuel de l'utilisateur]                               |
|  [Reflexions en cours de l'agent]                                |
|                                                                  |
|                 ... tout doit tenir ici ...                       |
+------------------------------------------------------------------+
```

### Tailles des context windows

Les context windows ont considérablement grandi :

| Modèle | Context window |
|-------|---------------|
| GPT-3 (2020) | 4 K tokens |
| GPT-4 (2023) | 128 K tokens |
| Gemini 1.5 Pro | 1 M tokens |
| Gemini 2.0 | 1 M+ tokens |

Un token représente environ 3/4 d'un mot en anglais. 1 million de tokens représente donc environ 750 000 mots - soit à peu près l'équivalent de 10 romans.

### Plus grand n'est pas toujours mieux

Vous pourriez penser qu'une context window plus grande résout tous les problèmes. Elle aide, mais il y a des compromis :

| Facteur | Petit contexte | Grand contexte |
|--------|--------------|---------------|
| **Coût** | Moins cher par requête | Plus cher par requête |
| **Latence** | Réponses plus rapides | Réponses plus lentes |
| **Précision** | Ciblé, moins de bruit | Peut avoir du mal à trouver l'information pertinente dans une mer de texte |
| **Simplicité** | Vous force à être sélectif | Tentant de tout y déverser |

Des recherches ont montré que les LLM peuvent avoir du mal avec l'information placée au milieu de très longs contextes - un effet parfois appelé « perdu au milieu » (lost in the middle). Ce n'est pas parce que vous *pouvez* faire tenir 1 M de tokens que vous *devriez* le faire.

---

## Du prompt engineering au context engineering

### Le prompt engineering : le point de départ

Le prompt engineering (ingénierie de prompt) consiste à concevoir le bon texte d'entrée pour obtenir une bonne sortie d'un LLM. Vous écrivez un prompt statique, peut-être avec quelques exemples, et vous l'envoyez.

Cela fonctionne bien pour des tâches simples, mais cela s'effondre pour les agents. Pourquoi ? Parce que les agents traitent une information dynamique et changeante :

- L'historique de conversation grandit à chaque tour
- Les résultats d'outils arrivent au moment de l'exécution
- Les documents récupérés varient selon la requête
- Les besoins de l'utilisateur évoluent en cours de session

### Le context engineering : l'étape suivante

Le context engineering (ingénierie du contexte) est la pratique consistant à assembler dynamiquement la bonne information dans la context window au bon moment. Au lieu d'un prompt statique, vous construisez un système qui décide :

- **Ce qui entre :** quelles informations sont pertinentes en ce moment ?
- **Ce qui reste dehors :** qu'est-ce qui peut être résumé, stocké en externe, ou supprimé ?
- **Dans quel ordre :** comment l'information doit-elle être organisée pour que le modèle la traite au mieux ?
- **Quand rafraîchir :** quand le contexte ancien doit-il être mis à jour ou remplacé ?

Voyez le prompt engineering comme le fait d'écrire un bon e-mail. Le context engineering, c'est construire le système de messagerie.

### Pourquoi le context engineering compte pour les agents

```
Prompt Engineering :             Context Engineering :
+------------------+          +---------------------------+
| Prompt statique  |          | Instructions systeme      |
| + question       |          | + Memoire pertinente      |
|   utilisateur    |          | + Historique de session   |
|                  |          |   (elague)                |
|                  |          | + Schemas d'outils        |
|                  |          |   (selectionnes)          |
|                  |          | + Contexte recupere       |
|                  |          | + Etat actuel de la tache |
|                  |          |                           |
| "Reponds a ceci" |          | Tout assemble             |
|                  |          | dynamiquement             |
+------------------+          +---------------------------+
```

Un agent qui réserve un hôtel a besoin d'un contexte différent d'un agent qui écrit du code. Même un même agent a besoin d'un contexte différent selon les étapes d'une tâche. Le context engineering est ce qui rend cet assemblage dynamique possible.

---

## Types de mémoire

Les agents ont besoin de différents types de mémoire pour différents usages. Cela reflète le fonctionnement de la mémoire humaine.

### La mémoire à court terme (session/conversation)

**Ce que c'est :** la conversation actuelle entre l'utilisateur et l'agent. Elle vit directement dans la context window.

**Analogie humaine :** votre mémoire de travail - ce à quoi vous pensez activement en ce moment.

**Caractéristiques :**
- Existe pendant la durée d'une session
- Grandit à chaque interaction
- Limitée par la taille de la context window
- Perdue à la fin de la session (sauf si elle est persistée)

**Exemple :**
```
Utilisateur : "Je m'appelle Alex et je préfère le mode sombre."
Agent : "Compris, Alex. J'utiliserai les réglages en mode sombre."

[50 messages plus tard]

Utilisateur : "Quel thème j'utilise ?"
Agent : "Vous utilisez le mode sombre, Alex."  <-- Ne fonctionne que si le
                                                    message précédent est
                                                    encore dans le contexte
```

### La mémoire à long terme (persistée entre les sessions)

**Ce que c'est :** de l'information stockée en dehors de la context window, qui persiste entre les sessions. L'agent la récupère quand c'est pertinent.

**Analogie humaine :** vos souvenirs à long terme - des faits que vous avez appris et des expériences dont vous vous souvenez, même si vous n'y pensez pas en ce moment.

**Caractéristiques :**
- Survit entre les sessions
- Stockée en externe (base de données, système de fichiers, base vectorielle)
- Doit être explicitement récupérée et chargée dans le contexte
- Peut grandir sans limite

**Exemples d'usage :**
- Préférences utilisateur (« Alex aime le mode sombre et utilise TypeScript »)
- Interactions passées (« La semaine dernière, Alex a posé une question sur le déploiement vers Cloud Run »)
- Faits appris (« La base de données de production est sur us-central1 »)

### La mémoire procédurale (comment faire les choses)

**Ce que c'est :** la connaissance de *comment* accomplir des tâches - workflows, procédures opérationnelles standard, et processus étape par étape.

**Analogie humaine :** savoir faire du vélo ou taper au clavier. Vous ne pensez pas à chaque étape - vous savez juste comment faire.

**Caractéristiques :**
- Souvent encodée dans les instructions système ou les définitions d'outils
- Relativement stable - ne change pas souvent
- Inclut des patterns, des modèles, et des procédures standard

**Exemple :**
```
Mémoire procédurale : "Quand un utilisateur signale un bug :
  1. Demander les étapes de reproduction
  2. Vérifier les logs d'erreurs
  3. Rechercher des problèmes similaires
  4. Proposer un correctif ou une solution de contournement"
```

### La mémoire déclarative (faits et connaissances)

**Ce que c'est :** l'information factuelle que l'agent connaît ou à laquelle il a accès - données, documents, spécifications, et documentation de référence.

**Analogie humaine :** savoir que Paris est la capitale de la France. Un fait que vous pouvez énoncer, pas une compétence que vous exécutez.

**Caractéristiques :**
- Peut être récupérée dynamiquement via le RAG
- Inclut la documentation, les bases de données, et les bases de connaissances
- Peut devenir obsolète et nécessiter une mise à jour

### Résumé des types de mémoire

| Type de mémoire | Durée | Emplacement | Mise à jour | Exemple |
|------------|----------|----------|-------------|---------|
| **Court terme** | Une session | Context window | Automatiquement (la conversation grandit) | Messages du chat en cours |
| **Long terme** | Entre les sessions | Stockage externe | Explicitement, par l'agent ou le système | Préférences utilisateur |
| **Procédurale** | Permanente | Prompt système / configuration | Par les développeurs | Instructions de workflow |
| **Déclarative** | Variable | Bases de connaissances / RAG | Par les pipelines de données | Documentation produit |

---

## Les sessions : des conteneurs pour les conversations

Une session est le conteneur qui héberge une conversation entre un utilisateur et un agent. Les sessions donnent aux agents une continuité - la capacité de se souvenir de ce qui s'est passé jusqu'ici dans cette interaction.

### Que contient une session ?

```
Session
+-----------------------------------------------+
| ID de session : abc-123                       |
| ID utilisateur : user-456                     |
| Creee le : 2025-01-15T10:30:00Z               |
|                                               |
| Evenements :                                  |
|   [Message utilisateur : "Aide-moi a          |
|    planifier un voyage"]                      |
|   [Message agent : "Ou aimeriez-vous..."]     |
|   [Message utilisateur : "Tokyo, 5 jours"]    |
|   [Appel d'outil : flight_search(dest="Tokyo")]|
|   [Resultat d'outil : {flights: [...]}]       |
|   [Message agent : "J'ai trouve plusieurs..."] |
|                                               |
| Etat :                                        |
|   destination : "Tokyo"                       |
|   duree : "5 jours"                           |
|   budget : null                               |
+-----------------------------------------------+
```

### Sessions contre état

- **Session :** l'historique complet des événements (messages, appels d'outils, résultats) dans une conversation.
- **État :** un résumé structuré des informations clés extraites de la session. Voyez cela comme le bloc-notes de l'agent.

L'état est utile car il donne à l'agent un accès rapide aux faits importants, sans avoir à relire tout l'historique de la conversation.

### La gestion des sessions sur Google Cloud

Google Cloud fournit une gestion des sessions via l'Agent Development Kit (ADK) et Vertex AI :

- **Les sessions ADK :** le [système de sessions ADK](https://google.github.io/adk-docs/sessions/) fournit une gestion des sessions intégrée, avec suivi des événements et de l'état.
- **Les sessions Vertex AI :** les [sessions de Vertex AI Agent Engine](https://cloud.google.com/agent-builder/agent-engine/sessions/overview) offrent un stockage de sessions managé avec mise à l'échelle automatique.

Ces outils gèrent l'infrastructure de stockage et de récupération des sessions, afin que vous puissiez vous concentrer sur la logique de l'agent.

---

## Mémoire contre RAG

La mémoire et le RAG (Retrieval-Augmented Generation, génération augmentée par récupération) sont liés mais servent des objectifs différents. Les ingénieurs les confondent souvent.

### L'assistant personnel contre la bibliothèque

La **mémoire**, c'est comme avoir un assistant personnel qui se souvient de vos préférences, de votre emploi du temps, et de vos conversations passées. Il vous connaît, *vous*.

Le **RAG**, c'est comme avoir accès à une bibliothèque. Quand vous avez besoin d'un fait, vous le recherchez. La bibliothèque ne vous connaît pas personnellement - elle dispose simplement de beaucoup d'informations disponibles.

### Comparaison côte à côte

| Aspect | Mémoire | RAG |
|--------|--------|-----|
| **Ce qui est stocké** | Des données spécifiques à l'utilisateur et à l'interaction | Des connaissances générales, documents, données |
| **De qui il s'agit** | Cet utilisateur, cet agent, ce contexte | N'importe qui - c'est une connaissance partagée |
| **Quand c'est écrit** | Pendant les interactions avec l'agent | Pendant l'ingestion des données (généralement hors ligne) |
| **Quand c'est lu** | Au début ou pendant une session | Quand l'agent a besoin d'une information spécifique |
| **Personnalisation** | Élevée - unique à l'utilisateur | Faible - identique pour tout le monde |
| **Exemple** | « Cet utilisateur préfère des réponses concises » | « La limite de débit de l'API est de 100 requêtes/minute » |

### Comment ils fonctionnent ensemble

En pratique, les agents utilisent les deux :

```
Utilisateur : "Rappelle-moi, c'était quoi ce problème de déploiement
              qu'on a eu le mois dernier ?"

Agent :
  1. Vérifier la MEMOIRE : "Cet utilisateur travaille dans l'équipe
     paiements et a eu un echec de deploiement Cloud Run le 3 janvier."
  2. Utiliser le RAG : rechercher dans les rapports d'incidents
     "echec de deploiement Cloud Run janvier" pour obtenir des
     details precis.
  3. Combiner : "Le mois dernier, vous avez eu un echec de deploiement
     Cloud Run lie a un compte de service mal configure. Voici les
     details du rapport d'incident..."
```

La mémoire indique à l'agent *quoi rechercher*. Le RAG fournit l'*information détaillée*.

Nous couvrirons le RAG en profondeur dans la [leçon 8 : RAG agentique](../08-agentic-rag/README.md).

---

## Le context rot

### Qu'est-ce que le context rot ?

Le context rot (dégradation du contexte) se produit quand une information critique se perd, se dilue, ou s'enfouit à mesure que la context window se remplit. L'agent « oublie » des choses importantes - non pas parce que l'information a été supprimée, mais parce qu'elle ne se trouve plus dans la partie du contexte à laquelle le modèle prête attention.

### L'analogie du bureau encombré

Vous vous souvenez de l'analogie du bureau ? Le context rot, c'est ce qui se passe quand votre bureau devient si encombré que le post-it critique contenant le mot de passe de la base de données se retrouve enfoui sous une pile de notes de réunion. Le post-it est techniquement toujours sur le bureau, mais vous ne pouvez pas le retrouver quand vous en avez besoin.

### Comment le context rot se produit

1. **Les longues conversations.** Chaque tour ajoute des messages au contexte. Après 50 tours ou plus, les premiers messages sont loin de l'endroit où le modèle se concentre.

2. **Les résultats d'outils verbeux.** Un outil renvoie un gros blob JSON. La majeure partie n'est pas pertinente, mais elle occupe un précieux espace de contexte.

3. **Les instructions accumulées.** Les instructions système, les exemples few-shot, et les guardrails occupent de l'espace qui pourrait contenir des informations pertinentes pour l'utilisateur.

4. **Le contenu répétitif.** Des messages ou des résultats similaires s'accumulent sans être consolidés.

### Les signes du context rot

- L'agent « oublie » des choses que l'utilisateur lui a dites plus tôt dans la conversation
- L'agent se contredit par rapport à des déclarations ou décisions antérieures
- Les résultats d'outils du début de la session sont ignorés
- L'agent demande une information que l'utilisateur a déjà fournie

---

## Stratégies pour gérer le contexte

### 1. La sliding window (fenêtre glissante)

**Comment ça marche :** ne garder que les N messages les plus récents dans le contexte. Les messages plus anciens sont supprimés.

```
Taille de fenetre : 10 messages

Messages 1-5 :   [supprimes]
Messages 6-15 :  [dans le contexte]
Message 16 :     [nouveau message, le message 6 est supprime]
```

**Avantages :** simple à implémenter, usage mémoire prévisible.

**Inconvénients :** perd un contexte ancien important. L'utilisateur pourrait faire référence à quelque chose du message 2, qui n'est plus dans la fenêtre.

**Idéal pour :** les agents conversationnels informels, où le contexte récent compte le plus.

### 2. La synthèse (summarization)

**Comment ça marche :** résumer périodiquement les tours de conversation plus anciens et les remplacer par le résumé. Le résumé occupe beaucoup moins d'espace que les messages originaux.

```
Avant synthese :
  [20 messages detailles sur la planification d'un voyage]  -> 4 000 tokens

Apres synthese :
  [Resume : "L'utilisateur planifie un voyage de 5 jours a Tokyo
   avec un budget de 3 000 $. Il prefere les hotels-boutiques et
   veut visiter des temples et gouter la cuisine locale. Les vols
   sont reserves pour le 15-20 mars."]  -> 200 tokens
```

**Avantages :** préserve l'information clé tout en économisant de l'espace.

**Inconvénients :** la synthèse peut perdre des nuances. L'agent qui effectue le résumé pourrait manquer quelque chose d'important.

**Idéal pour :** les sessions de longue durée où le contexte historique compte.

### 3. La troncature basée sur les tokens

**Comment ça marche :** définir un budget de tokens pour chaque section du contexte (instructions système, historique de conversation, résultats d'outils) et tronquer quand une section dépasse son budget.

```
Budget total : 32 000 tokens
  Instructions systeme :      4 000 tokens (fixe)
  Definitions d'outils :      2 000 tokens (fixe)
  Historique de conversation : 20 000 tokens (glissant)
  Tour actuel :                6 000 tokens (reserve)
```

**Avantages :** contrôle fin de l'allocation du contexte. Garantit qu'un espace est toujours disponible pour la tâche en cours.

**Inconvénients :** nécessite un réglage minutieux. Des limites strictes peuvent couper un message en plein milieu.

**Idéal pour :** les agents en production où vous avez besoin de coûts et de latence prévisibles.

### 4. La sélection basée sur l'importance

**Comment ça marche :** noter chaque élément de contexte selon sa pertinence pour la tâche actuelle et ne garder que les éléments les plus pertinents.

```
Tache actuelle : "Reserver un hotel a Tokyo"

Haute pertinence (garder) :
  - Destination de l'utilisateur : Tokyo
  - Dates de l'utilisateur : 15-20 mars
  - Budget de l'utilisateur : 3 000 $
  - Preferences hotelieres de l'utilisateur : boutique

Faible pertinence (supprimer) :
  - Discussion sur les recommandations de restaurants
  - Conversation anterieure sur les options de vol (deja reservees)
```

**Avantages :** maximise le rapport signal/bruit dans le contexte.

**Inconvénients :** le score de pertinence est imparfait. L'agent pourrait supprimer quelque chose qui s'avère important plus tard.

**Idéal pour :** les agents complexes avec de nombreux types de contexte en concurrence pour l'espace disponible.

### 5. Externaliser et récupérer

**Comment ça marche :** stocker l'information dans une mémoire externe (une base de données, une base vectorielle, ou un graphe de connaissances) et ne la récupérer que quand c'est nécessaire.

```
Au lieu de tout garder dans le contexte :

  Preferences utilisateur -> stockees dans la base de profils utilisateur
  Conversations passees -> stockees dans les archives de conversation
  Documents de reference -> stockes dans une base de donnees vectorielle

Quand c'est necessaire :
  L'agent interroge la base pertinente et ne charge que les
  elements dont il a besoin dans le contexte actuel.
```

**Avantages :** stockage pratiquement illimité. Seule l'information pertinente entre dans le contexte.

**Inconvénients :** ajoute de la latence pour la récupération. La qualité de la récupération dépend du système de recherche.

**Idéal pour :** les agents qui doivent accéder à de grandes quantités d'information, mais n'en utilisent qu'une petite fraction à la fois.

### Comparaison des stratégies

| Stratégie | Économie de contexte | Risque de perte d'information | Complexité | Impact sur la latence |
|----------|----------------|----------------------|------------|---------------|
| Sliding window | Élevée | Élevé | Faible | Aucun |
| Synthèse | Élevée | Moyen | Moyenne | Un peu (étape de synthèse) |
| Troncature par tokens | Moyenne | Moyen | Moyenne | Aucun |
| Sélection par importance | Élevée | Moyen | Élevée | Un peu (étape de notation) |
| Externaliser + récupérer | Très élevée | Faible | Élevée | Plus élevé (étape de récupération) |

---

## Options de stockage de la mémoire

Quand vous externalisez la mémoire, vous avez besoin d'un endroit où la mettre. Voici les principales options.

### Les bases de données vectorielles

**Ce qu'elles font :** stocker l'information sous forme de vecteurs mathématiques (embeddings, plongements vectoriels) et la récupérer par similarité.

**Comment ça marche :**
1. Convertir le texte en vecteur à l'aide d'un modèle d'embedding
2. Stocker le vecteur avec le texte original
3. Lors d'une recherche, convertir la requête en vecteur
4. Trouver les vecteurs stockés les plus similaires au vecteur de la requête

**Idéal pour :** trouver du contenu sémantiquement similaire. « De quoi avons-nous discuté à propos du déploiement ? » correspondra à des conversations passées sur le déploiement, même si elles utilisaient des mots différents.

**Exemples :** Vertex AI Vector Search, Pinecone, Weaviate, ChromaDB

**Compromis :**
- Excellentes pour la recherche par similarité sémantique
- Moins efficaces pour la correspondance exacte ou les requêtes structurées
- La qualité des embeddings affecte la qualité de la récupération

### Les graphes de connaissances

**Ce qu'ils font :** stocker l'information sous forme d'entités et de relations dans une structure de graphe.

**Comment ça marche :**
```
[Utilisateur : Alex] --travaille_sur--> [Projet : API Paiements]
[Projet : API Paiements] --deploye_sur--> [Plateforme : Cloud Run]
[Utilisateur : Alex] --prefere--> [Theme : Mode sombre]
```

**Idéal pour :** représenter des relations structurées entre entités. « Qui travaille sur l'API Paiements ? » ou « Sur quelle plateforme l'API Paiements est-elle déployée ? »

**Exemples :** Neo4j, Amazon Neptune, le Knowledge Graph de Google Cloud

**Compromis :**
- Excellents pour les requêtes de relations
- Nécessitent une conception et une maintenance de schéma
- Moins naturels pour le texte non structuré

### Les approches hybrides

En pratique, de nombreux systèmes combinent les deux :

- **Base de données vectorielle** pour la mémoire non structurée (conversations, documents, notes)
- **Graphe de connaissances** pour la mémoire structurée (relations, faits, préférences)
- **Stockage clé-valeur** pour l'état de session et les recherches rapides

```
"Qu'est-ce qu'Alex prefere ?"
  -> Stockage cle-valeur : {theme: "sombre", langage: "TypeScript"}

"De quoi Alex et moi avons-nous discute a propos des deploiements ?"
  -> Recherche vectorielle : [conversations passees similaires sur les deploiements]

"Quels services l'equipe d'Alex possede-t-elle ?"
  -> Graphe de connaissances : Alex -> equipe -> services -> dependances
```

### Choisir une approche de stockage

| Besoin | Meilleure approche |
|------|--------------|
| Recherche sémantique dans des conversations | Base de données vectorielle |
| Préférences et réglages utilisateur | Stockage clé-valeur |
| Relations entre entités | Graphe de connaissances |
| Historique de session récent | Stockage en mémoire / de session |
| Récupération de documents | Base de données vectorielle + filtres de métadonnées |
| Requêtes complexes à plusieurs sauts | Graphe de connaissances |

---

## Pour tout assembler

Voici comment la mémoire et le contexte s'intègrent dans une architecture d'agent :

```
                         +-------------------+
                         |  Message utilisateur |
                         +--------+----------+
                                  |
                                  v
                    +-------------+-------------+
                    |  Couche de Context        |
                    |  Engineering              |
                    |                           |
                    |  1. Charger le prompt     |
                    |     systeme               |
                    |  2. Recuperer la memoire  |
                    |     pertinente            |
                    |  3. Charger l'historique  |
                    |     de session (elague/   |
                    |     resume)               |
                    |  4. Ajouter les           |
                    |     definitions d'outils  |
                    |  5. Inclure le message    |
                    |     actuel                |
                    +-------------+-------------+
                                  |
                          Contexte assemble
                                  |
                                  v
                         +--------+----------+
                         |       LLM         |
                         +--------+----------+
                                  |
                          Reponse de l'agent
                                  |
                                  v
                    +-------------+-------------+
                    |  Couche de mise a jour    |
                    |  de la memoire            |
                    |                           |
                    |  1. Sauvegarder dans      |
                    |     la session            |
                    |  2. Extraire les faits    |
                    |     cles pour la memoire  |
                    |     a long terme          |
                    |  3. Mettre a jour le      |
                    |     profil utilisateur    |
                    +---------------------------+
```

La couche de context engineering assemble le bon contexte *avant* que le LLM ne le voie. La couche de mise à jour de la mémoire extrait l'information importante *après* que le LLM a répondu. Ensemble, elles donnent à l'agent la capacité de se souvenir et de rester concentré.

---

## Points clés à retenir

1. **La context window est la mémoire de travail de l'agent.** Tout ce que l'agent prend en compte doit tenir dans cet espace. Bien la gérer est essentiel.

2. **Le context engineering va au-delà du prompt engineering.** Les agents ont besoin d'un assemblage dynamique du contexte, pas seulement de prompts statiques. Ce qui entre dans la context window devrait varier selon la situation.

3. **Les agents ont besoin de plusieurs types de mémoire.** Court terme pour la conversation en cours, long terme pour la persistance entre les sessions, procédurale pour savoir comment faire les choses, et déclarative pour les faits.

4. **Les sessions offrent une continuité conversationnelle.** Elles suivent l'historique et l'état d'une interaction, donnant aux agents la capacité de maintenir des conversations cohérentes.

5. **La mémoire et le RAG servent des objectifs différents.** La mémoire est personnelle et spécifique à l'interaction. Le RAG concerne l'accès à des connaissances générales. La plupart des agents ont besoin des deux.

6. **Le context rot est un problème réel.** À mesure que le contexte grandit, l'information importante s'enfouit. Une gestion active via la synthèse, l'élagage, et le stockage externe maintient l'agent efficace.

7. **Choisissez votre stockage en fonction de vos schémas d'accès.** Bases de données vectorielles pour la recherche sémantique, graphes de connaissances pour les relations, stockages clé-valeur pour les recherches rapides. Les approches hybrides fonctionnent souvent le mieux.

---

## Pour aller plus loin

- [Documentation des sessions ADK](https://google.github.io/adk-docs/sessions/)
- [Vertex AI Agent Engine - Gérer les sessions](https://cloud.google.com/agent-builder/agent-engine/sessions/overview)
- [Vertex AI Vector Search](https://cloud.google.com/vertex-ai/docs/vector-search/overview)
- [Codelabs Google Cloud AI](https://codelabs.developers.google.com/?cat=AI)

---

**Leçon suivante :** [Planification et raisonnement - comment les agents abordent les tâches complexes](../06-planning-and-reasoning/README.md)
