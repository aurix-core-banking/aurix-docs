# Airflow connections e perfis dbt

Checklist de configuracao do Data Lakehouse (buckets, banco Airflow, DAGs): [docs/wiki/checklist-data-lakehouse.md](../wiki/checklists/data-lakehouse.md).

## Airflow (Admin > Connections)

Criar as seguintes connections no Airflow (UI em http://localhost:8082, usuario/senha `airflow`/`airflow`):

| Connection Id   | Type    | Host     | Schema/Extra | Login        | Password            | Port |
|----------------|---------|----------|--------------|--------------|---------------------|------|
| postgres_default | Postgres | postgres | aurix       | aurix_user  | aurix_secure_password | 5432 |
| minio_s3       | Generic | http://minio:9000 | -   | aurix_admin | aurix_secure_password | -    |
| clickhouse_default | HTTP  | clickhouse | -        | aurix_analytics | aurix_analytics_password | 8123 |

Para o DAG `ingest_postgres_to_bronze` nao e obrigatorio ter connection cadastrada; o codigo usa variaveis de ambiente. Opcionalmente defina:

- **Variable** `PG_BRONZE_URL`: `postgresql+psycopg2://aurix_user:aurix_secure_password@postgres:5432/aurix`
- **Variable** `MINIO_ENDPOINT`: `http://minio:9000`
- **Variable** `MINIO_ACCESS_KEY`: `aurix_admin`
- **Variable** `MINIO_SECRET_KEY`: `aurix_secure_password`
- **Variable** `BRONZE_BUCKET`: `aurix-bronze`

## dbt (profiles.yml)

O projeto dbt em `data-pipelines/dbt/` usa `profiles.yml` com variaveis de ambiente para nao versionar senhas:

- `DBT_POSTGRES_HOST` (default: localhost)
- `DBT_POSTGRES_PORT` (default: 5432)
- `DBT_POSTGRES_USER` (default: aurix_user)
- `DBT_POSTGRES_PASSWORD` (default: aurix_secure_password)
- `DBT_POSTGRES_DB` (default: aurix)
- `DBT_POSTGRES_SCHEMA` (default: aurix)

Em producao, defina essas variaveis no secret manager ou no ambiente do executor (Airflow, CI).

Para rodar dbt localmente:

```bash
cd data-pipelines/dbt
export DBT_POSTGRES_HOST=localhost DBT_POSTGRES_PASSWORD=aurix_secure_password
dbt run --target prod
dbt test --target prod
```

## Buckets MinIO

Antes do primeiro DAG de ingestao, crie os buckets no MinIO (Console http://localhost:9001 ou script):

```bash
cd infrastructure/data-stack/scripts
./create-minio-buckets.sh
```

Ou manualmente: buckets `aurix-bronze`, `aurix-silver`, `aurix-gold`.

## Banco Airflow

Se o Postgres ja existia antes de adicionar o init script `07-create-airflow-db.sql`, crie o banco e usuario manualmente:

```sql
CREATE DATABASE airflow;
CREATE USER airflow WITH PASSWORD 'airflow';
GRANT ALL PRIVILEGES ON DATABASE airflow TO airflow;
ALTER DATABASE airflow OWNER TO airflow;
```
