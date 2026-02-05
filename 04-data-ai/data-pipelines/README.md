# AUREUS Data Pipeline - Sistema Completo de Processamento de Dados

## Visao Geral

O AUREUS Data Pipeline e um sistema completo de processamento de dados em tempo real e batch para o AUREUS Core Banking Platform. Ele inclui componentes para analytics, machine learning, compliance LGPD e auditoria de dados.

## Arquitetura

- **PostgreSQL** (OLTP) -> **ClickHouse** (OLAP), **TimescaleDB** (Time-series)
- **Apache Kafka** (Streaming), **Elasticsearch** (Search), **Redis** (Cache)
- **Apache Spark** (Batch), **Apache Flink** (Streaming), **ML Pipeline**

## Componentes

1. **Apache Spark** - `data-pipelines/spark/` - Processamento batch
2. **Apache Flink** - `data-pipelines/flink/` - Processamento em tempo real
3. **ClickHouse** - `data-platform/clickhouse/` - Analytics OLAP
4. **TimescaleDB** - `data-platform/timescaledb/` - Time-series
5. **Sincronizacao** - `data-pipelines/sync/` - PostgreSQL e ClickHouse
6. **Analytics** - `data-pipelines/analytics/` - Dashboards tempo real
7. **Machine Learning** - `ml/models/`
8. **Compliance** - `data-pipelines/compliance/` - LGPD e auditoria

## Instalacao

- Linux/Mac: `chmod +x scripts/*.sh` e `./scripts/start-data-pipeline.sh`
- Windows: `scripts\start-data-pipeline.bat`

## Documentacao adicional

- [Arquitetura](../arquitetura/visao-geral.md)
- [Banco de dados](../banco-dados/README.md)

---

**AUREUS Core Banking Platform** - O padrao de excelencia financeira
