# Rapport — Lab 4 : packaging du modèle

Travail individuel · Dataset : german-credit · Modèle : `scoring-model` v13

## 1. Comparatif des tailles

| Image                   | Taille  | CVE HIGH + CRITICAL                  |
| ----------------------- | ------- | ------------------------------------ |
| `scoring-api:naive`     | 2.21 Go | 502 (487 HIGH, 15 CRITICAL)          |
| `scoring-api:optimized` | 402 Mo  | 48 (48 HIGH, 0 CRITICAL)             |
| `scoring-api:bentoml`   | 1.41 Go | 145 (145 HIGH, 0 CRITICAL)           |

Scans réalisés avec Trivy 0.75.0 (sévérités HIGH et CRITICAL). L'image optimisée est passée de 535 Mo à 402 Mo après deux ajustements : `uvicorn` sans l'extra `[standard]` (−16 Mo d'uvloop) et un `pip install --no-compile` suivi de la suppression des `__pycache__` et dossiers `tests/` (`/deps` : 276 Mo → 163 Mo).

## 2. Comparatif des latences

1000 appels par backend après 10 appels de chauffe, x86_64, Python 3.12.16.

| Backend          | P50 (ms) | P95 (ms) | P99 (ms) |
| ---------------- | -------- | -------- | -------- |
| sklearn_pipeline | 3.047    | 4.046    | 5.170    |
| onnx_python      | 0.054    | 0.081    | 0.142    |
| onnx_batch_100   | 0.231    | 0.343    | 0.506    |

ONNX Runtime est 56.9 fois plus rapide que le pipeline scikit-learn en prédiction unitaire (P50).

## 3. Interprétation

- **Taille de l'image optimisée.** Elle part de `python:3.12-slim` (87 paquets Debian) au lieu de `python:3.12` complet (469 paquets dans la naive). Le build multi-stage n'embarque que `/deps` et le code utile, sans pip ni cache de build. L'image n'embarque ni MLflow, ni scikit-learn, ni skl2onnx, seulement ONNX Runtime, FastAPI, numpy et pandas. La liste blanche du `.dockerignore` limite le contexte envoyé au démon.
- **Rapidité d'ONNX.** Pour une seule ligne, le pipeline scikit-learn est dominé par un surcoût fixe : validation du DataFrame, sélection des colonnes par nom, un appel Python par transformateur. Le graphe ONNX contient le prétraitement et le modèle, et s'exécute dans un moteur C++ optimisé, d'où 0.054 ms contre 3.047 ms. Le batch de 100 lignes (0.231 ms) montre que le coût par ligne est ensuite très faible.
- **Scans.** Trivy montre que le nombre de CVE suit le nombre de paquets embarqués : 502 pour la naive, 145 pour BentoML, 48 pour l'optimisée. Les CVE viennent surtout de paquets Debian inutiles au serving. ModelScan ne trouve rien sur l'ONNX, mais ne le lit pas (fichier ignoré) : il est sûr par construction, car c'est un graphe protobuf déclaratif sans code exécutable. Sur le pickle piégé, il détecte bien l'appel `posix.system` (CRITICAL). Le modèle skops du Registry est ignoré par ModelScan et skops ne lui demande que le type `numpy.dtype`. À noter : ModelScan 0.8.8 affiche des erreurs `_check_version` sur le skops avec notre version de numpy, sans conséquence sur le verdict.
- **Sécurité de l'image naive.** Elle copie tout le repo (`.git`, `.env` avec les identifiants S3) et tourne en root. L'image optimisée n'embarque que cinq fichiers utiles et tourne avec l'utilisateur 10001.

## 4. Choix recommandé

L'image `scoring-api:optimized` (FastAPI + ONNX Runtime), publiée sous `localhost:5001/scoring-api:1.0.0`. Elle est la plus petite (402 Mo) et la moins vulnérable (48 CVE HIGH, aucune CRITICAL). Elle tourne en non-root avec un `HEALTHCHECK`, des labels qui la relient à la version du modèle dans le Registry MLflow, et n'utilise aucun pickle. Sa latence est environ 57 fois plus faible que celle d'un pipeline scikit-learn pour une requête unitaire, ce qui compte pour l'autoscaling. La variante BentoML est pratique (service typé, packaging intégré) mais pèse 1.41 Go, embarque 145 CVE HIGH, exécute le pipeline scikit-learn (modèle sérialisé en pickle) et renvoie 400 au lieu de 422 sur une requête invalide.

## 5. Pistes d'amélioration

- Mettre à jour régulièrement l'image de base et épingler la base par digest, puis intégrer Trivy à la CI (Module 6) avec `--exit-code 1` pour bloquer sur les CVE CRITICAL.
- Passer à une image distroless ou Chainguard pour supprimer les 48 CVE Debian restantes.
- Quantization INT8 : peu utile ici (le modèle ONNX fait 11 ko pour une régression logistique), mais pertinente pour un modèle plus gros.
- Retirer pandas des dépendances de serving (≈ 45 Mo) en réécrivant la lecture de `dataset.yaml` sans lui.
- Précharger le modèle dans un Init Container ou un volume, pour pouvoir mettre à jour le modèle sans reconstruire l'image.
- Signer l'image (cosign) et publier un SBOM, puis servir avec plusieurs workers uvicorn pour mieux absorber la charge.
