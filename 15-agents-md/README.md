# Leçon 15 : AGENTS.md - donner aux agents le contexte du projet

## Introduction

Quand un nouvel ingénieur rejoint votre équipe, vous ne lui remettez pas simplement la base de code en lui disant « bonne chance ». Vous lui donnez des documents d'intégration, expliquez le système de build, indiquez où se trouvent les tests, et le mettez en garde contre les parties du code à ne pas toucher.

Les agents IA de codage ont besoin de la même chose. Sans contexte de projet, un agent devinera vos conventions, utilisera le mauvais lanceur de tests, manquera votre stratégie de branches, et produira généralement du code qui ne correspond pas à votre projet. AGENTS.md résout ce problème en donnant aux agents un document d'intégration structuré qu'ils peuvent lire avant de commencer à travailler.

### ELI5 : pensez à AGENTS.md comme à un livret d'accueil

Imaginez que votre entreprise donne à chaque nouvel employé une fiche récapitulative d'une page le premier jour. Elle liste le mot de passe WiFi, comment lancer le build, où trouver le guide de style, et ce qu'il ne faut jamais toucher. AGENTS.md est cette fiche récapitulative - mais pour les agents IA qui travaillent sur votre base de code.

> **Point clé à retenir :** AGENTS.md est un simple fichier Markdown que vous placez dans votre dépôt pour indiquer aux agents IA comment travailler sur votre projet. La plupart des grands outils de codage IA le lisent automatiquement.

---

## Qu'est-ce qu'AGENTS.md

AGENTS.md est un fichier Markdown au format ouvert qui vit dans votre dépôt. Il fournit des instructions et du contexte aux agents IA de codage - des outils comme Claude Code, Cursor, GitHub Copilot, Gemini, et d'autres.

Il n'y a pas de schéma spécial, aucune exigence de frontmatter YAML, aucune dépendance d'outillage. C'est du Markdown pur. Vous l'écrivez comme vous écririez des instructions pour un humain, car c'est essentiellement ce que c'est - des instructions pour une IA qui se lisent comme pour un humain.

### Le problème qu'il résout

Sans AGENTS.md, chaque fois que vous utilisez un agent IA de codage, vous finissez par vous répéter :

- « Nous utilisons pytest, pas unittest »
- « Lance `make lint` avant de commiter »
- « Le code de l'API est dans `src/api/`, le frontend est dans `web/` »
- « Ne modifie jamais les fichiers dans `vendor/` »

AGENTS.md capture cela une fois, à un seul endroit, afin que chaque agent qui touche à votre code commence avec le bon contexte.

### Comment ça marche

1. Vous placez un fichier `AGENTS.md` à la racine de votre dépôt
2. Les agents IA de codage le détectent et le lisent automatiquement quand ils commencent à travailler sur le projet
3. Les instructions informent la façon dont l'agent écrit du code, exécute les tests, et prend des décisions
4. Vous pouvez aussi placer des fichiers `AGENTS.md` supplémentaires dans des sous-dossiers pour des directives spécifiques à une zone

C'est tout. Aucune configuration, aucune étape de build, aucun travail d'intégration.

---

## Un bref historique

AGENTS.md a émergé en août 2025 d'une collaboration entre OpenAI (Codex), Amp, Google (Jules), Cursor, et Factory. Plutôt que chaque entreprise crée son propre format propriétaire, elles se sont accordées sur un standard ouvert et partagé.

En décembre 2025, AGENTS.md a été confié à l'[Agentic AI Foundation (AAIF)](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation) sous l'égide de la Linux Foundation, aux côtés du Model Context Protocol (MCP) d'Anthropic et de Goose de Block. C'est maintenant un standard ouvert géré par une fondation.

Début 2026, plus de 60 000 dépôts open source incluent un fichier AGENTS.md.

---

## Ce qui va dans un fichier AGENTS.md

L'analyse de milliers de dépôts réels a identifié six domaines centraux qui font la plus grande différence :

### 1. Les commandes

Les commandes exactes pour construire, lint, tester, et exécuter votre projet. Soyez précis - incluez les options et les indicateurs, pas seulement les noms d'outils.

```markdown
## Commands

- Build: `npm run build`
- Lint: `npm run lint -- --fix`
- Test all: `npm test`
- Test single file: `npm test -- --testPathPattern=<filename>`
- Dev server: `npm run dev` (runs on port 3000)
- Type check: `npx tsc --noEmit`
```

Pourquoi c'est important : les agents qui connaissent les commandes exactes peuvent vérifier leur propre travail en lançant les tests et le lint avant de présenter les résultats.

### 2. Les tests

Comment les tests sont organisés, quel framework vous utilisez, et toute convention que l'agent devrait suivre en écrivant de nouveaux tests.

```markdown
## Testing

- Framework: pytest with pytest-asyncio for async tests
- Test location: tests mirror src structure (src/api/users.py -> tests/api/test_users.py)
- Naming: test files start with `test_`, test functions start with `test_`
- Fixtures: shared fixtures live in tests/conftest.py
- Run a single test: `pytest tests/api/test_users.py::test_create_user -v`
```

### 3. La structure du projet

Une carte de l'emplacement des choses. Les agents travaillent mieux quand ils savent dans quels dossiers chercher.

```markdown
## Project Structure

- `src/api/` - REST API endpoints (FastAPI)
- `src/core/` - Business logic, domain models
- `src/db/` - Database models and migrations (SQLAlchemy + Alembic)
- `src/workers/` - Background task processors
- `web/` - React frontend (Vite + TypeScript)
- `infra/` - Terraform infrastructure definitions
- `scripts/` - Developer utility scripts
```

### 4. Le style de code

Les conventions de nommage, les règles de formatage, et les patterns préférés. Un court extrait de code montrant votre style vaut mieux que des paragraphes de description.

```markdown
## Code Style

- Python: Black formatting, isort for imports, Google-style docstrings
- TypeScript: Prettier + ESLint, functional components with hooks
- Naming: snake_case for Python, camelCase for TypeScript
- Prefer explicit over implicit - no magic imports or star exports
- Error handling: use custom exception classes from src/core/exceptions.py
```

### 5. Le workflow Git

La stratégie de branches, les conventions de message de commit, et les exigences de PR.

```markdown
## Git Workflow

- Branch from `main`, prefix with `feat/`, `fix/`, or `chore/`
- Commit messages: conventional commits format (e.g., "feat: add user search endpoint")
- Squash merge PRs
- All PRs require passing CI and one approval
```

### 6. Les limites

Ce que l'agent ne devrait jamais toucher. C'est l'une des sections les plus importantes.

```markdown
## Do Not Modify

- `vendor/` - third-party code, managed externally
- `.env` files - contain secrets, never commit
- `infra/production/` - production infrastructure, requires manual review
- `src/db/migrations/` - generate migrations with Alembic, do not write by hand
- `package-lock.json` - only modify via npm install
```

---

## AGENTS.md hiérarchique pour les monorepos

Vous pouvez placer des fichiers AGENTS.md à plusieurs niveaux de votre arborescence de dossiers. Les agents lisent le fichier le plus proche dans le dossier actuel ou ses parents. C'est utile pour les monorepos où différentes zones ont des conventions différentes.

```
my-monorepo/
  AGENTS.md              # Conventions partagees (workflow git, CI, etc.)
  services/
    api/
      AGENTS.md          # Specifique a Python : pytest, Black, patterns FastAPI
    frontend/
      AGENTS.md          # Specifique a TypeScript : Vitest, Prettier, patterns React
    ml-pipeline/
      AGENTS.md          # Python + notebooks : conventions de donnees, tests de modeles
```

L'AGENTS.md de chaque sous-projet peut se concentrer sur ce qui est unique à cette zone. Le fichier au niveau racine couvre les pratiques partagées.

---

## AGENTS.md contre les autres fichiers de configuration d'agent

Plusieurs outils IA ont leurs propres formats de fichier d'instructions. Le contenu de tous ces formats se chevauche significativement - commandes de build, standards de code, structure de projet. Les différences résident dans les fonctionnalités spécifiques à chaque outil.

| Fichier | Outil | Fonctionnalités spéciales |
|------|------|-----------------|
| **AGENTS.md** | Standard inter-outils | Universel, aucune syntaxe spéciale, lu par la plupart des agents |
| **CLAUDE.md** | Claude Code | Prend en charge les imports `@chemin` pour des instructions modulaires |
| **.cursorrules / .mdc** | Cursor | Frontmatter YAML avec des modes d'activation (Always, Auto, Agent Requested) |
| **.github/copilot-instructions.md** | GitHub Copilot | Fichiers `.instructions.md` ciblés avec des patterns glob |
| **GEMINI.md** | Gemini | Instructions spécifiques à Gemini |

### L'approche pratique

Placez les instructions partagées dans AGENTS.md. La plupart des outils le lisent. N'utilisez des fichiers spécifiques à un outil que lorsque vous avez besoin de fonctionnalités propres à cet outil.

Si vous avez déjà un fichier CLAUDE.md ou .cursorrules, vous n'avez pas nécessairement besoin de tout dupliquer dans AGENTS.md. Mais si vous voulez que vos instructions fonctionnent à travers plusieurs outils, AGENTS.md est le dénominateur commun.

---

## Bonnes pratiques

### Restez concis

Visez 150 lignes ou moins. Les longs fichiers enfouissent les informations importantes et gaspillent des tokens de la context window de l'agent. Si votre AGENTS.md est plus long que votre README, il est probablement trop long.

### Soyez précis et exploitable

Mauvais : « Nous utilisons une stack JavaScript moderne »
Bon : « React 18 avec TypeScript 5.3, Vite 5, et Tailwind CSS 3.4 »

Mauvais : « Suivre les pratiques de test standards »
Bon : « Lancer `npm test -- --coverage` et maintenir une couverture de lignes >80% sur le nouveau code »

### Montrez, ne dites pas

Un exemple de code communique le style plus efficacement qu'un paragraphe de description.

```markdown
## API Endpoint Pattern

New endpoints should follow this structure:

\```python
@router.post("/users", response_model=UserResponse, status_code=201)
async def create_user(
    request: CreateUserRequest,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
) -> UserResponse:
    """Create a new user account."""
    user = await user_service.create(db, request)
    return UserResponse.from_orm(user)
\```
```

### Itérez en fonction du comportement de l'agent

Commencez avec un AGENTS.md minimal. Quand vous remarquez qu'un agent fait la même erreur de manière répétée, ajoutez une instruction pour y remédier. Les meilleurs fichiers AGENTS.md grandissent par itération, pas par planification en amont.

### Traitez-le comme du code

Mettez à jour AGENTS.md dans la même PR quand vous changez les processus de build, les conventions de test, ou la structure du projet. Des instructions obsolètes sont pires que pas d'instructions du tout, car elles induisent activement les agents en erreur.

### Incluez le « pourquoi » quand c'est important

Pour les règles non évidentes, une brève explication aide l'agent à appliquer correctement la règle dans les cas limites.

```markdown
- Do not use `datetime.now()` directly. Use `src/core/clock.py` instead.
  This allows tests to control time without monkeypatching.
```

---

## Un exemple complet

Voici un AGENTS.md réaliste pour une application web Python :

```markdown
# AGENTS.md

## Project

Order management API built with FastAPI and PostgreSQL.
Python 3.12, managed with Poetry.

## Commands

- Install dependencies: `poetry install`
- Run dev server: `poetry run uvicorn src.main:app --reload --port 8000`
- Run all tests: `poetry run pytest`
- Run single test: `poetry run pytest tests/path/to/test.py -v`
- Lint: `poetry run ruff check src tests`
- Format: `poetry run ruff format src tests`
- Type check: `poetry run mypy src`
- Generate migration: `poetry run alembic revision --autogenerate -m "description"`
- Apply migrations: `poetry run alembic upgrade head`

## Project Structure

- `src/api/` - API route handlers
- `src/core/` - Business logic and domain models
- `src/db/` - SQLAlchemy models and Alembic migrations
- `src/services/` - External service integrations
- `tests/` - Mirrors src structure

## Code Style

- Ruff for linting and formatting (config in pyproject.toml)
- Google-style docstrings on public functions
- Type hints on all function signatures
- Prefer `async def` for all route handlers
- Use dependency injection via FastAPI's `Depends()`

## Testing

- pytest with pytest-asyncio
- Use factories from `tests/factories.py` to create test data
- Tests run against a real PostgreSQL database (not mocks)
- Each test function gets a fresh transaction that rolls back

## Git

- Branch naming: `feat/`, `fix/`, `chore/` prefixes
- Conventional commits
- Squash merge to main

## Do Not Modify

- `alembic/versions/` - Do not edit migration files by hand
- `.env` and `.env.local` - Contains secrets
- `src/core/legacy_adapter.py` - Scheduled for removal, do not add new code here
```

---

## Quand AGENTS.md ne suffit pas

AGENTS.md gère bien le contexte au niveau du projet. Mais il existe des situations où vous avez besoin de plus :

- **L'accès dynamique aux outils** - si les agents ont besoin d'interroger des bases de données, d'appeler des API, ou d'interagir avec des services externes, vous avez besoin de [serveurs MCP](../14-agent-protocols-mcp-and-a2a/README.md) ou d'outils, pas seulement d'instructions.
- **Les workflows réutilisables** - si vous voulez empaqueter des processus multi-étapes que les agents peuvent invoquer, consultez les [Agent Skills](../17-agent-skills/README.md).
- **La coordination inter-agents** - si plusieurs agents doivent travailler ensemble, vous avez besoin d'une [couche d'orchestration](../18-orchestrators/README.md).

AGENTS.md est la fondation. Voyez-le comme la première couche de contexte qui fait mieux fonctionner tout le reste.

---

## Essayez par vous-même

1. Créez un fichier `AGENTS.md` dans l'un de vos projets
2. Commencez avec juste les sections Commandes et Structure du projet
3. Utilisez un agent IA de codage sur ce projet et voyez s'il suit vos instructions
4. Quand vous remarquez que l'agent fait des erreurs, ajoutez des instructions pour y remédier
5. Continuez à affiner jusqu'à ce que l'agent produise systématiquement du code adapté à votre projet

---

## Points clés à retenir

- AGENTS.md est un simple fichier Markdown qui indique aux agents IA comment travailler sur votre projet
- C'est un standard ouvert pris en charge par la plupart des grands outils de codage IA
- Concentrez-vous sur six domaines : commandes, tests, structure de projet, style de code, workflow git, et limites
- Restez concis (moins de 150 lignes), précis, et à jour
- Commencez simple et itérez en fonction des erreurs de l'agent
- Utilisez des fichiers hiérarchiques dans les monorepos pour des directives spécifiques à une zone
- Placez les instructions partagées dans AGENTS.md, les fonctionnalités spécifiques à un outil dans leurs fichiers de configuration respectifs

---

## Pour aller plus loin

- [Site officiel d'AGENTS.md](https://agents.md/)
- [Spécification d'AGENTS.md sur GitHub](https://github.com/agentsmd/agents.md)
- [Comment écrire un excellent AGENTS.md - blog GitHub](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/)
- [Agentic AI Foundation (AAIF)](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)
- [Instructions personnalisées avec AGENTS.md - OpenAI Codex](https://developers.openai.com/codex/guides/agents-md)

---

[Leçon suivante : MCP en détail ->](../16-mcp-deep-dive/README.md)
