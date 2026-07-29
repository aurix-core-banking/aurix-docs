# ADR-0003: Integração com Dados de Mercado Abertos para Marcação a Mercado e Precificação

**Status**: Aceito
**Data**: 2026-06-22

---

## Contexto

O módulo de tesouraria e investimentos (`aurix-treasury`) gerencia a custódia de ativos financeiros por meio da entidade [Investimento.java](file:///c:/Users/wende/Projects/aurix-platform/apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/entity/Investimento.java). O modelo atual oferece suporte a diversos tipos de ativos comuns no mercado brasileiro (CDB, LCA, LCI, Tesouro Direto, Fundos de Investimento).

Contudo, a plataforma apresenta limitações operacionais no tratamento desses ativos:
1. **Marcação a Mercado (Mark-to-Market) Estática**: O cálculo de rendimento atual (`rendimentoAtual`) e valor total das carteiras é baseado em taxas fixas acordadas no momento da aplicação. Ativos de renda fixa pós-fixados ou atrelados a índices inflacionários (como `TESOURO_IPCA` ou `TESOURO_PREFIXADO`) demandam a atualização diária de seus Preços Unitários (PUs) e taxas indicativas para refletir o real valor de resgate.
2. **Ausência de Indexadores de Mercado**: Indexadores macroeconômicos fundamentais (como CDI, IPCA e taxas SELIC acumuladas) e benchmarks de renda variável (Ibovespa, IFIX) não são capturados de forma contínua, inviabilizando simulações de rentabilidade comparada para o cliente final.
3. **Catálogo de Ativos Manual**: A inclusão de debêntures, CRI, CRA ou novos fundos CVM exige cadastro manual pelo backoffice, limitando a escalabilidade do produto.

Plataformas abertas como o **Open Brazil Market (OBM)** unificam dados de portfólios CVM, preços diários do Tesouro Transparente, e cotações B3/ANBIMA sem barreiras de autenticação ou paywalls. A adoção de uma arquitetura similar de ingestão de dados abertos resolve as dores acima.

## Decisão

Fica decidida a inclusão de uma camada unificada de **Ingestão de Dados de Mercado Abertos** na arquitetura de dados da AURIX, operando sob as seguintes diretrizes:

1. **Ingestão Batch via Apache Airflow**:
   * Implementar o DAG de ingestão `market_data_ingestion` em `airflow/dags/` que consulta periodicamente as fontes oficiais (CVM, Tesouro Transparente, B3 e ANBIMA).
   * Estabelecer uma rotina diária de atualização síncrona no final do dia útil.

2. **Armazenamento Híbrido (TimescaleDB e ClickHouse)**:
   * **TimescaleDB (`aurix_timeseries`)**: Armazenar os preços unitários (PUs) diários dos títulos públicos/privados e o histórico de indexadores diários (taxa CDI e IPCA acumulado).
   * **ClickHouse (OLAP)**: Armazenar o histórico de cotas de fundos CVM e cotações de fechamento da B3 para permitir análises comparativas de alta performance na camada de analytics.

3. **Precificação Dinâmica na Tesouraria**:
   * Atualizar o [TesourariaAvancadaService.java](file:///c:/Users/wende/Projects/aurix-platform/apps/backend/aurix-treasury/src/main/java/com/aurix/platform/treasury/service/TesourariaAvancadaService.java) para buscar o PU mais recente da tabela de séries temporais do TimescaleDB ao calcular o valor total de custódia dos clientes.
   * Implementar um mecanismo de contingência (*fallback*): caso a API externa falhe, utilizar a última taxa/PU conhecida armazenada localmente com um aviso de "dados defasados".

4. **Exposição no Portal de Cliente**:
   * Habilitar rotas no gateway para expor o histórico de rentabilidade comparada da carteira do cliente contra o CDI/IPCA.
   * Alimentar o modelo de recomendação de investimentos no módulo `aurix-ml` usando a base consolidada de ativos.

## Consequências

### Positivas
* **Valoração Realista**: Clientes passam a ter visibilidade do valor real de seus ativos sob a curva atual de juros (marcação a mercado real), alinhando o Core Banking aos padrões exigidos pelas corretoras e bancos modernos.
* **Operação Eficiente**: Eliminação do overhead operacional de cadastrar taxas e debêntures manualmente no banco de dados.
* **Base para Analytics e IA**: A base histórica de cotações servirá para treinar modelos preditivos de alocação de carteira (`aurix-ml`) e precificação de crédito (`aurix-credit`).

### Negativas / Trade-offs
* **Dependência Externa**: O sistema passa a depender da disponibilidade das fontes públicas e APIs abertas. Mitigado pela persistência histórica local e mecanismos de fallback robustos.
* **Custos de Armazenamento**: Séries temporais de cotas de fundos e ações são volumosas. O uso do ClickHouse minimiza o impacto devido ao seu alto poder de compressão de colunas, mas requer monitoramento de disco.

## Alternativas consideradas

* **APIs de Provedores Privados (Bloomberg/Reuters/Broadcast)**: Descartado inicialmente devido aos altos custos de licenciamento e complexidade de integração para a fase atual do ecossistema.
* **Atualização manual via planilha/painel admin**: Rejeitado por ser suscetível a erros operacionais humanos em tarefas críticas de precificação e valuation de ativos sob custódia.

## Plano de adoção por módulo/fluxo

| Módulo / Fluxo | Situação Atual | Ação | Status |
|---|---|---|---|
| `aurix-data-pipelines` | Não possui fluxos de dados de mercado externos | Implementar DAG `market_data_ingestion` conectando a APIs públicas | ⏳ Planejado |
| `aurix-treasury` | Marcação de ativos com taxa de aplicação estática | Atualizar `TesourariaAvancadaService` para usar PUs históricos do TimescaleDB | ⏳ Planejado |
| `aurix-shared` | Entidade `Investimento` não possui campos de referência externa de ativos | Adicionar campos de Ticker / ID do ativo (ex: CNPJ do fundo, Código do título Tesouro) na coluna JSON `dadosInvestimento` | ⏳ Planejado |
| Frontends (`aurix-web`/`mobile`) | Exibição básica de saldo aplicado | Adicionar gráfico comparativo de rendimento acumulado vs CDI/IPCA | ⏳ Planejado |

[Voltar ao índice de ADRs](README.md)
