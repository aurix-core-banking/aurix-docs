# Checklist – Atualização regulatória

Checklist para manter relatórios e layouts em dia com BACEN e Receita Federal (self-hosted). Em modelo SaaS, o AUREUS Cloud mantém um "regulatory pack" versionado e deployado automaticamente.

---

## Relatórios implementados

| Relatório | Módulo | Periodicidade | Config / cron |
|-----------|--------|----------------|---------------|
| COSIF | aureus-bacen | MENSAL | RelatoriosBacenService, CosifReportGenerator |
| PIX | aureus-bacen | DIÁRIO | RelatoriosBacenService |
| Crédito | aureus-bacen | MENSAL | RelatoriosBacenService |
| E-Financeira | aureus-bacen | MENSAL | EFinanceiraReportGenerator |
| SCR/CCS | aureus-bacen | MENSAL | ScrCcsReportGenerator |
| SPED ECD | aureus-bacen | MENSAL | SpedReportGenerator |
| SPED ECF | aureus-bacen | ANUAL | SpedReportGenerator |
| SPED EFD-Reinf | aureus-bacen | MENSAL | SpedReportGenerator |
| BACEN Jud | aureus-bacen | DIÁRIO | BacenJudReportGenerator |

---

## Checklist de atualização (self-hosted)

- [ ] **BACEN**: Verificar circulares e resoluções no site do BACEN; atualizar layouts (COSIF, PIX, SCR, BACEN Jud) conforme nova versão do formato.
- [ ] **Receita Federal**: Verificar e-Financeira e SPED (ECD, ECF, EFD-Reinf); atualizar geradores em `backend/aureus-bacen/service/*ReportGenerator`.
- [ ] **Versão do formato**: Atualizar campo `versaoFormato` em RelatoriosBacenService (ex.: COSIF-2024 -> COSIF-2025) após mudança de layout.
- [ ] **Testes**: Rodar geração para data de referência de teste e validar arquivo gerado contra especificação oficial.
- [ ] **Vencimentos**: Conferir `calcularVencimentoPadrao` (RelatoriosBacenService) com prazos vigentes (ex.: COSIF até dia 10 do mês seguinte).
- [ ] **Scheduler**: Cron em `aureus.bacen.scheduler.cron-relatorios` (default 6h diário); habilitar com `aureus.bacen.scheduler.enabled=true`.

---

## Onde alterar no código

- **Layouts**: `backend/aureus-bacen/src/main/java/com/aureus/platform/bacen/service/`
  - CosifReportGenerator, EFinanceiraReportGenerator, ScrCcsReportGenerator, SpedReportGenerator, BacenJudReportGenerator
- **Serviço e jobs**: RelatoriosBacenService, RegTechSchedulerConfig
- **Config**: `backend/aureus-bacen/src/main/resources/application.yml` (scheduler, periodicidade)

---

## Eventos Kafka (core -> bacen)

Para fechamento contábil e reportes em tempo quase real: o módulo core (ou accounting) deve publicar eventos em tópicos Kafka (ex.: `aureus.fechamento-contabil`, `aureus.movimentacao-diaria`). O módulo bacen, quando configurado com Spring Kafka, poderá assinar esses tópicos e disparar validação ou pré-geração de relatórios. Hoje a geração é acionada por job agendado e por API; a integração via evento fica como próximo passo.

---

## SaaS

Em AUREUS Cloud, o regulatory pack é atualizado pela equipe; clientes recebem novas versões via deploy. Não é necessário que o cliente rode este checklist.

Documento completo: [regulatory-pack.md](../../03-compliance/regulatory-pack.md).

[Voltar aos checklists](README.md) | [Índice da wiki](../README.md)
