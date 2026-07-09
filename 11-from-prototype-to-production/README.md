# Leçon 11 : du prototype à la production - déployer votre agent

## Introduction

Vous avez construit un agent fonctionnel. Il gère vos cas de test, impressionne votre équipe lors des démos, et semble magique. Vous devez maintenant le livrer à de vrais utilisateurs. C'est là que les choses se compliquent.

L'écart entre « fonctionne sur mon ordinateur portable » et « fonctionne de façon fiable pour des milliers d'utilisateurs » est énorme. Dans le logiciel traditionnel, être prêt pour la production signifie principalement gérer les cas limites, la mise à l'échelle, et le monitoring. Avec les agents, vous avez tout cela, plus le défi fondamental que le comportement de votre système est non déterministe et difficile à prévoir entièrement.

### ELI5 : amener un agent en production, c'est comme ouvrir un restaurant

Vous avez cuisiné de bons petits plats à la maison pour vos amis. Tout le monde adore votre cuisine. Maintenant, vous voulez ouvrir un restaurant. Soudain, vous devez penser à des choses qui n'ont jamais compté à la maison : les inspections sanitaires, les chaînes d'approvisionnement, des recettes cohérentes afin que chaque plat ait le même goût, la formation du personnel, la gestion des plaintes, la maîtrise des coûts, et s'assurer que la cuisine ne prenne pas feu un samedi soir chargé. La compétence culinaire est la même, mais tout ce qui l'entoure change complètement.

C'est l'écart entre le prototype et la production. La logique centrale de votre agent ne changera peut-être pas beaucoup, mais tout ce qui l'entoure - évaluation, déploiement, monitoring, gestion des coûts, processus d'équipe - doit être construit à partir de zéro.

> **Point clé à retenir :** le « dernier kilomètre » du prototype à la production représente souvent 80% de l'effort total. Planifiez-le dès le départ.

---

## L'écart de production

Voici ce qui change quand vous passez du prototype à la production :

| Dimension | Prototype | Production |
|-----------|-----------|------------|
| **Utilisateurs** | Vous et votre équipe | Des centaines ou des milliers d'utilisateurs réels |
| **Entrées** | Cas de test sélectionnés | Tout ce que n'importe qui tape, y compris des entrées adverses |
| **Disponibilité** | Redémarrer quand ça casse | Doit être disponible 24/7 avec une dégradation gracieuse |
| **Latence** | « Ça prend quelques secondes » convient | Les utilisateurs attendent des réponses en moins d'une seconde pour les requêtes simples |
| **Coût** | Le taux de consommation n'a pas d'importance | Chaque token coûte de l'argent à grande échelle |
| **Qualité** | « Ça marche généralement » est acceptable | Une qualité constante est requise ; les mauvaises réponses érodent la confiance |
| **Sécurité** | Tests informels | Guardrails systématiques, monitoring, et réponse aux incidents |
| **Débogage** | Instructions print | Logs structurés, traces, et métriques |
| **Mises à jour** | Modifier et redémarrer | Pipeline CI/CD avec des portes d'évaluation |

### Pourquoi les démos nous trompent

Les démos fonctionnent parce que la personne qui la présente sait quelles entrées fonctionnent bien. Elle évite les cas limites. Elle relance les échecs hors écran. Elle choisit le meilleur exemple parmi plusieurs essais.

La production, c'est l'inverse. Les vrais utilisateurs vont :
- Faire des fautes de frappe, utiliser de l'argot, écrire dans des langues inattendues
- Poser des questions que votre agent n'a jamais été conçu pour gérer
- Fournir des entrées extrêmement longues ou extrêmement courtes
- Réessayer exactement la même requête plusieurs fois s'ils ne sont pas satisfaits du résultat
- Découvrir des modes d'échec que vous n'aviez jamais imaginés

C'est pourquoi le déploiement conditionné par l'évaluation est si important. Vous ne devriez pas livrer une version d'agent qui n'a pas été testée par rapport à un ensemble complet de scénarios réels.

---

## Rôles d'équipe : qui est impliqué

Mettre un agent en production n'est pas un effort solitaire. Cela implique généralement plusieurs rôles travaillant ensemble :

| Rôle | Responsabilité |
|------|---------------|
| **Ingénieur IA** | Logique de l'agent, conception de prompts, intégration d'outils, création d'evals |
| **Ingénieur plateforme** | Infrastructure, pipelines de déploiement, maillage de services, mise à l'échelle |
| **Ingénieur données** | Pipelines de données pour le RAG, bases de connaissances, gestion des données d'entraînement |
| **ML Ops / AI Ops** | Service de modèles, versionnage, tests A/B, tableaux de bord de monitoring |
| **DevOps / SRE** | Fiabilité, réponse aux incidents, alertes, suivi des coûts |
| **Chef de produit** | Exigences utilisateur, métriques de succès, priorisation |
| **Sécurité / Trust & Safety** | Guardrails, red teaming, conformité, revues de sécurité |

Dans les équipes plus petites, une personne peut porter plusieurs casquettes. Mais les responsabilités existent toujours, peu importe combien de personnes les partagent.

---

## Le déploiement conditionné par l'évaluation

C'est la pratique la plus importante pour livrer des agents en toute sécurité. Le principe est simple : **aucune version d'agent n'est livrée sans avoir réussi les evals.**

Dans la leçon 9, nous avons couvert comment construire des evals. Voici comment ils s'intègrent dans le processus de déploiement :

```
Changement de code --> Les evals passent-ils ? --Non--> Corriger et reessayer
                    |
                   Oui
                    |
                    v
              Deployer en staging (pre-production)
                    |
                    v
        Les evals de staging passent-ils ? --Non--> Corriger et reessayer
                    |
                   Oui
                    |
                    v
        Deployer en production (canari)
                    |
                    v
      Les metriques de production sont-elles bonnes ? --Non--> Rollback
                    |
                   Oui
                    |
                    v
        Deploiement complet en production
```

### Qu'est-ce qui fait une bonne porte d'évaluation ?

Votre suite d'evals pour le déploiement devrait couvrir :

| Catégorie | Quoi tester | Critère de succès |
|----------|-------------|---------------|
| **Exactitude fonctionnelle** | L'agent produit-il des réponses correctes ? | >= seuil sur les métriques d'exactitude |
| **Utilisation d'outils** | L'agent appelle-t-il les bons outils avec les bons arguments ? | Outils appelés correctement dans >= X% des cas |
| **Sécurité** | L'agent résiste-t-il à l'injection de prompt et suit-il les politiques ? | Taux de réussite de 100% sur les cas critiques pour la sécurité |
| **Latence** | L'agent répond-il dans un délai acceptable ? | Latence P95 < cible |
| **Coût** | L'agent reste-t-il dans les budgets de tokens ? | Coût moyen par interaction < budget |
| **Régression** | Les cas qui passaient précédemment passent-ils toujours ? | Aucune régression sur les cas déjà validés |

Les evals de sécurité devraient avoir une porte stricte - tout échec bloque le déploiement. Les autres catégories peuvent avoir des seuils plus souples, où vous acceptez de petites régressions si la qualité globale s'améliore.

---

## CI/CD pour les agents

L'intégration continue et le déploiement continu pour les agents suivent les mêmes principes que le CI/CD traditionnel, mais avec des étapes spécifiques aux agents. Pensez-y en trois phases.

### Phase 1 : pré-fusion (à chaque pull request)

Ces vérifications s'exécutent rapidement et détectent les problèmes évidents avant que le code ne soit fusionné.

```yaml
# Exemple : verifications pre-fusion
pre_merge:
  - lint:
      - Verifier le formatage et la syntaxe des prompts
      - Valider que les definitions d'outils correspondent aux schemas
      - Analyse statique de la configuration de l'agent

  - unit_tests:
      - Tester les fonctions d'outils individuelles
      - Tester la logique des guardrails
      - Tester les analyseurs d'entree/sortie

  - basic_evals:
      - Executer un petit ensemble d'evals rapide (50-100 cas)
      - Se concentrer sur la detection de regression
      - Cible : termine en moins de 5 minutes
```

### Phase 2 : post-fusion (à chaque fusion vers main)

Une fois le code fusionné, exécutez des vérifications plus complètes avant de promouvoir vers le staging.

```yaml
# Exemple : validation post-fusion
post_merge:
  - staging_deployment:
      - Deployer vers l'environnement de staging
      - Verifier que les health checks passent

  - broad_evals:
      - Executer la suite d'evals complete (500-1000+ cas)
      - Inclure les evals de securite
      - Inclure les benchmarks de latence et de cout
      - Cible : termine en moins de 30 minutes

  - integration_tests:
      - Tester les flux de bout en bout avec de vraies connexions d'outils
      - Verifier les integrations de services externes
```

### Phase 3 : porte de production (avant le déploiement en production)

La vérification finale avant que de vrais utilisateurs ne voient la nouvelle version.

```yaml
# Exemple : porte de production
production_gate:
  - full_evals:
      - Suite d'evals complete incluant les cas limites
      - Cas de test adverses
      - Verifications de coherence inter-modeles (si plusieurs modeles sont utilises)

  - safety_review:
      - Les evals de securite automatises doivent passer a 100%
      - Relecture humaine pour les changements de prompt significatifs
      - Validation du red team pour les changements de fonctionnalites majeurs

  - approval:
      - Approbation automatique si toutes les verifications passent
      - Approbation manuelle requise si une verification est limite
```

### Gérer les prompts dans le CI/CD

Les prompts méritent la même discipline de contrôle de version que le code :

- Stockez les prompts dans le contrôle de version (pas dans une base de données ou un service de configuration difficile à comparer)
- Relisez les changements de prompts dans des pull requests, tout comme les changements de code
- Suivez quelle version de prompt est déployée sur quel environnement
- Facilitez le retour à une version de prompt précédente

```
prompts/
  customer_support/
    system_prompt.txt      # Les instructions systeme principales
    tool_descriptions.txt  # Descriptions et schemas d'outils
    safety_rules.txt       # Instructions specifiques a la securite
    version.txt            # Identifiant de version actuel
```

---

## Stratégies de déploiement sûres

Même avec des evals complets, la production peut vous surprendre. Les stratégies de déploiement sûres limitent le rayon d'impact quand quelque chose tourne mal.

### Les déploiements canari (canary)

Router un petit pourcentage du trafic vers la nouvelle version. Surveiller les problèmes avant d'augmenter le pourcentage.

```
Trafic ---> [Repartiteur de charge]
                |         |
               95%       5%
                |         |
                v         v
         [Version 1]  [Version 2 - Canari]
         (actuelle)    (nouvelle)
```

**Comment ça marche :**
1. Déployer la nouvelle version aux côtés de l'actuelle
2. Router 5% du trafic vers la nouvelle version
3. Surveiller les métriques clés (taux d'erreur, latence, satisfaction utilisateur, incidents de sécurité)
4. Si les métriques sont saines après une période définie, augmenter à 25%, puis 50%, puis 100%
5. Si une métrique se dégrade, router tout le trafic de retour vers la version actuelle

### Les déploiements bleu-vert (blue-green)

Maintenir deux environnements de production identiques. Basculer tout le trafic de l'un vers l'autre.

```
Avant :   Trafic --> [Bleu - v1.2 ACTIF]      [Vert - inactif]
Pendant : Trafic --> [Bleu - v1.2]             [Vert - v1.3 ACTIF]
```

L'avantage est une bascule propre et un rollback instantané (il suffit de revenir au Bleu). L'inconvénient est que vous avez besoin du double d'infrastructure pendant la transition.

### Les tests A/B

Router le trafic vers différentes versions d'agent et comparer leur performance sur des interactions réelles.

| Version A | Version B | Métrique | Gagnant |
|-----------|-----------|--------|--------|
| Basée sur GPT, prompts verbeux | Basée sur Gemini, prompts concis | Taux d'achèvement de tâche | Comparer après N interactions |
| Pattern ReAct | Plan-puis-exécute | Satisfaction utilisateur | Comparer après N interactions |
| Modèle A, 3 nouvelles tentatives d'outil | Modèle A, 1 nouvelle tentative d'outil | Coût par interaction | Comparer après N interactions |

Les tests A/B sont particulièrement précieux pour les agents, car ils vous permettent de comparer différents modèles, prompts, et architectures sur du trafic réel.

### Les feature flags (indicateurs de fonctionnalité)

Contrôlez les capacités de l'agent avec des indicateurs à l'exécution qui peuvent être activés/désactivés sans redéploiement.

```python
# Exemple : feature flags pour les capacites de l'agent
if feature_flags.is_enabled("new_refund_flow", user_id=user.id):
    agent.enable_tool("process_refund_v2")
else:
    agent.enable_tool("process_refund_v1")

if feature_flags.is_enabled("extended_context_window"):
    agent.set_max_context(128000)
else:
    agent.set_max_context(32000)
```

Les feature flags vous permettent de déployer progressivement de nouvelles capacités, de désactiver rapidement des fonctionnalités problématiques, et de mener des expériences sur des sous-ensembles d'utilisateurs.

---

## L'observabilité en production

Une fois votre agent en cours d'exécution en production, vous devez voir ce qu'il fait. L'observabilité pour les agents a trois piliers, tout comme les systèmes traditionnels - mais les détails diffèrent.

### Les logs

Des logs structurés qui capturent chaque événement significatif du cycle de vie de l'agent :

```json
{
  "timestamp": "2025-06-15T10:23:45Z",
  "session_id": "sess_abc123",
  "event": "tool_call",
  "tool": "search_knowledge_base",
  "arguments": {"query": "politique de retour pour l'electronique"},
  "result_status": "success",
  "latency_ms": 234,
  "tokens_used": {"input": 1250, "output": 380}
}
```

**Quoi journaliser :**
- Chaque appel au LLM (modèle, tokens en entrée, tokens en sortie, latence)
- Chaque appel d'outil (nom de l'outil, arguments, statut du résultat, latence)
- Les décisions de l'agent (quel chemin a été choisi et pourquoi)
- Les activations de guardrails (ce qui a été bloqué et pourquoi)
- Les événements d'escalade
- Le début/fin de session avec des métriques de synthèse

### Les traces

Les traces montrent le parcours complet d'une seule requête à travers votre agent, incluant toutes les étapes, appels d'outils, et décisions en cours de route.

```
[Requete utilisateur] "Aide-moi a retourner ma commande"
    |
    +-- [Appel LLM 1] Comprendre l'intention (150ms)
    |       Modele : gemini-2.0-flash, Tokens : 800 en entree / 120 en sortie
    |
    +-- [Appel d'outil] lookup_order(order_id="12345") (340ms)
    |       Statut : succes
    |
    +-- [Appel d'outil] check_return_eligibility(order_id="12345") (180ms)
    |       Statut : succes, eligible=true
    |
    +-- [Appel LLM 2] Generer la reponse (200ms)
    |       Modele : gemini-2.0-flash, Tokens : 1200 en entree / 250 en sortie
    |
    +-- [Guardrail de sortie] Verification PII (15ms)
    |       Statut : reussi
    |
    [Reponse] "Votre commande #12345 est eligible au retour..."

    Total : 885ms, Cout : 0,003 $
```

[OpenTelemetry](https://opentelemetry.io/) est le standard de l'industrie pour le traçage distribué. De nombreux frameworks d'agents prennent en charge OpenTelemetry nativement, et la suite d'opérations de Google Cloud (Cloud Logging, Cloud Trace, Cloud Monitoring) s'intègre nativement avec OpenTelemetry.

### Les métriques

Des métriques agrégées qui vous indiquent la performance globale de votre agent :

| Métrique | Ce qu'elle vous indique | Exemple de seuil d'alerte |
|--------|-------------------|------------------------|
| **Taux d'achèvement de tâche** | À quelle fréquence l'agent termine avec succès les requêtes utilisateur | Chute en dessous de 85% |
| **Latence moyenne** | Combien de temps les utilisateurs attendent les réponses | P95 dépasse 5 secondes |
| **Coût par interaction** | Combien coûte chaque conversation | Moyenne dépasse 0,10 $ |
| **Taux d'escalade** | À quelle fréquence l'agent transmet à des humains | Dépasse 20% |
| **Taux d'incidents de sécurité** | À quelle fréquence les guardrails sont déclenchés | Toute augmentation au-dessus de la référence |
| **Taux d'erreur des outils** | À quelle fréquence les appels d'outils échouent | Dépasse 5% |
| **Satisfaction utilisateur** | Scores pouce levé/baissé ou CSAT | Chute en dessous de 4,0/5,0 |

### Construire des tableaux de bord

Un tableau de bord d'agent en production devrait montrer d'un coup d'œil :

```
+-------------------------------------------------------+
|  Tableau de bord de sante de l'agent                   |
+-------------------------------------------------------+
|                                                        |
|  Statut : SAIN              Sessions actives : 1 247   |
|                                                        |
|  +-------------------+  +-------------------+          |
|  | Taux d'achevement |  | Latence moyenne   |          |
|  | 92,3% (+0,5%)     |  | 1,2s (-0,1s)      |          |
|  +-------------------+  +-------------------+          |
|                                                        |
|  +-------------------+  +-------------------+          |
|  | Cout / Session    |  | Taux d'escalade   |          |
|  | 0,042 $ (-0,003 $)|  | 8,1% (+0,2%)      |          |
|  +-------------------+  +-------------------+          |
|                                                        |
|  Incidents de securite recents : 0 (24h)               |
|  Erreurs recentes : 12 (24h, 0,04% des sessions)       |
|                                                        |
+-------------------------------------------------------+
```

---

## La boucle Observer-Agir-Évoluer

La production n'est pas une destination. C'est le début d'un cycle d'amélioration continue.

```
    +----------+
    | Observer |  <-- Collecter metriques, logs, traces, retour utilisateur
    +----+-----+
         |
         v
    +----+-----+
    |  Agir    |  <-- Identifier les problemes, prioriser les ameliorations
    +----+-----+
         |
         v
    +----+-----+
    | Evoluer  |  <-- Mettre a jour les prompts, outils, evals, guardrails
    +----+-----+
         |
         +-------> Retour a Observer
```

### Observer

Collectez des données sur la performance de votre agent en production :

- **Quantitatif :** tableaux de bord de métriques, résultats d'evals automatisés sur le trafic de production
- **Qualitatif :** retour utilisateur, tickets de support, relectures de conversations
- **Adversarial :** red teaming continu, détection de nouveaux patterns d'attaque

### Agir

Transformez les observations en actions concrètes :

- Échoue sur un type spécifique de requête ? Ajoutez-le à votre ensemble d'evals et améliorez le prompt.
- Les erreurs d'outils s'accumulent ? Investiguez la cause racine et ajoutez une meilleure gestion des erreurs.
- Les utilisateurs sont systématiquement confus par un pattern de réponse ? Révisez les instructions de l'agent.
- Nouveau vecteur d'attaque découvert ? Ajoutez un guardrail et un eval de sécurité.

### Évoluer

Déployez les améliorations via votre pipeline CI/CD conditionné par l'évaluation :

- Mettez à jour les prompts et relancez les evals
- Ajoutez de nouveaux outils ou modifiez les existants
- Étendez la suite d'evals pour couvrir les nouveaux cas limites découverts
- Ajustez les guardrails en fonction des menaces observées
- Réentraînez ou remplacez les modèles si de meilleures options deviennent disponibles

L'idée clé est que votre suite d'evals grandit dans le temps. Chaque incident en production, chaque plainte d'utilisateur, et chaque cas limite devient un nouvel eval. Cela signifie que votre agent devient plus difficile à casser à chaque itération.

---

## Gestion des coûts

Les agents basés sur des LLM peuvent être coûteux à grande échelle. Une seule conversation peut impliquer plusieurs appels au LLM, chacun consommant des milliers de tokens. Multipliez cela par des milliers d'utilisateurs et les coûts s'accumulent rapidement.

### Le model routing

Utilisez le modèle le moins cher capable de gérer chaque tâche. Toutes les étapes n'ont pas besoin de votre modèle le plus puissant.

```
Requete utilisateur
    |
    v
[Routeur] --Requete simple--> Gemini Flash-Lite ($)
    |
    +-----Complexite moyenne--> Gemini Flash ($$)
    |
    +-----Raisonnement complexe--> Gemini Pro ($$$)
```

| Type de tâche | Palier de modèle recommandé | Justification |
|-----------|----------------------|-----------|
| Classification d'intention | Petit / Flash-Lite | Tâche de classification simple |
| Récupération d'information | Moyen / Flash | Nécessite une bonne compréhension, génération modérée |
| Raisonnement complexe | Grand / Pro | Raisonnement multi-étapes, jugement nuancé |
| Synthèse | Moyen / Flash | Bon équilibre entre qualité et coût |
| Vérifications de sécurité | Petit / Flash-Lite | Reconnaissance de motifs, classification |

### La mise en cache

Mettez en cache les réponses pour les requêtes répétées ou similaires afin d'éviter des appels redondants au LLM.

| Stratégie de cache | Quand l'utiliser |
|-----------------|-------------|
| **Cache de correspondance exacte** | Requêtes de type FAQ où de nombreux utilisateurs demandent la même chose |
| **Cache sémantique** | Requêtes différentes dans la formulation mais identiques dans le sens |
| **Cache de résultats d'outils** | Sorties d'outils qui ne changent pas fréquemment (par ex., consultation de catalogue produit) |
| **Cache de prompt** | Réutiliser des préfixes mis en cache pour les prompts système à travers les appels (Vertex AI prend en charge la mise en cache de contexte) |

### Les budgets de tokens

Fixez des limites strictes sur le nombre de tokens qu'un agent peut consommer par session.

```python
# Exemple : application d'un budget de tokens
class TokenBudget:
    def __init__(self, max_tokens: int):
        self.max_tokens = max_tokens
        self.used_tokens = 0

    def can_proceed(self, estimated_tokens: int) -> bool:
        return (self.used_tokens + estimated_tokens) <= self.max_tokens

    def record_usage(self, actual_tokens: int):
        self.used_tokens += actual_tokens

# Utilisation
budget = TokenBudget(max_tokens=50000)  # par session

while agent.has_next_step():
    estimated = agent.estimate_next_step_tokens()
    if not budget.can_proceed(estimated):
        agent.respond("J'ai atteint ma limite de traitement pour cette "
                      "session. Laissez-moi resumer ce que j'ai trouve "
                      "jusqu'ici.")
        break
    result = agent.execute_next_step()
    budget.record_usage(result.tokens_used)
```

### Le monitoring des coûts

Suivez les coûts à plusieurs niveaux :

| Niveau | Quoi suivre | Pourquoi |
|-------|--------------|-----|
| **Par requête** | Tokens utilisés, palier de modèle, appels d'outils | Déboguer des requêtes individuelles coûteuses |
| **Par session** | Coût total d'une conversation | Fixer et appliquer des budgets par session |
| **Par utilisateur** | Coût agrégé par utilisateur dans le temps | Identifier les patterns d'usage et les valeurs aberrantes |
| **Par fonctionnalité** | Coût de capacités spécifiques de l'agent | Décider quelles fonctionnalités sont rentables |
| **Global** | Dépenses quotidiennes/hebdomadaires/mensuelles | Planification budgétaire et prévisions |

---

## Une checklist de préparation pour la production

Avant de lancer votre agent auprès d'utilisateurs en production, parcourez cette checklist :

### Fiabilité
- [ ] Health checks et sondes de vivacité configurés
- [ ] Dégradation gracieuse quand les dépendances échouent (API du modèle en panne, outil indisponible)
- [ ] Logique de nouvelle tentative avec backoff exponentiel pour les échecs transitoires
- [ ] Circuit breakers pour les appels de services externes
- [ ] Limites de timeout sur tous les appels au LLM et aux outils

### Déploiement
- [ ] Pipeline CI/CD avec des portes d'évaluation à chaque étape
- [ ] Procédure de rollback testée et documentée
- [ ] Déploiement canari ou bleu-vert configuré
- [ ] Feature flags pour les nouvelles capacités
- [ ] Versionnage des prompts et suivi des changements

### Observabilité
- [ ] Journalisation structurée pour tous les événements de l'agent
- [ ] Traçage distribué avec OpenTelemetry
- [ ] Tableaux de bord pour les métriques clés (taux d'achèvement, latence, coût, sécurité)
- [ ] Alertes configurées pour les seuils critiques
- [ ] Rotation d'astreinte et runbook de réponse aux incidents

### Coût
- [ ] Model routing configuré (le bon modèle pour chaque tâche)
- [ ] Stratégie de mise en cache implémentée
- [ ] Budgets de tokens par session
- [ ] Monitoring et alertes de coûts
- [ ] Relectures de coûts régulières et optimisation

### Sécurité
- [ ] Guardrails de la leçon 10 implémentés et testés
- [ ] Evals de sécurité réussis à 100%
- [ ] Relecture du red team terminée
- [ ] Plan de réponse aux incidents pour les échecs de sécurité
- [ ] Canal de retour utilisateur pour signaler les problèmes

---

## Points clés à retenir

1. **L'écart prototype-production est réel et important.** Planifiez les préoccupations de production dès le départ. Le « dernier kilomètre » représente la majorité du travail.

2. **Le déploiement conditionné par l'évaluation est non négociable.** Aucune version d'agent ne devrait atteindre la production sans réussir une suite d'evals complète. Votre suite d'evals est votre garantie de qualité.

3. **Le CI/CD pour les agents a trois phases.** Les vérifications pré-fusion détectent rapidement les problèmes évidents. La validation post-fusion exécute des evals plus larges. Les portes de production garantissent la sécurité et la qualité avant que de vrais utilisateurs ne soient affectés.

4. **Les stratégies de déploiement sûres limitent le rayon d'impact.** Les déploiements canari, les feature flags, et les tests A/B vous permettent de détecter les problèmes avant qu'ils n'affectent tous les utilisateurs.

5. **L'observabilité est essentielle.** Vous ne pouvez pas améliorer ce que vous ne pouvez pas voir. Investissez dans les logs, les traces, et les métriques dès le premier jour.

6. **La gestion des coûts nécessite une attention active.** Le model routing, la mise en cache, et les budgets de tokens peuvent réduire considérablement les coûts sans sacrifier la qualité.

7. **La production est le début, pas la fin.** La boucle Observer-Agir-Évoluer signifie que votre agent s'améliore continuellement sur la base d'un usage réel.

---

## Pour aller plus loin

- [Déployer des agents sur Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/deploy) - Guide officiel pour déployer des agents sur Google Cloud
- [Agent Starter Pack](https://github.com/GoogleCloudPlatform/agent-starter-pack) - Modèles prêts pour la production pour déployer des agents sur Google Cloud avec CI/CD, observabilité, et évaluation
- [Vertex AI Model Monitoring](https://cloud.google.com/vertex-ai/docs/model-monitoring/overview) - Surveiller la performance des modèles en production
- [OpenTelemetry](https://opentelemetry.io/) - Le standard de l'industrie pour le traçage distribué et l'observabilité

---

Leçon suivante : [Premiers pas avec Vertex AI et ADK](../12-getting-started-with-vertex-and-adk/README.md)
