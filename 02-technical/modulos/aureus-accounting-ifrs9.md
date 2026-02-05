# IFRS 9 - Modulo de Contabilidade

## Visao Geral

O modulo IFRS 9 do AUREUS implementa os requisitos contabeis do IFRS 9 para instrumentos financeiros:

1. **Classificacao e Mensuracao** - Categorizacao de instrumentos financeiros
2. **Expected Credit Loss (ECL)** - Provisao para perdas esperadas de credito
3. **Hedge Accounting** - Contabilizacao de operacoes de hedge

## Arquitetura

- **Entidades**: InstrumentoFinanceiro, ExpectedCreditLoss, HedgeAccounting, RelatorioIFRS9
- **Services**: IFRS9Service, ECLCalculationService, ClassificationService, HedgeAccountingService, RelatorioIFRS9Service
- **Repositorios**: InstrumentoFinanceiroRepository, ExpectedCreditLossRepository

## Funcionalidades

- Classificacao e mensuracao: categorias AC, FVOCI, FVTPL, HEDGE, DERIVATIVOS
- ECL: modelo de tres estagios, calculo ECL = PD x LGD x EAD
- Hedge accounting: hedge de valor justo, fluxo de caixa, investimento liquido; avaliacao de efetividade (80%-125%)
- Relatorios: classificacao, ECL detalhado/consolidado, hedge, impairment, formatos PDF/Excel/CSV/XML/JSON/HTML

## APIs

- `POST /api/accounting/ifrs9/instrumentos/classificar`
- `POST /api/accounting/ifrs9/ecl/calcular/{instrumentoId}`
- `POST /api/accounting/ifrs9/instrumentos/{id}/reclassificar`
- `POST /api/accounting/ifrs9/hedge`, `POST .../hedge/{hedgeId}/avaliar-efetividade`
- `POST /api/accounting/ifrs9/relatorios/consolidado`, `/classificacao`, `/ecl-detalhado`, `/hedge`

## Integracao

- Modulos: aureus-credit, aureus-treasury, aureus-analytics, aureus-compliance
- Externos: agencias de rating, scoring, mercados, BACEN, CVM

## Compliance

IFRS 9, IAS 39, Circular BACEN 4.557/2017, Instrucao CVM 480/2009.
