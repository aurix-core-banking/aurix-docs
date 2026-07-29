# Checklist – Atualização regulatória

Checklist para manter relatórios e layouts em dia com BACEN e Receita Federal (self-hosted). Em modelo SaaS, o AURIX Cloud mantém um "regulatory pack" versionado e deployado automaticamente.

---

## Relatórios implementados

| Relatório | Módulo | Periodicidade | Config / cron |
|-----------|--------|----------------|---------------|
| COSIF | aurix-bacen | MENSAL | RelatoriosBacenService, CosifReportGenerator |
| PIX | aurix-bacen | DIÁRIO | RelatoriosBacenService |
| Crédito | aurix-bacen | MENSAL | RelatoriosBacenService |
| E-Financeira | aurix-bacen | MENSAL | EFinanceiraReportGenerator |
| SCR/CCS | aurix-bacen | MENSAL | ScrCcsReportGenerator |
| SPED ECD | aurix-bacen | MENSAL | SpedReportGenerator |
| SPED ECF | aurix-bacen | ANUAL | SpedReportGenerator |
| SPED EFD-Reinf | aurix-bacen | MENSAL | SpedReportGenerator |
| BACEN Jud | aurix-bacen | DIÁRIO | BacenJudReportGenerator |

---

## Checklist de atualização (self-hosted)

- [ ] **BACEN**: Verificar circulares e resoluções no site do BACEN; atualizar layouts (COSIF, PIX, SCR, BACEN Jud) conforme nova versão do formato.
- [ ] **Receita Federal**: Verificar e-Financeira e SPED (ECD, ECF, EFD-Reinf); atualizar geradores em `apps/backend/aurix-bacen/service/*ReportGenerator`.
- [ ] **Versão do formato**: Atualizar campo `versaoFormato` em RelatoriosBacenService (ex.: COSIF-2024 -> COSIF-2025) após mudança de layout.
- [ ] **Testes**: Rodar geração para data de referência de teste e validar arquivo gerado contra especificação oficial.
- [ ] **Vencimentos**: Conferir `calcularVencimentoPadrao` (RelatoriosBacenService) com prazos vigentes (ex.: COSIF até dia 10 do mês seguinte).
- [ ] **Scheduler**: Cron em `aurix.bacen.scheduler.cron-relatorios` (default 6h diário); habilitar com `aurix.bacen.scheduler.enabled=true`.

---

## Onde alterar no código

- **Layouts**: `apps/backend/aurix-bacen/src/main/java/com/aurix/platform/bacen/service/`
  - CosifReportGenerator, EFinanceiraReportGenerator, ScrCcsReportGenerator, SpedReportGenerator, BacenJudReportGenerator
- **Serviço e jobs**: RelatoriosBacenService, RegTechSchedulerConfig
- **Config**: `apps/backend/aurix-bacen/src/main/resources/application.yml` (scheduler, periodicidade)

---

## Eventos Kafka (core -> bacen)

Para fechamento contábil e reportes em tempo quase real: o módulo core (ou accounting) deve publicar eventos em tópicos Kafka (ex.: `aurix.fechamento-contabil`, `aurix.movimentacao-diaria`). O módulo bacen, quando configurado com Spring Kafka, poderá assinar esses tópicos e disparar validação ou pré-geração de relatórios. Hoje a geração é acionada por job agendado e por API; a integração via evento fica como próximo passo.

---

## SaaS

Em AURIX Cloud, o regulatory pack é atualizado pela equipe; clientes recebem novas versões via deploy. Não é necessário que o cliente rode este checklist.

Documento completo: [regulatory-pack.md](../../../01-business/conformidade/regulatory-pack.md).

[Voltar aos checklists](README.md) | [Índice da wiki](../../README.md)
