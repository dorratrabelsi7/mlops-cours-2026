# Benchmark de latence — german-credit, modèle v13

1000 appels par backend, après 10 appels de chauffe · x86_64 · Python 3.12.16

| Backend | P50 (ms) | P95 (ms) | P99 (ms) |
|---|---|---|---|
| sklearn_pipeline | 3.047 | 4.046 | 5.170 |
| onnx_python | 0.054 | 0.081 | 0.142 |
| onnx_batch_100 | 0.231 | 0.343 | 0.506 |

Accélération ONNX vs scikit-learn (P50, 1 ligne) : x56.9
