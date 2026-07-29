# Análise de Gaps de Negócio — AURIX Platform

> Data: 2026-06-24
> Escopo: 32 módulos backend, 3 frontends, API specs, ML, data pipelines, infraestrutura

---

## 1. Produtos Bancários Faltantes

Produtos essenciais de um core banking que não existem ou estão incompletos.

| Produto | Status | Impacto |
|---------|--------|---------|
| **Conta Investimento** | Frontend admin lista `INVESTIMENTO` como tipo de conta, mas backend não tem módulo. `aurix-treasury` é só posição de caixa, não investimento do cliente. Frontend admin lista `TIPOS_INVESTIMENTO = [CDB, LCI, LCA, TESOURO, FUNDO]` — nenhum tem backend. | Cliente não consegue investir |
| **Cartões** | `aurix-cartoes` existe mas é minimalista: 1 controller, 1 service, 3 entidades (Cartao, Fatura, TransacaoCartao). Sem bandeira, sem adquirente, sem antifraude. | Produto incompleto |
| **Crédito Consignado** | `aurix-credit` tem crédito geral mas sem produto consignado (desconto em folha para empréstimo). `aurix-salario` não implementa essa funcionalidade. | GAP comercial relevante no mercado BR |
| **Câmbio** | Zero. Sem módulo de câmbio, remessa internacional, ou operações cambiais. | Banco não opera moeda estrangeira |
| **Seguros** | Zero. Sem módulo de seguros (prestamista, residencial, vida). | Cross-sell perdido |
| **Financiamento** | Zero. Sem módulo de financiamento imobiliário ou veicular. | Produto básico ausente |
| **Conta Digital / Wallet** | Não existe conta digital simplificada (tipo Nubank, PagBank). | Nicho de mercado descoberto |

---

## 2. Inconsistências Frontend vs Backend

Divergências entre o que o frontend admin espera e o que o apps/backend/spec entregam. Quebram a experiência do usuário e podem causar erros em produção.

| Inconsistência | Frontend Admin | Backend / OpenAPI Spec | Risco |
|---------------|---------------|-----------------------|-------|
| `TIPOS_CONTA` | CORRENTE, POUPANCA, **INVESTIMENTO** | CORRENTE, POUPANCA, **SALARIO** | Dropdown quebrado — usuário cria conta INVESTIMENTO que backend rejeita |
| `TIPOS_TRANSACAO` | DEPOSITO, SAQUE, TRANSFERENCIA, PAGAMENTO, PIX | PIX, TED, DOC, BOLETO, DEBITO, CREDITO | Admin envia `DEPOSITO` e backend espera `PIX` |
| `ContaDTO` (admin) | `numero`, `agencia`, `saldo`, `status`, `cliente.nome` | Spec só tem `id`, `clienteId`, `tipoConta`, `limiteCredito` | Admin renderiza campos que API não retorna |
| `STATUS_COMPLIANCE` | PENDENTE, EM_ANALISE, APROVADO, REJEITADO | LIMITE, BLOQUEIO, ALERTA, KYCA | Modelos semânticos incompatíveis |
| Resources.js | 19 bases de API mapeadas | Apenas 12 tags documentadas no spec | Módulos sem spec (provisioning, billing, onboarding, baas, cartoes, accounting, settlement, pricing, webhooks, openfinance) |

---

## 3. Gaps Regulatórios (BACEN/CMN/COAF/BC)

Obrigações regulatórias do sistema financeiro brasileiro não implementadas.

| Obrigação | Status | Detalhe |
|-----------|--------|---------|
| **COAF** (comunicação de operações suspeitas) | Inexistente | ML tem framework de governança (R1/R2/R3) mas não conectado a nenhum compliance operacional. Nenhum backend reporta ao COAF. |
| **Basileia** (capital regulatório — PR, TCR, Nível I/II) | Inexistente | Nenhum módulo calcula capital regulatório. |
| **Recolhimento Compulsório** | Inexistente | Banco precisa calcular e recolher compulsório (encan­trado) no BACEN. |
| **Tabela Price / SAC / SACRE** | Inexistente | Nenhum enforcement de sistemas de amortização regulados pelo CMN. |
| **Registro de Crédito (SCR)** | Parcial | `aurix-bacen` gera relatório SCR, mas sem sincronia automática — é exportação manual, não integração em tempo real. |
| **LGPD — DPO e Titulares** | Parcial | `aurix-compliance` tem entidades (ConsentimentoLGPD, DireitoEsquecimento) mas sem portal do titular, sem workflow de requisição do art. 18, sem prazos de resposta. |
| **Ouvidoria (Res. BCB 160/2021)** | Inexistente | BACEN exige ouvidoria com prazos de resposta, relatórios semestrais, e registro de manifestações. |
| **Open Finance** | Existe (`aurix-openfinance`) | Mas não integrado com dados reais de contas/transações — usa stubs. APIs de compartilhamento de dados (fase 2, 3) não conectadas ao core. |
| **Juros e Encargos** | Inexistente | Nenhum enforcement de limites regulatórios de juros, mora, multa (CCB, resoluções CMN). |

---

## 4. Gaps Operacionais e de Governança

Funcionalidades operacionais de um banco em produção.

| Gap | Detalhe |
|-----|---------|
| **Limite de crédito centralizado** | `ContaDTO` tem `limiteCredito` no spec mas não há módulo de gestão de limites (cheque especial, porta­bilidade de limite, aumento emergencial). |
| **Encerramento de conta** | Nenhum módulo implementa workflow de encerramento (saldo residual, tarifas, carência, retenção de cliente). |
| **Chargeback / Disputa** | Mediação de transações (chargeback PIX, contestação de boleto, disputa de débito) não existe. |
| **Central de notificações** | Mobile tem Firebase push mas não há orquestrador central de notificações (SMS, email, push). Cada módulo implementa do seu jeito. |
| **Portabilidade bancária** | Só existe portabilidade de salário. Portabilidade de conta corrente (troca de banco mantendo número) não implementada. |
| **Tarifas integradas** | `aurix-pricing` (Drools) é standalone — não integrado com débito automático de tarifas, faturamento, ou isenções. |
| **Gateway — rotas faltantes** | `aurix-security:8085`, `aurix-cartoes:8103`, `aurix-tax:8091`, `aurix-cost:8092`, `aurix-budget:8093`, `aurix-financial:8089`, `aurix-ai:8090` **não têm rota no gateway**. Acessíveis direto ou inacessíveis em produção. |
| **Gateway — rate limit incompleto** | Gateway tem rate limit global (100 req/min, burst 200) mas as rotas de poupanca (8111), salario (8112) e outros módulos mais recentes **não herdam** as regras de plano (free=10, sandbox=30, starter=60, etc.). |

---

## 5. ML/AI: Framework sem Entrega

O projeto investiu pesado em ML/AI mas não conectou a produção.

| Constatação | Impacto |
|------------|---------|
| `aurix-ai` tem Spring AI 1.0.0, LangChain4j 0.36.2, 5 adapters de framework (LangChain, LlamaIndex, CrewAI, AutoGen, Haystack) mas **zero controllers** — nenhuma API é servida. | IA não entrega valor de negócio. |
| `aurix-ml` (Python) tem modelos de fraude, crédito, segmentação e drift detection — mas **não está deployado** em nenhum ambiente. Nenhum backend consome inferência. | Zero scoring em produção. |
| Governança R1/R2/R3 é sofisticada (nonce criptográfico, I6Q, CEFL, ambiguity gate, entropia) mas **não conectada a nenhuma decisão real** no backend. | Framework sem caso de uso. |
| `aurix-analytics` tem stubs de BI, Chatbot, ML. | Placeholders, não funcionais. |
| Modelos de fraude (Isolation Forest + Random Forest) existem mas não são consumidos por PIX, transações, ou onboarding. | Risco operacional sem mitigação ML. |

---

## 6. Gaps de Dados e BI

A plataforma de dados analíticos está montada mas o pipeline não flui.

| Gap | Detalhe |
|-----|---------|
| **ClickHouse sem dbt** | ClickHouse tem schema (transacoes_analytics, contas_analytics, dados_mercado) mas dbt só tem perfil PostgreSQL. O pipeline de analytics não chega ao ClickHouse. |
| **TimescaleDB órfão** | TimescaleDB tem funções PL/pgSQL (calcular_media_movel, detectar_anomalias) mas **nenhum backend ou serviço as consome**. Lógica analítica sem uso. |
| **Airflow com dados mockados** | DAGs usam dados simulados, não consomem Kafka real nem PostgreSQL de produção. |
| **Market data simulado** | CDI, SELIC, IPCA são gerados aleatoriamente — não integrados com fonte real (BACEN, B3, ANBIMA). |
| **Spark/Flink sem cluster** | Código PySpark e PyFlink existe mas não há definição de cluster (nem Docker, nem K8s). Processamento só roda local. |
| **Elasticsearch sem ingestão** | Elasticsearch está configurado (porta 9200, Kibana 5601) mas sem Logstash pipeline configurado (logstash.conf não existe). Nenhum dado chega ao ES. |
| **Reconciliação** | Não há processo de reconciliação entre core (PostgreSQL) e analítico (ClickHouse). Diferenças de saldo passam despercebidas. |

---

## 7. Gaps de Resiliência e Observabilidade

Riscos de operação em produção.

| Gap | Observação |
|-----|-----------|
| **Testcontainers como dependência** | Todo teste backend requer Docker. CI sem Docker não roda nenhum teste. Pipeline de CI frágil. |
| **E2E testa só health check** | Testes de ponta-a-ponta verificam apenas HTTP 200 em health endpoints. Zero teste de fluxo (abrir conta → depositar → transferir → sacar). |
| **Resilience4j em 1 módulo** | Só `aurix-bacen` usa retry/circuit breaker. 31 módulos sem proteção contra falhas de dependências externas. |
| **Observabilidade genérica** | Prometheus + Grafana existem mas sem dashboards customizados por módulo, sem SLIs/SLOs definidos, sem alertas configurados. |
| **Health endpoints sem dependências** | Health checks retornam UP mesmo se PostgreSQL, Kafka ou Redis estiverem indisponíveis (não verificam dependências). |

---

## Resumo por Prioridade

| Prioridade | Gap | Justificativa |
|-----------|-----|---------------|
| 🔴 **Crítico** | Inconsistência `TIPOS_CONTA` (admin tem INVESTIMENTO, backend tem SALARIO) | Usuário consegue criar recurso que backend rejeita |
| 🔴 **Crítico** | Nenhum módulo gateway-roteado tem retry/circuit breaker | Falha em cascata em produção |
| 🔴 **Crítico** | E2E tests só verificam health — sem teste de fluxo | Zero garantia de integração |
| 🟠 **Alto** | Produtos incompletos: investimento, cartão, câmbio | Receita potencial perdida |
| 🟠 **Alto** | Regulatório: COAF, Basileia, Ouvidoria ausentes | Risco de sanção BACEN |
| 🟠 **Alto** | Gateway sem rota para 7 módulos | Microserviços inacessíveis |
| 🟡 **Médio** | ML/AI sem entrega — framework sem inferência | Investimento sem ROI |
| 🟡 **Médio** | Pipeline de dados não flui (ClickHouse sem dbt) | BI/analytics não funciona |
| 🟢 **Baixo** | Stubs vs implementação real (BI, Chatbot, ML) | Placeholders em produção |

---

*Análise gerada em 2026-06-24 baseada em exploração completa do código-fonte, spec OpenAPI, frontends, ML, data pipelines e infraestrutura.*
