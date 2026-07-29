# Checklist – Data Lakehouse

Verificação e passos para o Data Lakehouse AURIX (Airflow, Trino, MinIO, dbt, DAGs). A stack já inclui os serviços no `infra/data-stack/docker-compose.yml`; este checklist garante configuração e primeiro uso.

---

## Infraestrutura

- [ ] **Airflow** no compose: serviços `airflow-init`, `airflow-webserver` (porta 8082), `airflow-scheduler`; banco Airflow no Postgres (script `init-scripts/07-create-airflow-db.sql` na primeira criação do volume).
- [ ] **Buckets MinIO**: `aurix-bronze`, `aurix-silver`, `aurix-gold` criados (script `infra/data-stack/scripts/create-minio-buckets.sh` ou manualmente no Console :9001).
- [ ] **Iceberg REST Catalog**: serviço `iceberg-rest` na porta 8181 (compose).
- [ ] **Trino**: serviço na porta 8090; config em `infra/data-stack/trino/` (postgresql, clickhouse, iceberg).

---

## Banco Airflow {#banco-airflow}

Se o Postgres já existia antes de adicionar o init script `07-create-airflow-db.sql`, criar manualmente:

```sql
CREATE DATABASE airflow;
CREATE USER airflow WITH PASSWORD 'airflow';
GRANT ALL PRIVILEGES ON DATABASE airflow TO airflow;
ALTER DATABASE airflow OWNER TO airflow;
```

---

## Airflow – Connections e variáveis

- [ ] Acessar Airflow UI: http://localhost:8082 (usuário/senha `airflow` / `airflow`).
- [ ] Criar connections conforme [AIRFLOW-DBT-CONNECTIONS.md](../../04-data-ai/data-pipelines/AIRFLOW-DBT-CONNECTIONS.md) (postgres_default, minio_s3, clickhouse_default).
- [ ] Opcional: variáveis `PG_BRONZE_URL`, `MINIO_ENDPOINT`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`, `BRONZE_BUCKET`.

---

## Ingestão e DAGs

- [ ] DAG **ingest_postgres_to_bronze** visível no Airflow; rodar uma vez manualmente (após buckets criados).
- [ ] DAG **bronze_to_silver_dbt** visível; roda `dbt run` e `dbt test` em `/opt/airflow/dbt`.
- [ ] Projeto dbt em `data-pipelines/dbt/` com `profiles.yml` usando variáveis de ambiente (ou connection); schemas/tabelas existentes no Postgres para os models.

---

## dbt (local ou via Airflow)

- [ ] Variáveis de ambiente para dbt (ou connection): `DBT_POSTGRES_HOST`, `DBT_POSTGRES_PORT`, `DBT_POSTGRES_USER`, `DBT_POSTGRES_PASSWORD`, `DBT_POSTGRES_DB`, `DBT_POSTGRES_SCHEMA`.
- [ ] Local: `cd data-pipelines/dbt && dbt run --target prod && dbt test --target prod`.

---

## Consulta (Trino)

- [ ] Trino UI ou CLI: http://localhost:8090 (ou `trino --server http://localhost:8090`).
- [ ] Catalogos disponíveis: `postgresql` (schema aurix), `clickhouse`, `iceberg` (quando warehouse e tabelas existirem).

---

## Opcional (evolução)

- [ ] DAG **ingest_kafka_to_bronze**: consumir Kafka e gravar em Bronze (Parquet/Iceberg).
- [ ] DAG **sync_gold_to_clickhouse**: materializar agregados no ClickHouse para dashboards.
- [ ] Backend publicando eventos em Kafka (já implementado em core/pix); bacen consumir para relatórios.
- [ ] OpenLineage e MLflow lendo do lakehouse.

Referência geral: [roadmap.md](../../../01-business/roadmap.md) (seção 6).

[Voltar aos checklists](README.md) | [Índice da wiki](../../README.md)
