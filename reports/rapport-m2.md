# Rapport comparatif — Kubeflow Pipelines vs Airflow (Lab 2)

**Binôme :** Dorra Trabelsi · **Date :** 10/11/2026 · **Machine :** Intel Core i7-13620H, 7,6 Go de RAM, Windows + WSL2 (Ubuntu) ·
**Dataset :** german-credit

## 1. Contexte

Nous avons migré le pipeline DVC du Lab 1 (scoring de crédit, 1000 lignes) vers un pipeline de 6 steps : ingest, validate, prepare, train, evaluate, register. Les deux orchestrateurs lancent chaque step dans un pod du cluster kind `mlops-lab`, avec la même image « projet » `scoring-pipeline:1.0.0` : le code de `src/` est réutilisé, jamais recopié. Sont partagés : l'image, `dataset.yaml`, `params.yaml`, MLflow (runs taggés `launched_by=kfp` ou `airflow`) et le Model Registry. Seule l'orchestration diffère.

## 2. Résultats quantitatifs

| Mesure | KFP | Airflow |
|---|---|---|
| Installation de zéro (min) | non mesuré | non mesuré |
| Temps jusqu'au premier run réussi (min) | non mesuré | non mesuré |
| Durée d'un run complet (min) | non mesuré | non mesuré |
| Relance avec les mêmes paramètres (min) | non mesuré | non mesuré |
| Hot-reload : modifier un step et voir le résultat (min) | non mesuré | non mesuré |
| Lignes de code propres à l'orchestrateur | 282 (components.py 193 + scoring_pipeline.py 89) | 69 (scoring_dag.py) |
| Mémoire consommée par l'outil (Go, `docker stats`) | ≈ 4,3 (cluster kind entier : KFP + MLflow + S3) | non mesuré |

## 3. Analyse qualitative

### Developer experience

KFP fournit des artefacts typés (Dataset, Model, Metrics), des métriques et une matrice de confusion dans l'UI, et un lineage visible en un clic. Le prix : 282 lignes, et chaque modification du pipeline demande de recompiler puis d'uploader une version. Le DAG Airflow est plus court (69 lignes) et lisible, mais Airflow ne transporte aucun fichier : il a fallu écrire la seconde moitié de `pipelines/stage.py` (dossier de travail S3, XCom), que KFP offre nativement.

### Scaling

Dans les deux cas, 1 step = 1 pod : le passage à l'échelle repose sur Kubernetes. KFP apporte le parallélisme natif et un cache basé sur le hash des entrées. Airflow sait lancer des tasks en parallèle mais n'a pas de cache par step. Ici Airflow tourne hors du cluster (Docker Compose : Postgres, Redis, scheduler, worker), ce qui ajoute une charge mémoire que le sujet estime à environ 4 Go.

### Debugging

Dans les deux outils, les logs d'un step sont accessibles dans l'UI. Notre incident le plus instructif a eu lieu sur KFP : tous les deployments du namespace `kubeflow` étaient à 0 réplique, `kubectl wait --for=condition=available` répondait « condition met » alors que rien ne tournait, et l'API était injoignable (endpoints `<none>`). Il a fallu `kubectl get deploy` et `get events` pour comprendre. Une plateforme de 14 composants offre plus de surface de panne qu'un DAG Airflow.

## 4. Recommandation

Choisir KFP pour une équipe orientée ML, avec des artefacts volumineux à typer et à tracer, un besoin de cache et de lineage, et un cluster Kubernetes dédié avec au moins 6 Go de RAM. Choisir Airflow quand l'orchestration dépasse le ML (ETL, planification, dépendances entre systèmes), quand l'équipe le maîtrise déjà, et que les artefacts passent par un stockage objet géré explicitement.

## 5. Limites de la comparaison

Un seul dataset de 1000 lignes, des durées mesurées sur un seul run, et un cluster kind de 3 nœuds sur 7,6 Go de RAM : ce lab ne dit rien du vrai passage à l'échelle. Airflow tournait en Docker Compose avec CeleryExecutor, pas avec le KubernetesExecutor, et KFP n'a pas été testé sur un cluster managé (GKE). La mémoire de KFP inclut MLflow et S3, donc elle est surestimée. Ni la sécurité ni le multi-utilisateur n'ont été évalués.
