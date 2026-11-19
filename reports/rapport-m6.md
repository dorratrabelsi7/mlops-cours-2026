# Rapport — Lab 6 : CI/CD/CT de bout en bout

Travail individuel · Fork : `dorratrabelsi7/mlops-cours-2026` · Dataset : german-credit ·
Image en prod à la fin du lab : `ghcr.io/dorratrabelsi7/mlops-cours-2026/scoring-api:2.1.8` ·
`@champion` : v15

## 1. Architecture

Trois chaînes ne communiquent que par Git. Pour chaque flèche, le texte dit qui pousse et qui tire.

    CI   : poste --git push--> GitHub --> GitHub Actions (ruff, pytest, build, test de fumée)
           Actions POUSSE l'image --> GHCR (scoring-api:2.1.N)
           Actions POUSSE le tag --> Git (k8s/overlays/staging), commit de github-actions[bot]
    CD   : Argo CD (dans kind) TIRE Git toutes les 60 s et applique les overlays
           staging --> namespace scoring-staging ; prod --> namespace scoring (canary KServe)
           les nœuds kind TIRENT l'image depuis GHCR (package public)
    CT   : Evidently (poste) --POST + jeton--> webhook (poste, 127.0.0.1:8095) --> KFP (kind)
           KFP : ingest, validate, prepare, train, evaluate, challenger, gate --> MLflow (@champion)
           le webhook TIRE l'état du run (KFP) et du Registry (MLflow), puis POUSSE sur Git :
           commit du modèle, canary 10 %, promotion 100 %

Les décisions de déploiement sont donc des commits (`pipelines/gitops.py`), jamais des
`kubectl patch` : Argo CD (selfHeal) annule toute modification manuelle, ce que j'ai vérifié
en forçant `canaryTrafficPercent` à 50 (retour à 100 en moins de 70 s).

## 2. Temps du cycle

Démo de l'étape 9 (dérive × 1,5). Heures locales (UTC+1), relevées dans le journal du webhook,
`git log`, l'API de KFP et le rapport de dérive.

| Étape | Début | Fin | Durée |
|---|---|---|---|
| Dérive détectée → run KFP lancé | 23:58:22 | 23:58:26 | 4 s |
| Pipeline CT (ingest → gate) | 23:58:26 | 00:02:45 | 4 min 19 s |
| Fin du run → commit du modèle par le bot | 00:02:45 | 00:03:00 | 15 s |
| Commit du modèle → image publiée et tag de staging commité (CI) | 00:03:00 | 00:05:07 | 2 min 07 s |
| Tag de staging → staging servi et vérifié → canary 10 % en prod | 00:05:07 | 00:05:52 | 45 s |
| Canary 10 % → promotion 100 % | 00:05:52 | 00:06:46 | 54 s |
| **Cycle complet** | 23:58:22 | 00:06:46 | **8 min 24 s** |

L'étape qui domine est le pipeline KFP (4 min 19 s, soit 51 % du cycle), puis la CI
(2 min 07 s). Pistes pour raccourcir : un entraînement plus léger ou parallélisé, la mise en
cache des étapes qui ne changent pas (sauf l'ingestion du lot, volontairement sans cache), et
un webhook GitHub vers Argo CD à la place de l'interrogation toutes les 60 s. La CI est déjà
optimisée par les caches de couches Docker.

## 3. Gates et décisions

- **Gate de données** (`validate-data`, Lab 3). Un premier essai avec une dérive forte
  (variables × 2) a été bloqué : 2 contrôles sur 67 ont échoué
  (`expect_column_values_to_be_between` sur `duration` et `age`), le pipeline s'est arrêté
  avant tout entraînement et rien n'a été déployé (prod inchangée, `@champion` inchangé).
  Une dérive qui sort du contrat de données est donc refusée avant d'atteindre la gate de
  promotion.
- **Gate de promotion** (KFP, `promotion-gate`). Premier cycle (× 1,5) : v14 promue
  (roc_auc 0,8094, coût métier 108, ancien champion v13). Cycle de la démo : v15 promue,
  mêmes règles :

| Règle | Résultat |
|---|---|
| roc_auc >= 0,7 | OK |
| business_cost <= 108 (champion v14) | OK |

| Modèle | roc_auc | coût métier |
|---|---|---|
| challenger v15 | 0,8094 | 108 |
| champion v14 | 0,8094 | 108 |

  Les deux modèles ont les mêmes chiffres : même lot simulé, mêmes hyperparamètres. La règle
  `coût <= champion` n'exige aucune amélioration, un modèle identique est donc promu.
- **Canary** (KServe, piloté par Git). En manuel (v14, image 2.1.7) : 24 réponses sur 200 en
  2.1.7 et 176 en 2.0.0, soit 12 % pour 10 % attendus (écart-type binomial ≈ 4,2 : écart
  d'environ 1 écart-type, normal). La promotion a donné 100 % de réponses en 2.1.7.
  Dans le cycle automatique (v15, image 2.1.8), l'analyse naïve (50 prédictions sur la
  nouvelle révision, aucune erreur tolérée) a conduit à la promotion 54 s après le canary.
  Aucun retour arrière n'a eu lieu (`gitops canary 0 --push` ou `gitops rollback --push`
  restent disponibles).

## 4. Points d'attention

- **Webhook.** Il exige un jeton (`Authorization: Bearer`), comparé en temps constant ; sans
  jeton il répond 401, sans jeton configuré côté serveur 503. Il n'écoute que sur
  `127.0.0.1`, valide les événements (`drift_share` entre 0 et 1) et limite le débit : un
  seul ré-entraînement à la fois, 10 minutes minimum entre deux, seuil de 10 % des variables.
- **Gate automatique.** Aucune validation humaine : la protection repose sur la gate de
  données, la gate de promotion, la CI (tests, test de fumée), staging, puis 10 % du trafic.
  Un lot aux étiquettes corrompues n'a pas été testé ici. Un modèle identique au champion est
  promu (règle `<=`) : une règle de gain strict, ou une revue humaine, éviterait des
  promotions inutiles.
- **Retour arrière.** Trois niveaux : la part du canary à 0 (un commit, la révision précédente
  est restée chaude), `gitops rollback` (qui restaure aussi `@champion`, grâce au tag
  `previous_champion`), et le `git revert` de l'historique de `k8s/overlays/prod`.
- **Secrets.** Le jeton du webhook et les accès S3 sont dans `.env` (ignoré par Git), lisibles
  par tout processus de mon poste. La CI n'utilise que le `GITHUB_TOKEN` éphémère. Le mot de
  passe initial d'Argo CD est lisible dans le Secret `argocd-initial-admin-secret`. Le bot
  pousse sur `main` avec mes droits Git : ses commits sont signés de mon nom.

## 5. Différences avec la prod

- **Données.** Le lot « récent » est le jeu d'origine dont trois variables sont multipliées
  par 1,5 : pas de vraies données récentes et étiquetées, et la gate évalue les deux modèles
  sur un lot qui ressemble à celui de l'entraînement.
- **Secrets et accès.** Secrets managés (Vault, External Secrets) à la place de `.env`, et un
  bot dans le cluster avec un jeton limité au dépôt (GitHub App) plutôt que mes droits.
- **Revue humaine.** Une pull request ouverte par le bot, avec le rapport de la gate, au lieu
  d'un commit direct sur `main` ; environnements staging et prod dans des clusters séparés.
- **Analyse du canary.** 50 prédictions sans erreur est une analyse naïve : il faudrait comparer
  erreurs, latence et distribution des scores entre révisions (Prometheus, Lab 7).
- **Synchronisation.** Argo CD interroge Git toutes les 60 s ; un webhook GitHub la rendrait
  immédiate.
- **Exploitation.** Sauvegarde et reprise après sinistre, audit des déploiements, supervision
  du webhook (ici un processus de mon poste), ressources dimensionnées (7,6 Go de RAM ici).

## 6. Captures d'écran

![Argo CD : Applications Synced et Healthy](m6/argocd.png)

![Run CT réussi dans KFP, avec le rapport de la gate](m6/kfp-run.png)

![Onglet Actions : runs CI scoring verts](m6/actions.png)




Canary observé en prod (`kubectl get isvc scoring-model -n scoring`) :

    NAME            READY   PREV   LATEST   PREVROLLEDOUTREVISION           LATESTREADYREVISION
    scoring-model   True    90     10       scoring-model-predictor-00004   scoring-model-predictor-00005

Répartition de 200 requêtes : 176 réponses en `2.0.0` et 24 en `2.1.7`.
