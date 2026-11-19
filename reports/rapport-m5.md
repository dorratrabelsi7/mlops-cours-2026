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

Le run 100 a tourné avec un processus Locust, les runs 500 et 1000 avec quatre. Les rares erreurs du run 1000 sont des 502 (33 sur 216 670). Ces trois runs ont été faits à la suite, sur la révision 1.0.0, juste après un démarrage à froid.

Run supplémentaire « à chaud », sur la révision 2.0.0 (1 pod au départ), 1000 utilisateurs pendant 3 minutes :

| Run | Requêtes | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Erreurs (%) |
|---|---|---|---|---|---|---|
| warm_1000 | 427712 | 2432.0 | 210 | 300 | 350 | 0.00 |

Point de saturation : sur les trois premiers runs, le débit n'augmente que de 3 % entre 500 et 1000 utilisateurs (1195 → 1227 req/s) alors que le P50 triple et que le P99 passe de 740 ms à 1,8 s : la charge supplémentaire ne fait qu'allonger les files d'attente. Mais le run à chaud atteint 2432 req/s avec le même nombre de pods maximum (5), sans erreur, et un P99 de 350 ms. Deux autres runs de 500 utilisateurs (hors CSV) ont fait 1969 et 2143 req/s. Le plateau de 1200 req/s n'est donc pas une limite fixe du service : il dépend de l'état du système au moment du test (révision, démarrage à froid, charge de la machine). La cause exacte n'a pas été identifiée.

## 3. Scale-up observé

Source : événements Kubernetes du namespace `scoring` (heure UTC). Les trois runs d'origine ont eu lieu sans metrics-server (`Metrics API not available`) : leur CPU n'a pas été mesuré. Il l'a été ensuite lors d'un run de contrôle équivalent au run à chaud (500 utilisateurs, 90 s, révision 2.0.0, 5 pods : 182 009 requêtes, 2075 req/s, P50 21 ms, P99 310 ms, 0 erreur), avec un relevé `kubectl top` toutes les 10 s.

| Run | Pods (max observé) | CPU moyen par pod | Temps pour atteindre ce nombre de pods |
|---|---|---|---|
| run_100 | 5 | non mesuré (metrics-server absent) | ≈ 26 s (2 pods à 5 s, 3 à 7 s, 4 à 12 s) |
| run_500 | 5 | non mesuré (metrics-server absent) | ≈ 9 s (départ à 2 pods) |
| run_1000 | 5 | non mesuré (metrics-server absent) | déjà à 5 pods, aucun nouvel événement de montée |
| run à chaud (500 utilisateurs, révision 2.0.0) | 5 | ≈ 915 m (≈ 92 % de la limite d'1 CPU, mesure du run de contrôle) | ≈ 7 s (départ à 1 pod : 4 pods à 5 s, 5 à 7 s) |

Dès le run 100 (≈ 460 req/s pour une cible effective de 140 req/s par pod), l'autoscaler monte jusqu'au maximum de 5 pods, au-delà de ce que le débit moyen exige : la réaction à la montée brutale est forte. La réduction commence avant la fin du run (5→4 à 12 s de la fin) puis se poursuit par paliers (4→3 à +38 s, 3→2 à +55 s après la fin).

Scale-to-zero : avant les tests, le service était à 0 pod (`NoTraffic`). Le premier événement de retour à 0 après le run 1000 date de ≈ 90 s après sa fin ; la valeur est indicative, Kubernetes agrège ces événements. Démarrage à froid : 9,24 s pour le premier appel, 0,006 s pour le suivant.

## 4. Analyse canary

- Répartition observée sur 200 requêtes : 185 × 1.0.0 / 15 × 2.0.0.
- Le split est cohérent avec 90/10 : l'attendu est 180/20, soit un écart de 5 requêtes. L'écart-type d'un tirage binomial à 10 % sur 200 requêtes est d'environ 4,2 : la valeur observée est à environ 1,2 écart-type de l'attendu, donc normale.
- Décision : promotion à 100 %. La v2 contient le même modèle (v13, même seuil) et la même logique de prédiction, la seule différence est la route `/version`, qui répond `2.0.0`. Cette décision est manuelle (`kubectl patch`), prise sans comparaison automatique des métriques entre versions.

## 5. Points d'amélioration

- Cible de l'autoscaler : le maximum de 5 pods est atteint dès 460 req/s ; une cible plus haute, ou la métrique de concurrence, éviterait de sur-dimensionner. Il faudrait aussi tester un `maxReplicas` plus élevé, puisque 5 pods ont tenu 2400 req/s à chaud.
- Plusieurs workers uvicorn par pod : l'image n'en lance qu'un, c'est le goulot probable d'un pod.
- `minReplicas: 1` en production : le démarrage à froid de 9 s est trop long pour un parcours de demande de crédit.
- CPU : sous charge, chaque pod tourne à 86-97 % de sa limite d'1 CPU (≈ 915 m en moyenne, ≈ 415 req/s par pod), alors que la mémoire reste à ≈ 100-135 Mi. Le worker uvicorn unique est donc saturé : augmenter le nombre de workers et la limite CPU, ou baisser la cible de l'autoscaler, serait plus efficace que de monter `maxReplicas`.
- Batching des requêtes (l'API accepte jusqu'à 1000 demandes par appel) et automatisation du canary (Modules 6 et 7).
- Comprendre pourquoi le débit varie du simple au double (1200 contre 2400 req/s) selon l'état du système.

## 6. Comparatif des serveurs

Triton n'a pas été testé (image trop lourde pour la machine, étape optionnelle) ; le bonus batch n'a pas été réalisé. Mesure à 20 utilisateurs pendant 1 minute, un pod chacun.

| Serveur | Protocole | P50 (ms) | P95 (ms) | P99 (ms) | Image | Mémoire du pod |
|---|---|---|---|---|---|---|
| FastAPI + ONNX Runtime (`scoring-model`) | V1 `/predict` | 4 | 9 | 17 | 402 Mo | non mesurée |
| BentoML + scikit-learn (`scoring-bento`) | V1 `/predict` | 11 | 26 | 70 | 1,41 Go | non mesurée |
| Triton + ONNX Runtime (`scoring-triton`) | V2 | non testé | non testé | non testé | non testé | non testé |

Débits quasi identiques (100,3 et 95,8 req/s, 0 erreur), car la charge est la même. Sur une requête invalide, FastAPI répond 422 et BentoML 400.