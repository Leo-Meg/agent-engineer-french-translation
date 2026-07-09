# Leçon 1 : Qu'est-ce qu'un agent IA ?

## Introduction

Vous avez probablement déjà utilisé ChatGPT, Gemini ou Claude pour répondre à une question ou écrire du code. Vous avez tapé quelque chose, obtenu une réponse, et êtes passé à autre chose. C'est un modèle de langage qui fait son travail : prédire un texte utile à partir de ce que vous lui fournissez en entrée.

Un agent IA, c'est différent. Un agent peut **penser**, **agir** et **se souvenir**. Il ne se contente pas de répondre à votre question : il détermine les étapes à suivre, utilise des outils pour les mettre en œuvre, et ajuste son approche en fonction de ce qui se passe en cours de route.

Voyez les choses ainsi : si vous embauchiez un nouvel ingénieur en ne le laissant que parler, sans jamais toucher à un clavier, ouvrir un navigateur ou lire de la documentation, il serait limité. C'est un LLM (grand modèle de langage) livré à lui-même. Donnez maintenant à cet ingénieur l'accès à votre codebase (base de code), à un terminal, à la documentation de votre entreprise, et la possibilité de poser des questions pour clarifier les choses. C'est un agent.

Cette leçon couvre ce que sont les agents IA, en quoi ils diffèrent des simples modèles de langage, quels composants les font fonctionner, et quand vous devriez (ou ne devriez pas) les utiliser.

---

## Qu'est-ce qu'un agent IA, en termes simples ?

Un agent IA est un système logiciel qui utilise un modèle de langage comme moteur de raisonnement central, combiné à la capacité d'entreprendre des actions dans le monde réel. Ces actions peuvent inclure :

- Effectuer une recherche sur le web
- Interroger une base de données
- Appeler une API
- Lire ou écrire des fichiers
- Envoyer un e-mail
- Exécuter du code

La distinction clé, c'est l'**autonomie**. Un LLM simple répond à un seul prompt (une invite). Un agent reçoit un objectif, puis décide de façon autonome des étapes à suivre, les exécute, observe les résultats, et continue jusqu'à ce que l'objectif soit atteint (ou jusqu'à ce qu'il détermine que l'objectif est inatteignable).

### L'analogie du nouvel employé

Imaginez que vous embauchiez un nouvel ingénieur logiciel. Le premier jour, vous ne vous attendriez pas à ce qu'il sache déjà tout. Mais vous vous attendriez à ce qu'il :

1. **Lise la documentation** pour comprendre la codebase
2. **Utilise des outils** comme un IDE, un terminal et un navigateur
3. **Pose des questions** quand quelque chose n'est pas clair
4. **Décompose les tâches** en étapes plus petites
5. **Vérifie son travail** avant de dire qu'il a terminé
6. **Apprenne de ses erreurs** et ajuste son approche

Un agent IA fonctionne de la même manière. Il dispose d'une base de connaissances (le modèle de langage), d'un accès à des outils, et d'une couche d'orchestration qui gère la boucle penser-agir-observer.

---

## LLM contre agent : quelle est la différence ?

C'est la distinction la plus importante à intégrer dès le départ.

| Aspect | LLM (seul) | Agent IA |
|---|---|---|
| **Ce qu'il fait** | Génère du texte à partir d'un prompt | Poursuit un objectif à travers plusieurs étapes |
| **Interaction** | Un seul tour (ou chat multi-tours) | Boucle autonome de réflexion et d'action |
| **Outils** | Aucun - texte en entrée, texte en sortie | Peut appeler des fonctions, des API, effectuer des recherches, etc. |
| **Mémoire** | Limitée à la context window (fenêtre de contexte) | Peut faire persister l'information à travers les étapes |
| **Prise de décision** | Répond à ce que vous demandez | Décide seul de la prochaine action |
| **Gestion des erreurs** | Vous donne une réponse (juste ou fausse) | Peut observer les erreurs et retenter avec une nouvelle approche |

Un modèle mental utile :

- **LLM** = Cerveau
- **Agent** = Cerveau + Mains + Mémoire

Le cerveau (le LLM) effectue le raisonnement. Les mains (les outils) lui permettent d'agir. La mémoire (la gestion de l'état) lui permet de garder la trace de ce qui s'est passé et de ce qui reste à faire.

### Un exemple concret

**LLM seul :** Vous demandez « Quel est le cours actuel de l'action GOOG ? ». Le modèle pourrait répondre « D'après mes dernières données d'entraînement, il était autour de 140 $ » - une information potentiellement vieille de plusieurs mois.

**Agent :** Vous posez la même question. L'agent se dit : « J'ai besoin de données boursières actuelles, je devrais utiliser une API financière. » Il appelle un outil de cours boursier, obtient le prix en temps réel, et renvoie une réponse exacte. Si l'appel à l'API échoue, il peut essayer une autre source de données.

Cette boucle - penser, agir, observer, recommencer - est ce qui fait d'un agent un agent.

---

## Les composants fondamentaux d'un agent IA

Tout système d'agent, quel que soit le framework (cadre applicatif) utilisé, comporte trois composants fondamentaux :

### 1. le modèle (le cerveau)

C'est le modèle de langage au centre de l'agent. Il se charge de :

- **Comprendre** l'objectif de l'utilisateur
- **Raisonner** sur les étapes à suivre
- **Décider** quel outil utiliser (et avec quels paramètres)
- **Interpréter** les résultats des appels d'outils
- **Générer** la réponse finale

Le choix du modèle a de l'importance. Les tâches plus difficiles (raisonnement multi-étapes, génération de code complexe, prise de décision nuancée) tirent parti de modèles de pointe comme Gemini ou Opus. Les tâches plus simples (classification, extraction, questions-réponses directes) peuvent utiliser des modèles plus légers comme Gemini Flash, pour réduire le coût et la latence.

### 2. les outils (les mains)

Les outils sont ce qui permet à un agent d'interagir avec le monde au-delà de la simple génération de texte. Sans outils, un agent n'est qu'un chatbot. Avec des outils, il peut :

- **Récupérer de l'information** : effectuer une recherche sur le web, interroger une base de données, lire un fichier
- **Entreprendre des actions** : envoyer un e-mail, créer un ticket, déployer du code
- **Calculer** : effectuer des calculs, exécuter du code, transformer des données

Les outils sont généralement définis comme des fonctions dotées de noms clairs, de descriptions et de schémas de paramètres. Le modèle décide quand et comment les appeler. Nous approfondirons les outils dans la leçon 3.

### 3. la couche d'orchestration (la boucle de contrôle)

C'est la colle qui relie le modèle et les outils en un système fonctionnel. La couche d'orchestration gère :

- **La boucle de l'agent** : Penser -> Agir -> Observer -> Recommencer
- **La gestion de l'état** : ce qui s'est passé jusqu'ici, quel contexte le modèle a besoin
- **La gestion des erreurs** : que faire quand un appel d'outil échoue
- **Les conditions d'arrêt** : quand cesser de boucler et renvoyer un résultat
- **Les guardrails (garde-fous)** : vérifications de sécurité, validation des sorties, limites de périmètre

Le pattern (patron) d'orchestration le plus simple ressemble à ceci :

```
1. Recevoir l'objectif de l'utilisateur
2. Envoyer l'objectif et les outils disponibles au modele
3. Le modele renvoie soit :
   a. Une reponse finale -> Retour a l'utilisateur
   b. Un appel d'outil -> Executer l'outil, ajouter le resultat au contexte, retour a l'etape 2
```

C'est ce qu'on appelle souvent une **boucle ReAct** (Reasoning + Acting, raisonnement + action). Des patterns plus sophistiqués existent - nous les explorerons dans les leçons suivantes.

### Comment les composants fonctionnent ensemble

```
Objectif de l'utilisateur
    |
    v
+-------------------+
| Couche             |
| d'orchestration    |
|                    |
|  +-------------+   |
|  |   Modele    |   |    "J'ai besoin de chercher X"
|  |  (Cerveau)  |---+--->  Appel d'outil
|  +-------------+   |         |
|        ^           |         v
|        |           |  +-------------+
|        +-----------+--+   Outils    |
|   Resultats d'outil|  |  (Mains)    |
|                     |  +-------------+
+---------------------+
    |
    v
Reponse finale
```

---

## Une taxonomie des systèmes d'agents

Tous les agents ne se valent pas. Il est utile de penser les systèmes d'agents comme un spectre d'autonomie et de capacité, du niveau 0 au niveau 4.

### Niveau 0 : raisonnement de base (LLM simple)

**Ce que c'est :** un modèle de langage qui répond à des questions sans outils ni mémoire.

**Exemple :** vous demandez à Gemini « Explique-moi le théorème CAP » et il vous fournit une explication claire à partir de ses données d'entraînement.

**Capacités :**
- Génération et compréhension de texte
- Conversation à un tour ou à plusieurs tours
- Aucun accès à des données externes
- Aucune capacité à entreprendre des actions

**Quand cela fonctionne bien :** questions de culture générale, écriture créative, brainstorming, synthèse d'un texte fourni.

### Niveau 1 : résolveur de problèmes connecté (agent utilisant des outils)

**Ce que c'est :** un modèle capable d'appeler des outils pour récupérer de l'information ou effectuer des actions simples. C'est ici que l'on franchit la frontière entre « chatbot » et « agent ».

**Exemple :** un bot de support client capable de consulter le statut d'une commande en appelant votre API de commandes, ou un assistant de codage capable de rechercher dans la documentation.

**Capacités :**
- Tout ce qui est inclus dans le niveau 0
- Function calling (appel de fonction, les outils)
- RAG (Retrieval-Augmented Generation, génération augmentée par récupération) pour ancrer les réponses dans des données réelles
- Réalisation de tâches simples, à une étape ou à quelques étapes

**Quand cela fonctionne bien :** tâches nécessitant des données à jour, des intégrations API, des workflows (flux de travail) simples comportant peu d'étapes.

### Niveau 2 : agent stratégique (autonome avec contexte)

**Ce que c'est :** un agent capable de planifier des approches à plusieurs étapes, de maintenir le contexte sur une session plus longue, et d'adapter sa stratégie en fonction des résultats intermédiaires.

**Exemple :** un agent de recherche qui reçoit une question comme « Compare les 3 principaux fournisseurs cloud sur la tarification serverless », puis recherche des pages de tarification, en extrait les données, construit un tableau comparatif, et synthétise les résultats.

**Capacités :**
- Tout ce qui est inclus dans le niveau 1
- Planification et exécution multi-étapes
- Replanification dynamique en cas de changement
- Mémoire de travail à travers les étapes
- Auto-évaluation (« Ce résultat est-il assez bon ? »)

**Quand cela fonctionne bien :** tâches de recherche, dépannage complexe, workflows multi-étapes où le chemin dépend des résultats intermédiaires.

### Niveau 3 : système multi-agent collaboratif

**Ce que c'est :** plusieurs agents spécialisés travaillant ensemble, chacun gérant un aspect différent d'une tâche plus vaste. Un agent peut coordonner les autres.

**Exemple :** un système de développement logiciel où un agent écrit du code, un autre écrit les tests, un troisième relit le code, et un agent orchestrateur gère le workflow.

**Capacités :**
- Tout ce qui est inclus dans le niveau 2
- Communication agent-à-agent
- Rôles spécialisés et délégation
- Exécution parallèle des sous-tâches
- Mécanismes de consensus ou de vote pour garantir la qualité

**Quand cela fonctionne bien :** projets complexes qui bénéficient de la spécialisation, tâches nécessitant plusieurs perspectives ou des points de contrôle qualité.

### Niveau 4 : agent auto-évolutif

**Ce que c'est :** un agent capable de réfléchir sur sa propre performance, d'apprendre de ses exécutions passées, de mettre à jour ses stratégies, et de s'améliorer avec le temps sans intervention manuelle.

**Exemple :** un agent de déploiement qui suit quelles stratégies de rollback (retour arrière) ont historiquement le mieux fonctionné, et ajuste son approche pour les déploiements futurs.

**Capacités :**
- Tout ce qui est inclus dans le niveau 3
- Mémoire à long terme et apprentissage
- Optimisation de la stratégie à partir des résultats passés
- Auto-modification des prompts ou de la sélection d'outils
- Suivi de la performance et auto-correction

**Quand cela fonctionne bien :** tâches récurrentes où des patterns émergent avec le temps, systèmes qui bénéficient d'une amélioration continue.

### Tableau récapitulatif

| Niveau | Nom | Caractéristique clé | Exemple |
|---|---|---|---|
| 0 | Raisonnement de base | Texte en entrée, texte en sortie | Chatbot, questions-réponses |
| 1 | Résolveur de problèmes connecté | Utilisation d'outils | Bot de consultation de commandes |
| 2 | Agent stratégique | Planification multi-étapes | Assistant de recherche |
| 3 | Multi-agent collaboratif | Coordination d'agents | Simulation d'équipe de développement |
| 4 | Auto-évolutif | Apprentissage par l'expérience | Agent d'opérations adaptatif |

Aujourd'hui, la plupart des systèmes d'agents en production fonctionnent au niveau 1 ou 2. Les niveaux 3 et 4 sont des domaines de recherche actifs et deviennent de plus en plus concrets, mais ils ajoutent une complexité significative. Commencez simple, et ne montez de niveau que si vous avez une bonne raison de le faire.

---

## Quand utiliser des agents, et quand un simple prompt suffit

Les agents ajoutent de la puissance, mais aussi de la complexité, du coût et de la latence. Tous les problèmes n'ont pas besoin d'un agent. Voici un guide pratique.

### Utilisez un simple prompt quand :

- La tâche peut être accomplie en une seule étape
- Aucune donnée externe ni action n'est nécessaire
- La réponse existe déjà dans les données d'entraînement du modèle
- Une faible latence est critique (les agents ajoutent plusieurs allers-retours)
- Le coût de plusieurs appels au modèle n'est pas justifié

**Exemples :**
- « Résume ce paragraphe »
- « Convertis ce JSON en dataclass Python »
- « Écris une regex qui correspond à des adresses e-mail »
- « Explique la différence entre TCP et UDP »

### Utilisez un agent quand :

- La tâche nécessite plusieurs étapes qui dépendent les unes des autres
- Des données ou des outils externes sont nécessaires (API, bases de données, recherche)
- La tâche nécessite une information en temps réel ou à jour
- L'approche peut avoir besoin de changer en fonction des résultats intermédiaires
- La tâche implique d'entreprendre des actions (pas seulement de générer du texte)

**Exemples :**
- « Trouve les trois bugs les plus récents dans notre issue tracker et rédige un résumé pour le stand-up d'équipe »
- « Consulte la commande du client, vérifie le statut de livraison, et envoie-lui un e-mail de mise à jour »
- « Recherche la tarification des concurrents et construis un tableur comparatif »
- « Relis cette pull request, lance les tests, et suggère des améliorations »

### L'arbre de décision

```
La tache necessite-t-elle des donnees ou des actions externes ?
  |
  +-- Non --> Le modele peut-il repondre a partir de ses donnees d'entrainement ?
  |            |
  |            +-- Oui --> Utilisez un simple prompt
  |            +-- Non --> Envisagez d'abord le RAG (recuperation), puis un agent
  |
  +-- Oui --> S'agit-il d'un seul appel d'outil ?
               |
               +-- Oui --> Une simple configuration de function calling peut suffire
               +-- Non --> Utilisez un agent avec orchestration
```

### Considérations de coût et de latence

Chaque étape d'une boucle d'agent implique un appel au modèle. Un workflow d'agent à 5 étapes signifie 5 appels au modèle ou plus, en plus du temps d'exécution des outils. Cela s'additionne :

- **Latence** : chaque appel au modèle prend 1 à 10 secondes selon le modèle et la taille du prompt. Un agent à 5 étapes peut prendre 15 à 30 secondes.
- **Coût** : chaque appel au modèle consomme des tokens (jetons). Les workflows d'agents peuvent utiliser 10 à 50 fois plus de tokens qu'un simple prompt.
- **Fiabilité** : plus il y a d'étapes, plus il y a de risques d'erreurs ou d'hallucinations.

Le principe d'ingénierie est le même que partout ailleurs : utilisez l'approche la plus simple qui permette d'accomplir la tâche.

---

## Exemples concrets

### Agent de support client

**Objectif :** traiter les demandes des clients de bout en bout.

**Comment ça marche :**
1. Le client écrit : « Où est ma commande #12345 ? »
2. L'agent appelle l'outil de consultation de commande avec l'identifiant de commande
3. Il obtient le statut : « Expédiée, numéro de suivi XYZ, livraison estimée le 20 mars »
4. L'agent formule une réponse conviviale avec le lien de suivi
5. Si le client demande à changer l'adresse de livraison, l'agent appelle l'outil de mise à jour d'adresse

**Niveau :** 1-2 (utilisation d'outils avec une logique multi-étapes limitée)

### Agent assistant de code

**Objectif :** aider les développeurs à écrire, déboguer et améliorer du code.

**Comment ça marche :**
1. Le développeur demande : « Pourquoi cette fonction renvoie-t-elle null ? »
2. L'agent lit les fichiers source pertinents
3. Il recherche les tests associés
4. Il identifie le bug (vérification de null manquante à la ligne 42)
5. Il propose un correctif avec du code
6. Il peut éventuellement lancer les tests pour vérifier que le correctif fonctionne

**Niveau :** 2 (raisonnement multi-étapes avec utilisation d'outils)

### Agent de recherche

**Objectif :** rassembler et synthétiser de l'information provenant de sources multiples.

**Comment ça marche :**
1. L'utilisateur demande : « Quels sont les avantages et les inconvénients du server-side rendering (rendu côté serveur) en 2026 ? »
2. L'agent recherche des articles récents et des benchmarks (analyses comparatives)
3. Il lit et extrait les points clés de plusieurs sources
4. Il recoupe les affirmations et vérifie leur cohérence
5. Il produit une synthèse structurée avec des citations

**Niveau :** 2 (rechercher, lire, synthétiser à travers plusieurs étapes)

### Agent de réponse aux incidents DevOps

**Objectif :** aider à diagnostiquer et résoudre les incidents de production.

**Comment ça marche :**
1. Une alerte se déclenche : « Pic de latence API sur service-auth »
2. L'agent interroge les tableaux de bord de monitoring (surveillance) des 30 dernières minutes
3. Il vérifie les déploiements récents à la recherche de changements
4. Il examine les logs à la recherche de motifs d'erreurs
5. Il corrèle ses observations : « Le pic de latence a commencé 5 minutes après le déploiement #789, qui a modifié le TTL (durée de vie) du cache de tokens d'authentification »
6. Il suggère un rollback et rédige un rapport d'incident

**Niveau :** 2-3 (investigation multi-étapes, pouvant coordonner avec d'autres agents)

---

## ELI5 (« explique-moi comme si j'avais 5 ans ») : qu'est-ce qu'un agent IA ?

### Pensez à un agent comme à un stagiaire vraiment compétent

Imaginez que vous ayez un tout nouveau stagiaire pour son premier jour. Il est intelligent - il a été major de sa promotion - mais il n'a encore jamais vu votre codebase.

**Un LLM livré à lui-même, c'est comme ce stagiaire assis dans une pièce sans ordinateur.** Vous pouvez lui poser des questions et il vous donnera des réponses réfléchies, fondées sur ce qu'il a appris à l'école. Mais il ne peut rien consulter, il ne peut exécuter aucun code, et il ne peut envoyer aucun e-mail. Tout ce qu'il peut faire, c'est parler.

**Un agent, c'est comme ce même stagiaire avec un poste de travail complet.** Il a un ordinateur portable, un accès à vos outils internes, un navigateur, et le Slack de votre entreprise. Maintenant, quand vous lui posez une question, il peut :

- Se renseigner s'il ne connaît pas la réponse
- Essayer d'exécuter du code pour tester ses idées
- Vérifier la documentation pour s'assurer qu'il a raison
- Demander de l'aide à un collègue (un autre agent)
- Revenir vers vous avec une réponse vérifiée

Le stagiaire fait encore des erreurs parfois - après tout, il est nouveau. Mais il peut détecter la plupart de ses erreurs, car il peut vérifier son propre travail. Et s'il est bloqué, il sait qu'il doit demander de l'aide plutôt que de deviner.

**L'idée clé :** le cerveau du stagiaire n'a pas changé. Ce qui a changé, c'est à quoi il a accès et comment il aborde le travail. C'est exactement la différence entre un LLM et un agent. Même cerveau, plus de capacités, meilleur processus.

---

## La place de Google Cloud dans tout ça

Google Cloud fournit l'infrastructure nécessaire pour construire et déployer des agents à travers plusieurs services :

- **Vertex AI Agent Engine** - une plateforme managée pour construire, déployer et gérer des agents IA en production. Elle prend en charge l'orchestration, la gestion des outils, l'état de session et la mise à l'échelle, afin que vous puissiez vous concentrer sur la logique de l'agent plutôt que sur l'infrastructure.

- **Les modèles Gemini** - les modèles de langage qui servent de « cerveau » à vos agents, disponibles en différentes tailles selon les cas d'usage.

- **Agent Development Kit (ADK)** - une boîte à outils open source, axée sur le code, pour construire des agents, avec des fonctionnalités comme l'orchestration multi-agents, un support d'outils intégré, et un déploiement facile vers Agent Engine.

Nous utiliserons ces outils tout au long de la formation. Pour l'instant, sachez simplement qu'ils existent.

> **Pour en savoir plus :** [Présentation de Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview)

---

## Points clés à retenir

1. **Un agent IA est un système qui utilise un modèle de langage pour raisonner, des outils pour agir, et une couche d'orchestration pour gérer la boucle entre la réflexion et l'action.**

2. **LLM = cerveau. Agent = cerveau + mains + mémoire.** Le modèle assure le raisonnement. Les outils assurent l'action. La couche d'orchestration assure le flux de contrôle.

3. **Les agents existent sur un spectre**, des simples assistants utilisant des outils (niveau 1) aux systèmes auto-évolutifs (niveau 4). Commencez au niveau le plus bas qui résout votre problème.

4. **Tout ne nécessite pas un agent.** Si un simple prompt suffit à accomplir la tâche, utilisez un simple prompt. N'ajoutez des capacités d'agent que lorsque la tâche nécessite véritablement des outils, un raisonnement multi-étapes, ou des actions dans le monde réel.

5. **La boucle centrale est simple :** recevoir l'objectif -> réfléchir à quoi faire -> utiliser un outil -> observer le résultat -> recommencer jusqu'à ce que ce soit terminé.

---

## Et maintenant ?

Dans la prochaine leçon, nous regarderons sous le capot le « cerveau » de l'agent : le modèle de langage. Vous apprendrez comment les LLM traitent l'information, comment différentes stratégies de raisonnement affectent la performance des agents, et comment choisir le bon modèle pour la tâche.

[Suivant : Leçon 2 - Comment pensent les agents : les LLM comme moteur de raisonnement -->](../02-how-agents-think/README.md)
