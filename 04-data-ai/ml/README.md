# AUREUS ML

Diretorio unificado de Machine Learning: modelos e MLOps do AUREUS Core Banking.

## Estrutura

- **models/** - Implementacoes dos modelos (deteccao de fraude, scoring de credito). Ver [models.md](./models.md).
- **ops/** - Treinamento, registro (MLflow), API de predicao, drift e metricas. Ver [ops.md](./ops.md).

## Uso rapido

Da raiz do repo:

```bash
cd ml/ops
pip install -r requirements.txt
python -m pipelines.train_pipeline --config config/config.yaml --model-dir models --no-mlflow
./scripts/run_serve.sh
```

API de predicao: http://localhost:8000 (health: `/health`, predicao: `POST /predict`).

## Convencoes

- Modelos ficam em `ml/models/`; artefatos treinados (.pkl) em `ml/ops/models/`.
- Pipelines, config e scripts ficam em `ml/ops/`.
- Build Docker do serving: na raiz do repo, `docker build -f ml/ops/serving/Dockerfile .`
