# Leçon 17 : agent skills - des connaissances réutilisables pour les agents

## Introduction

Dans la [leçon 3](../03-tools-giving-agents-hands/README.md), nous avons appris ce que sont les outils - des fonctions qui permettent aux agents d'entreprendre des actions comme appeler des API, interroger des bases de données, et exécuter du code. Les outils, c'est **faire des choses**.

Les skills, c'est **savoir des choses**. Une skill empaquette une expertise métier - des instructions, des bonnes pratiques, des cadres de décision, et des documents de référence - en une unité modulaire qu'un agent peut découvrir et utiliser en cas de besoin.

Pensez à la différence entre donner à quelqu'un une clé à molette (un outil) et lui donner un manuel de réparation (une skill). La clé lui permet de tourner des boulons. Le manuel lui indique quels boulons tourner, dans quel ordre, et à quoi faire attention.

### ELI5 : pensez aux skills comme à des fiches recettes dans une cuisine

Une cuisine professionnelle a des outils (couteaux, poêles, fours) et des fiches recettes. Un nouveau chef peut prendre un couteau sans instructions. Mais pour préparer un plat spécifique, il a besoin de la fiche recette - elle lui indique quels outils utiliser, dans quel ordre, à quelle température, et à quoi le résultat devrait ressembler.

Les agent skills fonctionnent de la même manière. Ce sont les fiches recettes qui indiquent à un agent comment aborder un type de tâche spécifique, quels outils utiliser, et à quoi ressemble une bonne sortie.

> **Point clé à retenir :** les skills encodent une expertise métier sous forme de paquets portables et réutilisables. Les outils permettent aux agents d'agir. Les skills indiquent aux agents comment et quand agir.

---

## Pourquoi les skills existent

Considérez ce scénario : votre équipe a un agent qui aide aux relectures de code. Vous voulez qu'il suive la checklist de relecture spécifique de votre équipe, signale les patterns courants qui vous intéressent, et formate son retour d'une manière particulière.

Vous pourriez mettre tout cela dans le prompt système de l'agent. Mais les prompts système s'encombrent vite. Si vous ajoutez des instructions de relecture, des procédures de déploiement, des standards de documentation, et des conventions de test, le tout dans un seul prompt, vous vous retrouvez avec une context window surchargée et un agent médiocre en tout.

Les skills résolvent cela en vous permettant de :

1. **Empaqueter l'expertise séparément** - chaque skill est son propre fichier, ciblé sur un domaine
2. **Charger à la demande** - les skills ne sont chargées que quand elles sont pertinentes, économisant de l'espace de context window
3. **Partager entre équipes** - une skill bien écrite peut être réutilisée à travers les projets et les agents
4. **Itérer indépendamment** - mettre à jour une skill sans changer la configuration centrale de l'agent

### Le problème de la context window

C'est la motivation technique clé. Chaque token dans la context window a un coût - à la fois en argent et en attention. Si vous chargez 50 000 tokens d'instructions au démarrage, l'agent paie ce coût à chaque tour, même quand la plupart de ces instructions ne sont pas pertinentes.

Les skills utilisent la divulgation progressive (progressive disclosure) pour garder le coût bas :

- Au démarrage, ne charger que les noms et descriptions des skills (~100 tokens chacune)
- Quand une skill est déclenchée, charger ses instructions complètes
- Ne charger les documents de référence que quand les instructions en ont explicitement besoin

Une analyse a montré que cette approche réduit un workflow de 150 000 tokens à environ 2 000 tokens au démarrage.

---

## La spécification des skills

Les skills suivent une spécification ouverte maintenue sur [agentskills.io](https://agentskills.io/specification). Le format est simple :

### La structure de dossiers

```
my-skill/
  SKILL.md          # Requis : metadonnees + instructions
  references/       # Optionnel : documentation additionnelle
  assets/           # Optionnel : modeles, schemas, fichiers de donnees
  scripts/          # Optionnel : code executable
```

Le seul fichier requis est `SKILL.md`. Tout le reste est optionnel.

### Le format de SKILL.md

Un fichier SKILL.md a deux parties : un frontmatter YAML pour les métadonnées, et du contenu Markdown pour les instructions.

```markdown
---
name: code-review
description: >
  Reviews pull requests following team standards. Checks for
  security issues, test coverage, naming conventions, and
  documentation. Use when asked to review code or a PR.
---

## Code Review Process

When reviewing code, follow these steps in order:

### 1. Security check
- Look for hardcoded secrets, SQL injection, XSS vulnerabilities
- Check that user input is validated and sanitized
- Verify authentication and authorization on new endpoints

### 2. Test coverage
- New public functions should have tests
- Edge cases should be covered (empty input, null values, errors)
- Check that tests actually assert meaningful behavior

### 3. Naming and structure
- Functions and variables should have descriptive names
- Files should be in the correct directory per project conventions
- No single function should exceed 50 lines

### 4. Documentation
- Public APIs should have docstrings
- Non-obvious logic should have inline comments
- README should be updated if behavior changes

### Output format
Present findings as a list grouped by category (Security, Tests,
Style, Docs). For each finding, include the file path, line number,
severity (high/medium/low), and a suggested fix.
```

### Les champs du frontmatter

| Champ | Requis | Objectif |
|-------|----------|---------|
| `name` | Oui | Identifiant unique, en minuscules avec des tirets (par ex., `code-review`) |
| `description` | Oui | Ce que fait la skill et quand la déclencher (jusqu'à 1024 caractères) |
| `license` | Non | Licence de la skill |
| `compatibility` | Non | Exigences d'environnement (par ex., « nécessite Python 3.10+ ») |
| `metadata` | Non | Paires clé-valeur arbitraires (auteur, version, tags) |

Le champ `description` est critique. C'est la façon principale dont les agents décident s'ils doivent activer une skill. Écrivez-le pour décrire clairement à la fois ce que fait la skill et quand elle devrait être utilisée.

---

## La divulgation progressive - les trois niveaux

Les skills sont conçues pour se charger de façon incrémentale. C'est l'idée architecturale clé qui les rend pratiques :

### Niveau 1 : les métadonnées (toujours chargées)

Au démarrage, l'agent ne charge que le `name` et la `description` du frontmatter de chaque skill installée. Cela coûte environ 100 tokens par skill. Même avec 50 skills installées, le coût de démarrage n'est que d'environ 5 000 tokens.

L'agent utilise ces métadonnées pour décider : « Compte tenu de la tâche actuelle, cette skill est-elle pertinente ? »

### Niveau 2 : les instructions (chargées à l'activation)

Quand l'agent décide qu'une skill est pertinente, il charge le corps complet du SKILL.md. C'est là que vivent les instructions étape par étape, les cadres de décision, et les exemples. La recommandation est de garder cela sous 5 000 tokens.

### Niveau 3 : les ressources (chargées à la demande)

Les fichiers dans les dossiers `references/`, `assets/`, et `scripts/` ne sont chargés que quand les instructions du niveau 2 les référencent. Cela peut inclure :

- `references/security-checklist.md` - critères étendus de relecture de sécurité
- `assets/api-schema.json` - spécification d'API pour la validation
- `assets/response-template.md` - modèle pour la sortie formatée
- `scripts/run-linter.sh` - script que l'agent peut exécuter

Cette approche à trois niveaux signifie que vous pouvez écrire des skills très détaillées sans payer le coût de contexte à l'avance.

```
Demarrage :        [N1 : nom + description]      ~100 tokens par skill
                            |
Tache correspond : [N2 : instructions completes] ~2 000-5 000 tokens
                            |
Si necessaire :    [N3 : fichiers de reference]  Variable
```

---

## Skills contre outils contre MCP

Ces trois concepts opèrent à des couches différentes. Comprendre la distinction vous aide à décider lequel utiliser :

| Dimension | Skills | Outils / function calling | MCP |
|-----------|--------|------------------------|-----|
| **Ce que ça fournit** | Connaissances et instructions | Fonctions exécutables | Protocole standardisé pour l'intégration d'outils |
| **Analogie** | Une fiche recette | Un appareil de cuisine | Un standard de prise électrique |
| **Nature** | Guidance en langage naturel | Code qui s'exécute | Couche de communication JSON-RPC |
| **Exécution** | Le LLM interprète les instructions | Appel de fonction déterministe | Protocole pour appeler des outils distants |
| **Latence** | Locale (juste du texte) | Selon la fonction | Aller-retour réseau |
| **Idéal pour** | Encoder de l'expertise, des workflows, des critères de relecture | Entreprendre des actions (appels d'API, opérations sur fichiers, requêtes) | Se connecter à des services externes avec authentification et découverte |
| **Coût de contexte** | Faible (chargement progressif) | Moyen (schéma par outil) | Plus élevé (schémas complets à l'avance) |

### Comment ils fonctionnent ensemble

Dans un agent typique, les trois sont utilisés :

1. **Les skills** indiquent à l'agent comment aborder la tâche et quels outils utiliser
2. **Les outils** (function calling) permettent à l'agent d'exécuter des actions
3. **MCP** fournit une façon standard de se connecter à des serveurs d'outils distants

Exemple : une skill « deploy-to-staging » pourrait inclure des instructions comme :
- Étape 1 : lancer la suite de tests en utilisant l'outil `run_tests`
- Étape 2 : vérifier le statut de l'environnement de staging en utilisant le serveur MCP Kubernetes
- Étape 3 : si les tests passent et que le staging est sain, déployer en utilisant l'outil `deploy`
- Étape 4 : vérifier le déploiement en consultant les endpoints de santé

La skill fournit la logique de workflow. Les outils et les serveurs MCP fournissent la capacité d'exécution.

---

## Écrire de bonnes skills

### Se concentrer sur un domaine

Une skill devrait bien faire une seule chose. Plutôt qu'une skill « développement » qui couvre tout, créez des skills séparées pour la relecture de code, le déploiement, la documentation, et les tests.

### Écrire pour le LLM, pas pour un humain

Les skills sont interprétées par un modèle de langage. Soyez explicite sur :
- **Quand** utiliser la skill (conditions de déclenchement)
- **Quelles** étapes suivre (processus ordonné)
- **Comment** gérer les cas limites (points de décision)
- **À quoi** ressemble une bonne sortie (exemples ou modèles)

### Inclure des points de décision

La véritable expertise inclut de savoir quand dévier du processus standard :

```markdown
### Handling large PRs (>500 lines changed)

If the PR changes more than 500 lines:
- Focus review on the most critical files first (API endpoints, auth, data models)
- Skip cosmetic issues (formatting, naming) unless they affect readability
- Suggest splitting the PR if the changes cover multiple unrelated concerns
```

### Montrer la sortie attendue

Incluez des exemples de ce à quoi devrait ressembler la sortie de la skill :

```markdown
### Example output

**Security - High**
`src/api/auth.py:45` - Password is compared using `==` instead of
`hmac.compare_digest()`. This is vulnerable to timing attacks.
Suggested fix: Replace with `hmac.compare_digest(stored_hash, provided_hash)`
```

### Garder les instructions du niveau 2 sous 5 000 tokens

Si vos instructions deviennent longues, déplacez le matériel de référence détaillé vers des fichiers du niveau 3 dans le dossier `references/` et référencez-les depuis les instructions principales :

```markdown
For the full security checklist, refer to `references/security-checklist.md`.
```

---

## Les skills dans Google ADK

L'Agent Development Kit de Google prend en charge les skills via la classe `SkillToolset`. Voici un aperçu conceptuel de son fonctionnement :

### Les skills basées sur des fichiers (recommandé)

Placez les dossiers de skills dans un dossier `skills/` au sein de votre projet d'agent :

```
my-agent/
  agent.py
  skills/
    code-review/
      SKILL.md
      references/
        security-checklist.md
    deploy/
      SKILL.md
      scripts/
        pre-deploy-check.sh
```

L'agent découvre et charge les skills depuis ce dossier. Seules les métadonnées du niveau 1 sont chargées au démarrage. Les instructions complètes se chargent quand l'agent active la skill.

### Les skills basées sur le code

Pour la création ou la modification dynamique de skills, ADK prend aussi en charge la définition de skills dans le code en utilisant la classe de modèle `Skill`. C'est utile quand le contenu de la skill doit changer selon des conditions d'exécution.

Pour des directives d'implémentation détaillées, consultez la [documentation des skills ADK](https://google.github.io/adk-docs/skills/).

---

## Les skills à travers les plateformes

La spécification Agent Skills a été adoptée par plusieurs plateformes :

| Plateforme | Prise en charge | Détails |
|----------|---------|---------|
| **Claude Code** (Anthropic) | Oui | Skills comme `/slash-commands`, dépôt [anthropics/skills](https://github.com/anthropics/skills) |
| **Google ADK** | Oui | Classe `SkillToolset`, basée sur des fichiers et sur le code |
| **GitHub Copilot** | Oui | Fonctionne dans VS Code, le CLI, et l'agent de codage Copilot |
| **OpenAI** | Oui | Agents SDK avec prise en charge des skills |
| **Spring AI** | Oui | Écosystème Java via `spring-ai-agent-utils` |

La spécification est maintenue par un groupe de travail communautaire et publiée sur [agentskills.io](https://agentskills.io/specification). Comme le format n'est que des fichiers Markdown dans un dossier, les skills sont portables à travers les plateformes qui prennent en charge la spécification.

---

## Exemples pratiques

### Exemple 1 : skill de migration de base de données

```markdown
---
name: database-migration
description: >
  Creates and reviews database migrations. Use when the user asks to
  add, modify, or remove database tables or columns, or when reviewing
  migration files.
---

## Creating Migrations

1. Verify the current migration state: run `alembic heads` to check for conflicts
2. Create the migration: `alembic revision --autogenerate -m "description"`
3. Review the generated migration file for:
   - Correct up/down operations (both directions should work)
   - No data loss in down migration
   - Appropriate indexes for new columns
   - Nullable columns for existing tables (to avoid breaking existing rows)
4. Test the migration: `alembic upgrade head` then `alembic downgrade -1`

## Common Pitfalls

- Adding a NOT NULL column to an existing table without a default value
  will fail if the table has existing rows. Always add a default or make
  it nullable first, then backfill.
- Renaming columns requires a two-step migration: add new column, migrate
  data, drop old column. Alembic's autogenerate does not handle renames.
- Large table alterations should be done in batches on production. Add a
  note in the migration file if the table has >1M rows.
```

### Exemple 2 : skill de réponse aux incidents

```markdown
---
name: incident-response
description: >
  Guides incident response and post-mortem creation. Use when there is
  a production incident, outage, or when creating post-mortem documents.
---

## During an Incident

1. Assess severity using the service dashboard at `monitoring.internal/overview`
2. Check recent deployments: `gcloud run revisions list --service=api --limit=5`
3. Check error rates: `gcloud logging read "severity>=ERROR" --limit=50 --freshness=1h`
4. If a recent deployment is suspect, rollback:
   `gcloud run services update-traffic api --to-revisions=PREVIOUS_REVISION=100`

## After Resolution

Create a post-mortem document using the template in `assets/postmortem-template.md`
with these sections filled in:
- Timeline of events (with timestamps)
- Root cause analysis
- Impact (users affected, duration, data loss if any)
- What went well in the response
- Action items with owners and due dates
```

---

## Quand utiliser des skills contre d'autres approches

| Situation | Utiliser des skills | Utiliser autre chose |
|-----------|-----------|-------------------|
| L'équipe a des critères de relecture spécifiques | Oui | - |
| L'agent doit suivre un workflow multi-étapes | Oui | - |
| L'agent doit appeler une API | Non | Utiliser un outil ou MCP |
| L'agent a besoin du contexte du projet (commandes de build, structure) | Non | Utiliser AGENTS.md |
| Le workflow est simple et ponctuel | Non | Mettez-le simplement dans le prompt |
| La connaissance change rarement et est spécifique au domaine | Oui | - |
| La connaissance change fréquemment ou nécessite des données en direct | Non | Utiliser le RAG ou des outils |

---

## Points clés à retenir

- Les skills empaquettent une expertise métier sous forme de fichiers Markdown portables et réutilisables
- Elles utilisent la divulgation progressive (N1/N2/N3) pour minimiser le coût en context window
- Les skills indiquent aux agents comment et quand agir ; les outils permettent aux agents d'agir réellement
- Le fichier SKILL.md est le seul composant requis - le frontmatter pour les métadonnées, le corps pour les instructions
- Écrivez des skills ciblées sur un domaine, avec des étapes claires, des points de décision, et des exemples de sortie
- Gardez les instructions du niveau 2 sous 5 000 tokens ; déplacez le matériel détaillé vers des références de niveau 3
- Les skills sont prises en charge à travers plusieurs plateformes : Claude Code, ADK, GitHub Copilot, OpenAI, Spring AI
- Les skills, les outils, et MCP opèrent à des couches différentes et se complètent

---

## Pour aller plus loin

- [Spécification Agent Skills](https://agentskills.io/specification)
- [Documentation des skills ADK](https://google.github.io/adk-docs/skills/)
- [Dépôt Skills d'Anthropic](https://github.com/anthropics/skills)
- [Agent Skills de GitHub Copilot](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills)
- [Skills contre outils MCP - LlamaIndex](https://www.llamaindex.ai/blog/skills-vs-mcp-tools-for-agents-when-to-use-what)

---

[Leçon précédente : MCP en détail](../16-mcp-deep-dive/README.md) | [Leçon suivante : Orchestrateurs ->](../18-orchestrators/README.md)
