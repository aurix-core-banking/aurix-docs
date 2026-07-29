# Manutenção

Backup, restore, DR, atualizações de versão, regulatório, Data Lakehouse, feature flags.

---

## Backup e restore

- **Backup**: scripts `infra/scripts/backup-postgres.sh` (e `.bat`); retenção configurável; em multi-tenant incluir todos os bancos.
- **Restore**: `restore-postgres.sh <arquivo.sql> [database]`; validar dados e comunicar impacto.
- **Teste**: rodar restore periodicamente (ex.: trimestral).

**Doc**: [backup-restore.md](../../05-infra/infra/backup-restore.md).

---

## Disaster recovery

- Procedimento documentado; teste periódico para garantir RTO/RPO.

**Doc**: [dr-procedimento.md](../../05-infra/infra/dr-procedimento.md).

---

## Atualizações de versão

- Deploy sem downtime: blue/green, canary, migrations (Flyway/Liquibase) antes ou durante o deploy.

**Doc**: [deploy-sem-downtime.md](../../05-infra/infra/deploy-sem-downtime.md).

---

## Atualização regulatória

- Manter relatórios e layouts (BACEN, Receita) em dia; versão do formato, vencimentos, scheduler.

**Checklist**: [01-guides-checklists/checklists/regulatorio.md](../../01-guides-checklists/checklists/regulatorio.md). **Doc**: [regulatory-pack.md](../../../01-business/conformidade/regulatory-pack.md).

---

## Data Lakehouse

- Buckets MinIO, connections Airflow, DAGs, dbt; evolução (novos DAGs, OpenLineage, MLflow).

**Checklist**: [01-guides-checklists/checklists/data-lakehouse.md](../../01-guides-checklists/checklists/data-lakehouse.md).

---

## Feature flags

- Por tenant via módulo provisioning (tabela e API).

**Doc**: [feature-flags.md](../../05-infra/infra/feature-flags.md).

[Voltar ao ciclo de vida](README.md) | [Índice da wiki](../../README.md)
