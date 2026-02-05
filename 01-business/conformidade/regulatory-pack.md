# Regulatory Pack - Checklist de atualizacao regulatoria

Checklist para manter relatorios e layouts em dia com BACEN e Receita Federal (self-hosted). Em modelo SaaS, o AUREUS Cloud mantem um "regulatory pack" versionado e deployado automaticamente.

**Checklist completo em**: [docs/wiki/checklists/regulatorio.md](../wiki/checklists/regulatorio.md). Todos os checklists da plataforma ficam em [docs/wiki/checklists/](../wiki/checklists/).

## Relatorios implementados

| Relatorio | Modulo | Periodicidade | Config / cron |
|-----------|--------|---------------|----------------|
| COSIF | aureus-bacen | MENSAL | RelatoriosBacenService, CosifReportGenerator |
| PIX | aureus-bacen | DIARIO | RelatoriosBacenService |
| Credito | aureus-bacen | MENSAL | RelatoriosBacenService |
| E-Financeira | aureus-bacen | MENSAL | EFinanceiraReportGenerator |
| SCR/CCS | aureus-bacen | MENSAL | ScrCcsReportGenerator |
| SPED ECD | aureus-bacen | MENSAL | SpedReportGenerator |
| SPED ECF | aureus-bacen | ANUAL | SpedReportGenerator |
| SPED EFD-Reinf | aureus-bacen | MENSAL | SpedReportGenerator |
| BACEN Jud | aureus-bacen | DIARIO | BacenJudReportGenerator |

## Checklist de atualizacao (self-hosted)

- [ ] **BACEN**: Verificar circulares e resolucoes no site do BACEN; atualizar layouts (COSIF, PIX, SCR, BACEN Jud) conforme nova versao do formato.
- [ ] **Receita Federal**: Verificar e-Financeira e SPED (ECD, ECF, EFD-Reinf); atualizar geradores em `backend/aureus-bacen/service/*ReportGenerator`.
- [ ] **Versao do formato**: Atualizar campo `versaoFormato` em RelatoriosBacenService (ex.: COSIF-2024 -> COSIF-2025) apos mudanca de layout.
- [ ] **Testes**: Rodar geracao para data de referencia de teste e validar arquivo gerado contra especificacao oficial.
- [ ] **Vencimentos**: Conferir `calcularVencimentoPadrao` (RelatoriosBacenService) com prazos vigentes (ex.: COSIF ate dia 10 do mes seguinte).
- [ ] **Scheduler**: Cron em `aureus.bacen.scheduler.cron-relatorios` (default 6h diario); habilitar com `aureus.bacen.scheduler.enabled=true`.

## Onde alterar no codigo

- **Layouts**: `backend/aureus-bacen/src/main/java/com/aureus/platform/bacen/service/`
  - CosifReportGenerator, EFinanceiraReportGenerator, ScrCcsReportGenerator, SpedReportGenerator, BacenJudReportGenerator
- **Servico e jobs**: RelatoriosBacenService, RegTechSchedulerConfig
- **Config**: `backend/aureus-bacen/src/main/resources/application.yml` (scheduler, periodicidade)

## Eventos Kafka (core -> bacen)

Para fechamento contabil e reportes em tempo quase real: o modulo core (ou accounting) deve publicar eventos em topicos Kafka (ex.: `aureus.fechamento-contabil`, `aureus.movimentacao-diaria`). O modulo bacen, quando configurado com Spring Kafka, podera assinar esses topicos e disparar validacao ou pre-geracao de relatorios. Hoje a geracao e acionada por job agendado e por API; a integracao via evento fica como proximo passo apos unificacao do uso de Kafka no projeto.

## SaaS

Em AUREUS Cloud, o regulatory pack e atualizado pela equipe; clientes recebem novas versoes via deploy. Nao e necessario que o cliente rode este checklist.
