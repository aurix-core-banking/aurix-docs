# Run – Execução e operação

O que fazer no dia a dia: subir/parar serviços, health checks, monitoramento, logs.

---

## Stack de dados

- **Subir**: `cd infrastructure/data-stack && docker-compose up -d`
- **Parar**: `docker-compose down`
- **Logs**: `docker-compose logs -f [servico]`
- **Portas**: Postgres 5432, Redis 6379, Kafka 9092, ClickHouse 8123, MinIO 9000/9001, Airflow 8082, Trino 8090, Kafka UI 8080

---

## Aplicação (backend/frontend)

- Scripts: `infrastructure/scripts/start-aurix.sh` ou `start-aurix.bat` (e `stop-aurix.*`).
- Health: `GET /actuator/health` ou `/health` em cada serviço; gateway como ponto único.

---

## Monitoramento e alertas

- **Prometheus**: métricas; config em `infrastructure/monitoring/prometheus.yml`.
- **Grafana**: dashboards (overview AURIX, Keycloak).
- **Alertas**: regras em `infrastructure/monitoring/prometheus/rules/sla-uptime.yml`; cada alerta referencia um procedimento do runbook.

**Doc**: [monitoramento-sla.md](../../05-infrastructure/infrastructure/monitoramento-sla.md).

---

## Runbook (resumo)

Escopo: deploy centralizado (K8s), pipeline CI/CD, secrets em secret manager, config por ambiente. Health em `/actuator/health`; métricas Prometheus (latência, erros, uso por tenant); logs com tenant_id e correlation_id.

**Alertas sugeridos**

- Críticos: serviço indisponível, taxa de erro alta, banco inacessível, disco/DB no limite.
- Altos: latência p95 acima do SLA, filas Kafka atrasadas, CPU/memória > 80%.
- Informativos: deploy concluído, novo tenant, limite de uso do tenant ~80%.

**Procedimentos (R1–R6)**

| Código | Situação | Ação resumida |
|--------|----------|----------------|
| R1 | Serviço não sobe após deploy | Logs/eventos do pod, config e secrets, migração Flyway/Liquibase, rollback se necessário |
| R2 | Latência alta ou timeout | Dashboard por serviço/endpoint, carga do banco e Kafka, escalar ou otimizar, rate limit se limite do plano |
| R3 | Banco do tenant inacessível | Conectividade e credenciais, status da instância no cloud, provisioning do tenant, restore de backup |
| R4 | Incidente de segurança ou vazamento | Conter (isolar, revogar tokens), avaliar alcance, acionar resposta a incidentes e notificação (LGPD/BACEN) |
| R5 | Novo tenant a provisionar | Cadastrar no provisioning, executar provisionar, validar acesso, registrar no billing e enviar doc ao cliente |
| R6 | Backup e restore | Seguir política/scripts 14.2; testar restore periodicamente; script `restore-postgres.sh` |

**Doc completo**: [aurix-cloud-runbook.md](../../05-infrastructure/infrastructure/aurix-cloud-runbook.md).

[Voltar ao ciclo de vida](README.md) | [Índice da wiki](../../README.md)
