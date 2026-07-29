# Checklist – Completude da Plataforma (Prontidão para Produção)

Consolida a auditoria de prontidão para produção realizada em 2026-06-17 (backend, regras de negócio/compliance, infraestrutura e frontend) em um plano de ação rastreável. Objetivo: sair do estágio atual de **protótipo funcional com lógica crítica simulada** para um core banking real. Os documentos de visão (`roadmap.md`, `big-picture.md`) descrevem a plataforma como "produção" — este checklist é a lista do que falta para essa afirmação ser verdadeira.

Prioridades: **P0** = bloqueador (segurança/regulatório/financeiro, não pode ir a produção sem isso), **P1** = alto risco, necessário antes de operar com clientes reais, **P2** = relevante para maturidade/escala, não bloqueia um go-live controlado.

---

## 1. Segurança – bloqueadores imediatos (P0)

- [ ] **Remover todos os segredos hardcoded do repositório**: JWT secret e chave AES em `apps/backend/aurix-security/.../application.yml` e `apps/backend/aurix-compliance/.../application.yml`; senha de banco em claro; client secrets OAuth em `infra/iam/keycloak/import/aurix-clients.json`. Migrar para secret manager (Vault/AWS Secrets Manager/Azure Key Vault) ou no mínimo variáveis de ambiente injetadas em runtime.
- [ ] **Trocar todas as senhas padrão** (`aurix123`, `admin123`) em `infra/docker-compose.yml`, `docker-compose.dev.yml`, `infra/juju/bundle.yaml`, `data/pipelines/dbt/profiles.yml` — hoje idênticas entre "dev" e "prod".
- [ ] **Corrigir geração de TOTP/MFA**: `MfaService.java` usa `random.nextInt` em vez de RFC 6238; comparação de token/biometria via `String.equals` (vulnerável a timing attack) — usar comparação em tempo constante.
- [ ] **Habilitar TLS no Nginx** (`infra/nginx/nginx.conf` só escuta porta 80) e configurar rate limiting (`limit_req`/`limit_conn`). Avaliar necessidade de WAF.
- [ ] **Gitleaks/scan de histórico**: rodar scan retroativo no git history para confirmar se algum segredo já exposto precisa ser rotacionado (não só removido do HEAD).

## 2. Liquidação, PIX e BACEN reais (P0)

- [ ] **Substituir geração de protocolo por `Math.random()`** em `SettlementService` por integração real com retorno do BACEN/SPI.
- [ ] **Habilitar e corrigir o client SPI/STR** (`SpiStrApiClientImpl`, hoje `enabled: false` e ignora a resposta real, sempre retornando sucesso) — implementar parsing real do payload, mTLS com certificado carregado de fato (hoje configurado em YAML mas nunca lido pelo código).
- [ ] **Remover taxa SELIC hardcoded** em `BacenIntegrationService` — consumir API SGS real.
- [ ] **Implementar idempotência em endpoints financeiros** (PIX, settlement, billing) — chave de idempotência por requisição para evitar duplicidade em retry de cliente.
- [ ] **Implementar limites PIX** (valor por transação, limite noturno, limite diário) — hoje inexistentes tanto na documentação quanto no código.
- [ ] **Implementar MED (Mecanismo Especial de Devolução)** e fluxo de portabilidade de chave/SPI-DICT real.
- [ ] **Corrigir geradores de relatório BACEN** (`CosifReportGenerator`, `ScrCcsReportGenerator`, `EFinanceiraReportGenerator` etc. em `aurix-bacen`) — hoje produzem dados fake (CNPJ fixo, saldo zero); qualquer envio real ao Bacen como está seria submissão incorreta.
- [ ] **Implementar SPED/SCR de fato** — hoje apenas estrutura de classe com campos genéricos.
- [x] **PIX não movimenta saldo nenhum** — corrigido: `PixTransferenciaService.processarTransferencia` agora debita a conta origem atomicamente e credita o destino quando a chave PIX resolve para uma conta local, seguindo o padrão do [ADR-0002](../../../02-technical/arquitetura/adr/0002-jpa-vs-sql-nativo-fluxos-financeiros.md).
- [x] **Corrigir race condition em atualização de saldo** — corrigido em `aurix-core` (`ContaRepository`/`ContaService`, `UPDATE` atômico condicional) e `aurix-settlement` (`SaldoContaRepository`/`SettlementService.movimentarSaldos`, mesmo padrão; também corrigido um bug onde o saldo era movimentado mesmo em liquidação rejeitada). Pendente: **unificar a fonte de saldo** (`Conta.saldo` vs `SaldoConta` duplicados) — é migração de dado real, ver [ADR-0002](../../../02-technical/arquitetura/adr/0002-jpa-vs-sql-nativo-fluxos-financeiros.md).

## 3. Compliance, AML/PLD e LGPD reais (P0 — exigência legal)

- [ ] **Implementar verificação de sanções** (listas OFAC, ONU) no fluxo de onboarding/transação.
- [ ] **Implementar verificação PEP de fato** (hoje só CRUD sem checagem ativa) e comunicação com COAF/RIF quando aplicável (Lei 9.613/98).
- [ ] **Corrigir `LgpdService.excluirDadosCliente()`** — hoje só loga, não exclui dados; implementar exclusão/anonimização real (Art. 18 LGPD).
- [ ] **Trocar hash não criptográfico** (`String.hashCode()`) por SHA-256 ou equivalente onde usado para anonimização/auditoria.
- [ ] **Especificar regras de negócio de compliance em documento formal** (matriz de decisão PLD, fluxo de bloqueio/desbloqueio) — hoje não existe nenhuma especificação, só código.

## 4. Motor de crédito real (P0/P1)

- [ ] **Substituir `CreditBureauStub`** (score = `ThreadLocalRandom.nextInt(200,901)`) por integração real com bureau (Serasa, Boa Vista, Quod).
- [ ] **Corrigir cálculo de CET** — hoje `taxa × 12`, incorreto; implementar fórmula real (inclui IOF, taxas, prazo, sistema de amortização).
- [ ] **Implementar SAC** (hoje só Tabela Price está correta).
- [ ] **Implementar registro em SCR** (Sistema de Crédito do Bacen) — hoje inexistente.
- [ ] **Implementar antifraude em autorização de cartão** — hoje autorização é apenas 6 dígitos aleatórios, sem nenhuma regra antifraude.

## 5. Conformidade fiscal (P1)

- [ ] **Implementar cálculo real de IOF** (alíquotas por tipo de operação, não genérico base×alíquota).
- [ ] **Implementar IRRF com tabela progressiva real**, PIS/COFINS/CSLL conforme regime tributário.
- [ ] **Implementar COSIF de fato** — hoje só comentário no código.

## 6. Testes automatizados (P0 para módulos críticos)

- [ ] **Cobrir com testes**: `aurix-bacen`, `aurix-settlement`, `aurix-compliance`, `aurix-security`, `aurix-credit` — hoje com **zero testes**.
- [ ] **Expandir testes em `aurix-core`** (apenas 5 testes para 100 classes).
- [ ] **Vincular Checkstyle/PMD/SpotBugs ao lifecycle `verify`** do Maven (hoje configurados mas não bloqueiam build) e revisar `checkstyle-suppressions.xml` (atualmente desativa ~31 regras relevantes).
- [ ] **Substituir `throw new RuntimeException(...)` genérico** por hierarquia de exceções de domínio com mapeamento HTTP consistente (padrão repetido em quase todos os módulos).

## 7. Infraestrutura como código (P1)

- [ ] **Substituir módulos Terraform mock** (`null_resource` com outputs simulados como `vpc_id = "vpc-stub-..."`) por recursos reais provisionáveis (VPC, banco gerenciado, cluster Kubernetes).
- [ ] **Configurar backend remoto de state** (S3+DynamoDB ou equivalente) — hoje sem gestão de state.
- [ ] **Criar tfvars por ambiente** (dev/staging/produção) — hoje só uma variável `environment` default `dev`.
- [ ] **Criar manifests Kubernetes para os 28 serviços que não têm** (hoje só `aurix-core` tem deployment completo com probes/limits).
- [ ] **Adicionar HPA, NetworkPolicy e PodDisruptionBudget**.
- [ ] **Implantar gestão de segredos em runtime** (Vault, AWS/Azure Secrets Manager, ou sealed-secrets no Kubernetes) com rotação de credenciais.

## 8. CI/CD (P1)

- [ ] **Reativar etapa de análise estática no `backend-ci.yml`** (hoje comentada).
- [ ] **Adicionar testes ao `frontend-ci.yml`** (hoje só build+lint).
- [ ] **Definir gate de cobertura mínima** em backend e frontend.
- [ ] **Criar pipeline de CD real** (hoje não existe nenhum deploy automatizado — tudo após o build é manual).

## 9. Observabilidade (P1)

- [ ] **Configurar Alertmanager** (`infra/monitoring`) — hoje regras de alerta existem no Prometheus mas não há `alertmanager.yml`; nenhum alerta chega a notificar ninguém.
- [ ] **Conectar Alertmanager a canal real** (Slack/PagerDuty/e-mail).

## 10. Frontend (P1/P2)

- [ ] **aurix-mobile**: implementar autenticação real (hoje `mock_jwt_token_...`, senha fixa `123456`, sem chamada de rede real) e inicializar projeto nativo (faltam pastas `android/`/`ios/` — não builda como app nativo hoje).
- [ ] **aurix-web**: remover dados mock de `Dashboard.js`/`Contas.js` e o fallback `'mock_token'` no login; completar fluxo de solicitação de crédito (`Credito.js` é um esqueleto de 42 linhas).
- [ ] **Adicionar testes reais** (Cypress está configurado em `aurix-web` mas sem specs; Jest configurado em mobile sem specs) nos 3 apps, com foco nos fluxos de PIX, login e transações.

---

## 11. Oportunidade estratégica: CRM de originação próprio com conector Salesforce

**Situação atual**: não existe nenhum módulo de CRM/funil de vendas. `aurix-onboarding` só começa a atuar quando já existe uma `SolicitacaoConta` formal (ou seja, depois que o lead já decidiu abrir conta); `aurix-organization` é voltado para gestão interna/RH, não para originação de clientes.

**Oportunidade**: criar um módulo `aurix-crm` (ou `aurix-origination`) responsável pelo funil de vendas anterior ao onboarding formal — Lead, Oportunidade, Funil/Etapa, Vendedor/Parceiro, Atividade — reaproveitando o padrão arquitetural já usado em `aurix-onboarding` (interface de serviço + stub) e em `aurix-organization` (Workflow/EtapaWorkflow) para manter o domínio desacoplado.

- [ ] **Modelar domínio de originação**: entidades `Lead`, `Oportunidade`, `Funil`, `EtapaFunil`, `Vendedor/Parceiro`, `Atividade` (ligação posterior com `SolicitacaoConta` do onboarding quando o lead converte).
- [ ] **Expor API REST própria** para captura de leads (formulário web, parceiros, indicação) e gestão do funil — usar isso como CRM nativo padrão, mais barato e já integrado aos dados do core (crédito, score, histórico).
- [ ] **Desenhar a porta de integração antes de implementar o CRM nativo**: interface `OriginationCrmPort` (paralelo ao padrão `BureauService`/`BureauStub`) com duas implementações possíveis — `NativeCrmAdapter` (banco de dados próprio) e `SalesforceCrmAdapter` (Sales Cloud) — selecionável por tenant via `aurix-provisioning`, que já existe para configuração multi-tenant.
- [ ] **Sincronização com Salesforce**: usar `aurix-webhooks` (já existe o mecanismo de dispatch genérico) para publicar eventos (`origination.lead.created`, `origination.opportunity.won`, `origination.lead.converted`) e consumir change events do Salesforce (Platform Events ou polling via REST/Bulk API) para manter os dois lados sincronizados quando o cliente final optar por usar Salesforce.
- [ ] **Decidir modelo de dado mestre**: ao usar Salesforce, definir se Salesforce é "system of engagement" (vendas) e AURIX continua "system of record" para entidades regulatórias (Cliente, Conta, Crédito) — evita duplicar verdade sobre dados sensíveis/regulados em uma ferramenta de terceiro.
- [ ] **Roadmap de fases**: (1) CRM nativo mínimo cobrindo captura de lead → conversão em `SolicitacaoConta`; (2) eventos de webhook para integrações externas; (3) adapter Salesforce sob demanda, sem reescrever o domínio.

## 12. Avaliação: ERP de contabilidade/fiscal/financeiro agregado ao core

**Situação atual**: o "ERP" já existe parcialmente, espalhado em três módulos hoje desenvolvidos de forma isolada:
- `aurix-financial` — contas a pagar/receber, fluxo de caixa, fornecedor (claramente back-office da própria instituição, não livro do cliente final).
- `aurix-accounting` — plano de contas, lançamento contábil, IFRS9/ECL (o módulo mais maduro do backend hoje).
- `aurix-tax` — impostos, retenção, SPED (hoje em estágio inicial, ver seção 5).

**Achado relevante**: há duplicação de entidades entre módulos — `ConciliacaoBancaria` existe de forma independente em **quatro** módulos (`aurix-core`, `aurix-financial`, `aurix-accounting`, `aurix-settlement`) e `Cliente` existe de forma independente em pelo menos quatro lugares (`aurix-core`, `aurix-financial`, `aurix-credit`, `aurix-shared`). Isso indica que os módulos foram criados em paralelo sem um domínio compartilhado consistente — sintoma comum de "esqueleto plausível" sem integração real.

**Recomendação**: faz sentido consolidar, mas não é necessário criar um quarto módulo do zero — o núcleo de ERP já está desenhado, falta integração e maturidade:

- [ ] **Definir fronteira clara do domínio "ERP interno"**: este conjunto cuida das finanças/contabilidade/fiscal da **instituição financeira em si** (CAPEX, fornecedores, livro contábil, impostos da empresa) — separado do livro de clientes (saldo/extrato) que pertence ao core banking. Documentar essa fronteira para evitar mais duplicação.
- [ ] **Eliminar duplicação de `ConciliacaoBancaria` e `Cliente`**: migrar para `aurix-shared` como fonte única, com os módulos consumindo via API/evento em vez de entidade própria.
- [ ] **Fechar o ciclo financeiro automaticamente**: lançamento de fatura em `aurix-billing` deve gerar `ContaReceber` em `aurix-financial` e lançamento em `aurix-accounting` automaticamente (hoje não há evidência dessa integração ponta a ponta).
- [ ] **Conectar `aurix-tax` ao fluxo de billing/settlement** para cálculo automático de impostos sobre receita/operações, em vez de cálculo manual isolado.
- [ ] **Avaliar exposição como produto multi-tenant** (ERP-as-a-service para instituições clientes do BaaS) alinhado ao roadmap SaaS já mencionado em commits recentes — decisão de produto, não só técnica: o ERP interno serve só a própria operação da AURIX, ou se torna oferta para os tenants?
- [ ] Após consolidação, tratar os itens da seção 5 (cálculo fiscal real) como parte deste núcleo, não como módulo isolado.

---

## 13. Comunicação entre serviços/módulos (P1)

Padrão formalizado em [ADR-0001](../../../02-technical/arquitetura/adr/0001-comunicacao-entre-servicos.md): REST com client gerado de OpenAPI para chamadas síncronas, Kafka com outbox transacional para eventos de domínio, saga coreografada para fluxos multi-módulo. Hoje a adoção é parcial — só `aurix-core` e `aurix-pix` publicam eventos, e há clients REST escritos à mão em vez de gerados.

- [ ] **Migrar clients REST escritos à mão** (`CoreApiClient` em `aurix-onboarding`, `CoreApiClientImpl` em `aurix-openfinance`) para clients gerados a partir do OpenAPI do `aurix-core`.
- [ ] **Expandir `aurix-api-specs`** para cobrir todos os módulos (hoje só `aurix-core.yaml` existe) e adicionar AsyncAPI para os tópicos Kafka.
- [ ] **Adotar outbox transacional** nos módulos que mudam estado financeiro crítico sem publicar eventos: `aurix-settlement`, `aurix-billing`, `aurix-bacen`, `aurix-treasury`, `aurix-credit`, `aurix-tax`, `aurix-accounting`.
- [ ] **Resolver dependências Kafka mortas** (declaradas no `pom.xml` mas nunca usadas): `aurix-analytics`, `aurix-audit`, `aurix-compliance`, `aurix-security` — adotar uso real (ex.: analytics/audit/compliance como consumidores) ou remover a dependência. (`aurix-gateway` já resolvido — dependência removida.)
- [x] **Mover `EventListener` de `aurix-shared` para os serviços donos** de cada evento — feito: removido de `aurix-shared`; os 3 listeners com lógica real foram para `aurix-core/.../event/ContaTransacaoEventListener.java` (com lookups reais no lugar dos dados fabricados); os outros 5 (sem nenhum publicador) foram removidos.
- [ ] **Padronizar nome de tópico** para `<dominio>.<entidade>.<evento>.<versao>` (hoje são nomes planos como `conta-criada`, sem domínio nem versão).
- [x] **Corrigir `EventHub.publishEventWithDelay`** (e `publishEventWithRetryInternal`, mesmo bug) — agora usam `ScheduledExecutorService` em vez de `Thread.sleep` bloqueante.

Tabela completa de ação por módulo: ver [ADR-0001](../../../02-technical/arquitetura/adr/0001-comunicacao-entre-servicos.md#plano-de-adoção-por-módulo).

---

## Ordem sugerida de execução

1. **Segurança (seção 1)** — segredos e senhas hardcoded são o risco mais barato de corrigir e mais caro de ignorar.
2. **Liquidação/PIX/BACEN e Compliance/AML (seções 2-3)** — bloqueadores regulatórios e financeiros; sem isso não é um banco operando com dinheiro real.
3. **Crédito e fiscal (seções 4-5)** — dependem de integrações externas (bureau, BACEN), iniciar contratação/homologação em paralelo.
4. **Testes e CI/CD (seções 6, 8)** — rede de segurança para todas as mudanças acima.
5. **Infra, observabilidade e comunicação entre serviços (seções 7, 9, 13)** — necessário antes de qualquer ambiente de produção real; a adoção de outbox/Kafka (seção 13) deve acompanhar a correção dos fluxos financeiros da seção 2, já que são os mesmos módulos.
6. **Frontend (seção 10)** — pode avançar em paralelo, mobile é o mais atrasado.
7. **CRM de originação e consolidação do ERP (seções 11-12)** — oportunidades estratégicas de expansão de produto, não bloqueiam o go-live de core banking, mas valem priorização de roadmap comercial assim que os itens P0 estiverem endereçados.

[Voltar aos checklists](README.md) | [Índice da wiki](../../README.md)
