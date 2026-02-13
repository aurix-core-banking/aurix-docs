# Item 14 – Infra e operacao para SaaS (indice)

Implementacoes e documentacao do bloco 14 do roadmap (Infra e operacao para SaaS). Todos os subitens foram implementados.

| Item | Titulo | Onde esta |
|------|--------|-----------|
| 14.1 | Alta disponibilidade | [alta-disponibilidade.md](alta-disponibilidade.md) |
| 14.2 | Backups automatizados e restore | [backup-restore.md](backup-restore.md), scripts `infrastructure/scripts/backup-postgres.sh`, `restore-postgres.sh` (.bat) |
| 14.3 | DR – procedimento e teste periodico | [dr-procedimento.md](dr-procedimento.md) |
| 14.4 | Monitoramento e SLA 99,9% | [monitoramento-sla.md](monitoramento-sla.md), `infrastructure/monitoring/prometheus/rules/sla-uptime.yml` |
| 14.5 | Deploy sem downtime | [deploy-sem-downtime.md](deploy-sem-downtime.md) |
| 14.6 | Feature flags por tenant | [feature-flags.md](feature-flags.md), modulo `aureus-provisioning` (tabela, API) |
| 14.7 | Particionamento e indices | [evolucao-arquitetura-dados.md](../../02-technical/arquitetura/evolucao-arquitetura-dados.md), scripts em `infrastructure/data-stack/init-scripts/04-*` e `05-*` |

Visão geral de roadmap e status: [../../01-business/roadmap.md](../../01-business/roadmap.md).
