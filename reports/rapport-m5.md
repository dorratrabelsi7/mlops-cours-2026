# Rapport — Lab 5 : serving avec KServe

Travail individuel · Dataset : german-credit · Modèle : `scoring-model` v13 ·
Seuil de décision servi : 0.3182

## 1. Résumé

L'image FastAPI + ONNX Runtime du Lab 4 est servie par KServe 0.21 en mode Knative : un InferenceService `scoring-model` (namespace `scoring`) exposé par la passerelle Kourier sur le port 30080, routé par l'en-tête `Host`. L'autoscaler Knative (KPA) pilote le nombre de pods sur le débit (cible de 200 requêtes/s par pod, de 0 à 5 pods). La version en production à la fin du lab est la 2.0.0 (même modèle v13), promue à 100 % après un canary 90/10 ; pendant le canary, `minReplicas` est passé à 1.

## 2. Benchmark de latence

| Run | Requêtes | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Erreurs (%) |
|---|---|---|---|---|---|---|
| run_100 | 53051 | 457.2 | 10 | 67 | 140 | 0.00 |
| run_500 | 211160 | 1194.9 | 200 | 330 | 740 | 0.00 |
| run_1000 | 216670 | 1226.9 | 590 | 870 | 1800 | 0.02 |

Le run 100 a tourné avec un processus Locust, les runs 500 et 1000 avec quatre. Les rares erreurs du run 1000 sont des 502 (33 sur 216 670).

Point de saturation : entre 500 et 1000 utilisateurs, le débit n'augmente que de 3 % (1195 → 1227 req/s) alors que le P50 triple et que le P99 passe de 740 ms à 1,8 s : la charge supplémentaire ne fait qu'allonger les files d'attente. Cependant, deux runs de 500 utilisateurs réalisés ensuite (hors CSV, sur la révision 2.0.0, 5 pods) ont atteint 1969 et 2143 req/s sans erreur, avec un P50 de 12 et 22 ms. Ce plateau de 1200 req/s n'est donc pas une limite fixe du service : il dépend de l'état de l'ensemble au moment du test. La cause n'a pas été identifiée.

## 3. Scale-up observé

Source : événements Kubernetes du namespace `scoring` (heure UTC). Le CPU n'a pas pu être mesuré (metrics-server indisponible : `Metrics API not available`).

| Run | Pods (max observé) | CPU moyen par pod | Temps pour atteindre ce nombre de pods |
|---|---|---|---|
| run_100 | 5 | non mesuré | ≈ 26 s (2 pods à 5 s, 4 pods à 12 s) |
| run_500 | 5 | non mesuré | ≈ 9 s (départ à 2 pods) |
| run_1000 | 5 | non mesuré | déjà à 5, aucun nouvel événement de montée |

Dès le run 100 (≈ 460 req/s pour une cible de 140 req/s par pod), l'autoscaler va jusqu'au maximum de 5 pods : le mode « panique » de Knative semble réagir fortement à la montée brutale. Après le run, la réduction se fait par paliers (5→4 puis 3 puis 2 en ≈ 70 s). Sur un run à chaud (révision 2.0.0, 1 pod au départ), 5 pods sont atteints en 6 s.

Scale-to-zero : avant les tests, le service était à 0 pod (`NoTraffic`) ; le délai exact avant retour à 0 n'a pas été mesuré. Démarrage à froid : 9,24 s pour le premier appel, 0,006 s pour le suivant.

## 4. Analyse canary

- Répartition observée sur 200 requêtes : 185 × 1.0.0 / 15 × 2.0.0.
- Le split est cohérent avec 90/10 : l'attendu est 180/20, soit un écart de 5 requêtes. L'écart-type d'un tirage binomial à 10 % sur 200 requêtes est d'environ 4,2 : la valeur observée est à environ 1,2 écart-type de l'attendu, donc normale.
- Décision : promotion à 100 %. La v2 contient le même modèle (v13, même seuil) et la même logique de prédiction, la seule différence est la route `/version` : elle répond `2.0.0` et la révision passe `Ready`. Cette décision est manuelle (`kubectl patch`), prise sans comparaison automatique des métriques entre versions.

## 5. Points d'amélioration

- Cible de l'autoscaler : le maximum de 5 pods est atteint dès 460 req/s ; une cible plus haute, ou la métrique de concurrence, éviterait de sur-dimensionner. Il faudrait aussi tester un `maxReplicas` plus élevé, puisque 5 pods suffisent pour ≈ 2000 req/s à chaud.
- Plusieurs workers uvicorn par pod : l'image n'en lance qu'un, c'est le goulot probable d'un pod.
- `minReplicas: 1` en production : le démarrage à froid de 9 s est trop long pour un parcours de demande de crédit.
- Installer ou réparer metrics-server pour mesurer le CPU (non mesuré ici) et rendre les observations d'autoscaling complètes.
- Batching des requêtes (l'API accepte jusqu'à 1000 demandes par appel) et automatisation du canary (Modules 6 et 7).
- Comprendre pourquoi le débit mesuré varie du simple au double (1200 contre 2000 req/s) selon les runs.

## 6. Comparatif des serveurs

Triton n'a pas été testé (image trop lourde pour la machine, étape optionnelle) ; le bonus batch n'a pas été réalisé. Mesure à 20 utilisateurs pendant 1 minute, un pod chacun.

| Serveur | Protocole | P50 (ms) | P95 (ms) | P99 (ms) | Image | Mémoire du pod |
|---|---|---|---|---|---|---|
| FastAPI + ONNX Runtime (`scoring-model`) | V1 `/predict` | 4 | 9 | 17 | 402 Mo | non mesurée |
| BentoML + scikit-learn (`scoring-bento`) | V1 `/predict` | 11 | 26 | 70 | 1,41 Go | non mesurée |
| Triton + ONNX Runtime (`scoring-triton`) | V2 | non testé | non testé | non testé | non testé | non testé |

Débits quasi identiques (100,3 et 95,8 req/s, 0 erreur), car la charge est la même. Sur une requête invalide, FastAPI répond 422 et BentoML 400.
