# Airflow connections e perfis dbt

Checklist de configuracao do Data Lakehouse (buckets, banco Airflow, DAGs): [docs/wiki/checklist-data-lakehouse.md](../wiki/checklists/data-lakehouse.md).

## Airflow (Admin > Connections)

Criar as seguintes connections no Airflow (UI em http://localhost:8082, usuario/senha `airflow`/`airflow`):

| Connection Id   | Type    | Host     | Schema/Extra | Login        | Password            | Port |
|----------------|---------|----------|--------------|--------------|---------------------|------|
| postgres_default | Postgres | postgres | aureus       | aureus_user  | aureus_secure_password | 5432 |
| minio_s3       | Generic | http://minio:9000 | -   | aureus_admin | aureus_secure_password | -    |
| clickhouse_default | HTTP  | clickhouse | -        | aureus_analytics | aureus_analytics_password | 8123 |

Para o DAG `ingest_postgres_to_bronze` nao e obrigatorio ter connection cadastrada; o codigo usa variaveis de ambiente. Opcionalmente defina:

- **Variable** `PG_BRONZE_URL`: `postgresql+psycopg2://aureus_user:aureus_secure_password@postgres:5432/aureus`
- **Variable** `MINIO_ENDPOINT`: `http://minio:9000`
- **Variable** `MINIO_ACCESS_KEY`: `aureus_admin`
- **Variable** `MINIO_SECRET_KEY`: `aureus_secure_password`
- **Variable** `BRONZE_BUCKET`: `aureus-bronze`

## dbt (profiles.yml)

O projeto dbt em `data-pipelines/dbt/` usa `profiles.yml` com variaveis de ambiente para nao versionar senhas:

- `DBT_POSTGRES_HOST` (default: localhost)
- `DBT_POSTGRES_PORT` (default: 5432)
- `DBT_POSTGRES_USER` (default: aureus_user)
- `DBT_POSTGRES_PASSWORD` (default: aureus_secure_password)
- `DBT_POSTGRES_DB` (default: aureus)
- `DBT_POSTGRES_SCHEMA` (default: aureus)

Em producao, defina essas variaveis no secret manager ou no ambiente do executor (Airflow, CI).

Para rodar dbt localmente:

```bash
cd data-pipelines/dbt
export DBT_POSTGRES_HOST=localhost DBT_POSTGRES_PASSWORD=aureus_secure_password
dbt run --target prod
dbt test --target prod
```

## Buckets MinIO

Antes do primeiro DAG de ingestao, crie os buckets no MinIO (Console http://localhost:9001 ou script):

```bash
cd infrastructure/data-stack/scripts
./create-minio-buckets.sh
```

Ou manualmente: buckets `aureus-bronze`, `aureus-silver`, `aureus-gold`.

## Banco Airflow

Se o Postgres ja existia antes de adicionar o init script `07-create-airflow-db.sql`, crie o banco e usuario manualmente:

```sql
CREATE DATABASE airflow;
CREATE USER airflow WITH PASSWORD 'airflow';
GRANT ALL PRIVILEGES ON DATABASE airflow TO airflow;
ALTER DATABASE airflow OWNER TO airflow;
```
