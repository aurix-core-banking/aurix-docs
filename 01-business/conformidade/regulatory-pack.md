# Regulatory Pack - Checklist de atualizacao regulatoria

Checklist para manter relatorios e layouts em dia com BACEN e Receita Federal (self-hosted). Em modelo SaaS, o AURIX Cloud mantem um "regulatory pack" versionado e deployado automaticamente.

**Checklist completo em**: [docs/wiki/checklists/regulatorio.md](../wiki/checklists/regulatorio.md). Todos os checklists da plataforma ficam em [docs/wiki/checklists/](../wiki/checklists/).

## Relatorios implementados

| Relatorio | Modulo | Periodicidade | Config / cron |
|-----------|--------|---------------|----------------|
| COSIF | aurix-bacen | MENSAL | RelatoriosBacenService, CosifReportGenerator |
| PIX | aurix-bacen | DIARIO | RelatoriosBacenService |
| Credito | aurix-bacen | MENSAL | RelatoriosBacenService |
| E-Financeira | aurix-bacen | MENSAL | EFinanceiraReportGenerator |
| SCR/CCS | aurix-bacen | MENSAL | ScrCcsReportGenerator |
| SPED ECD | aurix-bacen | MENSAL | SpedReportGenerator |
| SPED ECF | aurix-bacen | ANUAL | SpedReportGenerator |
| SPED EFD-Reinf | aurix-bacen | MENSAL | SpedReportGenerator |
| BACEN Jud | aurix-bacen | DIARIO | BacenJudReportGenerator |

## Checklist de atualizacao (self-hosted)

- [ ] **BACEN**: Verificar circulares e resolucoes no site do BACEN; atualizar layouts (COSIF, PIX, SCR, BACEN Jud) conforme nova versao do formato.
- [ ] **Receita Federal**: Verificar e-Financeira e SPED (ECD, ECF, EFD-Reinf); atualizar geradores em `backend/aurix-bacen/service/*ReportGenerator`.
- [ ] **Versao do formato**: Atualizar campo `versaoFormato` em RelatoriosBacenService (ex.: COSIF-2024 -> COSIF-2025) apos mudanca de layout.
- [ ] **Testes**: Rodar geracao para data de referencia de teste e validar arquivo gerado contra especificacao oficial.
- [ ] **Vencimentos**: Conferir `calcularVencimentoPadrao` (RelatoriosBacenService) com prazos vigentes (ex.: COSIF ate dia 10 do mes seguinte).
- [ ] **Scheduler**: Cron em `aurix.bacen.scheduler.cron-relatorios` (default 6h diario); habilitar com `aurix.bacen.scheduler.enabled=true`.

## Onde alterar no codigo

- **Layouts**: `backend/aurix-bacen/src/main/java/com/aurix/platform/bacen/service/`
  - CosifReportGenerator, EFinanceiraReportGenerator, ScrCcsReportGenerator, SpedReportGenerator, BacenJudReportGenerator
- **Servico e jobs**: RelatoriosBacenService, RegTechSchedulerConfig
- **Config**: `backend/aurix-bacen/src/main/resources/application.yml` (scheduler, periodicidade)

## Eventos Kafka (core -> bacen)

Para fechamento contabil e reportes em tempo quase real: o modulo core (ou accounting) deve publicar eventos em topicos Kafka (ex.: `aurix.fechamento-contabil`, `aurix.movimentacao-diaria`). O modulo bacen, quando configurado com Spring Kafka, podera assinar esses topicos e disparar validacao ou pre-geracao de relatorios. Hoje a geracao e acionada por job agendado e por API; a integracao via evento fica como proximo passo apos unificacao do uso de Kafka no projeto.

## SaaS

Em AURIX Cloud, o regulatory pack e atualizado pela equipe; clientes recebem novas versoes via deploy. Nao e necessario que o cliente rode este checklist.
