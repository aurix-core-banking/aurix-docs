# Checklist – Plataforma de pé

Passos para colocar a plataforma AUREUS de pé (primeira vez ou ambiente novo), na ordem recomendada.

---

## 1. Pré-requisitos

- [ ] **Docker** 20.10+ e **Docker Compose** 2.0+
- [ ] **Java 25** e **Maven** (para backend)
- [ ] **Node 18+** e **npm** (para frontend)
- [ ] **Python 3.10+** (para data-pipelines, E2E, ml)
- [ ] **Git** e acesso ao repositório
- [ ] **8GB+ RAM** e **20GB+ disco** para a stack de dados

---

## 2. Stack de dados (PostgreSQL, Kafka, Redis, etc.)

- [ ] Entrar na pasta: `cd infrastructure/data-stack`
- [ ] Subir a stack: `docker-compose up -d`
- [ ] Verificar serviços: `docker-compose ps` (postgres, redis, kafka, clickhouse, minio, etc.)
- [ ] Se o Postgres já existia antes do script de Airflow: criar DB e usuário Airflow (ver [data-lakehouse.md](data-lakehouse.md#banco-airflow))
- [ ] Criar buckets MinIO: executar `scripts/create-minio-buckets.sh` ou criar manualmente no Console (porta 9001) os buckets `aureus-bronze`, `aureus-silver`, `aureus-gold`

Referência: [Arquitetura de dados](../../../02-technical/arquitetura/evolucao-arquitetura-dados.md).

---

## 3. Backend

- [ ] Na raiz: `cd backend && mvn clean install -DskipTests`
- [ ] Configurar `application.yml` de cada módulo (ou usar defaults) com URL do Postgres: `jdbc:postgresql://localhost:5432/aureus`, user `aureus_user`, password `aureus_secure_password`
- [ ] Configurar Kafka (se usar eventos): bootstrap `localhost:9092`
- [ ] Subir os serviços necessários (ex.: gateway, core, pix, credit) ou usar script: `infrastructure/scripts/start-aureus.sh` / `start-aureus.bat`
- [ ] Validar health: ex. `GET http://localhost:8080/actuator/health` (ajustar porta conforme gateway)

---

## 4. Frontend

- [ ] **aureus-admin**: `cd frontend/aureus-admin && npm install && npm start` (ver [Instalação Aureus Admin](../guias/instalacao-admin.md))
- [ ] **aureus-web**: `cd frontend/aureus-web && npm install && npm start` (se existir)
- [ ] Configurar URL da API (env ou config) apontando para o gateway

---

## 5. Airflow e Data Lakehouse (opcional na primeira subida)

- [ ] Stack de dados já inclui Airflow (webserver porta 8082, scheduler); aguardar subir após `docker-compose up -d`
- [ ] Acessar Airflow UI: http://localhost:8082 (usuário/senha `airflow` / `airflow`)
- [ ] Configurar connections e variáveis conforme [AIRFLOW-DBT-CONNECTIONS.md](../../04-data-ai/data-pipelines/AIRFLOW-DBT-CONNECTIONS.md)
- [ ] Rodar DAG `ingest_postgres_to_bronze` manualmente uma vez (após buckets MinIO criados) para popular Bronze
- [ ] Trino: http://localhost:8090 (consultas SQL sobre Postgres, ClickHouse, Iceberg)

Ver [data-lakehouse.md](data-lakehouse.md) para detalhes.

---

## 6. Testes rápidos

- [ ] Backend: `cd backend && mvn test`
- [ ] E2E (se configurado): `pip install -r tests/e2e/requirements.txt` e `infrastructure/scripts/run-e2e-tests.bat` (ou `.sh`)

Ver [testes/e2e.md](../../03-development/testes/e2e.md).

---

## 7. Próximos passos

- Para **go live de um novo cliente**: usar [go-live.md](go-live.md).
- Para **atualização regulatória** (relatórios BACEN/Receita): usar [regulatorio.md](regulatorio.md).

[Voltar aos checklists](README.md) | [Índice da wiki](../../README.md)
