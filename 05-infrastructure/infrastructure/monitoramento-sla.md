# 14.4 Monitoramento, alertas e SLA 99,9%

Implementacao do item 14.4 do roadmap: monitoramento e alertas (Prometheus/Grafana) e medicao de SLA 99,9%.

## O que esta implementado

- **Prometheus**: coleta de metricas dos servicos AUREUS (gateway, core, pix, credit, treasury, compliance, security, analytics, audit) e de dependencias (PostgreSQL, Redis, Kafka, etc.). Config: `infrastructure/monitoring/prometheus.yml`.
- **Regras de alerta**: `infrastructure/monitoring/prometheus/rules/sla-uptime.yml` – alertas de servico critico down e SLA em risco; `keycloak-alerts.yml` para IAM.
- **Grafana**: dashboards e datasources provisionados em `infrastructure/monitoring/grafana/`. Dashboards: overview AUREUS, Keycloak.
- **Alertmanager**: configurado no Prometheus (target `alertmanager:9093`). Adicionar container Alertmanager no docker-compose se quiser envio de alertas por email/Slack/PagerDuty.

## Medicao de SLA 99,9%

SLA 99,9% significa disponibilidade de no minimo 99,9% no periodo (ex.: mes calendario), ou seja, tempo de indisponibilidade maximo de **43,2 minutos por mes**.

### Como medir

1. **Prometheus**: as regras em `sla-uptime.yml` expoem `aureus:up:count` e `aureus:up:total` (servicos criticos: gateway, core, pix). Servico considerado disponivel quando `up{job="aureus-*"} == 1`.
2. **Grafana**: criar painel com query tipo:
   - `avg_over_time(up{job=~"aureus-gateway|aureus-core|aureus-pix"}[30d]) * 100` para percentual medio de uptime no ultimo mes (por instancia).
   - Para um unico numero de “disponibilidade do produto”: considerar o conjunto dos tres; se qualquer um estiver 0, o produto esta degradado. Formula: `(sum(avg_over_time(up{job=~"aureus-gateway|aureus-core|aureus-pix"}[30d])) / 3) * 100`.
3. **Relatorio mensal**: exportar ou registrar o percentual de uptime (ex.: 99,95%) e o tempo total de indisponibilidade (minutos) para auditoria e contrato.

### Alertas criticos (acordar / acao imediata)

- **AureusCriticalServiceDown**: um dos servicos criticos (gateway, core, pix) esta down por mais de 2 minutos.
- **AureusSLAAtRisk**: disponibilidade do conjunto de servicos criticos abaixo de 100% por 5 minutos.
- **PostgreSQLDown**: banco principal indisponivel por 1 minuto.

Cada alerta referencia o runbook: [aureus-cloud-runbook.md](aureus-cloud-runbook.md) (R1, R3).

## Referencias

- Roadmap e status: [../roadmap.md](../roadmap.md). Indice infra: [index.md](./index.md).
- Runbook: [aureus-cloud-runbook.md](aureus-cloud-runbook.md).
