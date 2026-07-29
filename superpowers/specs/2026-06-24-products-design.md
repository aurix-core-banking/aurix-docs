# Design: 6 Novos Produtos Bancários — AURIX Platform

> Data: 2026-06-24
> Status: Design aprovado, aguardando planos de implementação
> Módulos: investimento, consignado, cartoes, cambio, seguros, financiamento

---

## 1. Visão Geral

### 1.1 Stack comum

| Característica | Padrão |
|---------------|--------|
| **Java** | 25 |
| **Spring Boot** | 4.1.0 |
| **Spring Cloud** | 2025.1.2 |
| **Lombok** | Não (manual getters/setters/constructors) |
| **HTTP Client** | `@HttpExchange` + `@ImportHttpServices` |
| **Retry** | `@Retryable` de `org.springframework.resilience.annotation` |
| **Pacote** | `jakarta.*` (não `javax.*`) |
| **Null-safety** | `@NullMarked` package-level (JSpecify) |
| **Gateway** | Rota com `StripPrefix=0`, rate limit global |
| **Persistência** | PostgreSQL, Spring Data JPA, H2 em testes |
| **Eventos** | Kafka, fire-and-forget com try-catch |
| **Cache** | Redis (`spring-boot-starter-data-redis`) |
| **Documentação** | Springdoc OpenAPI 2.2.0 |

### 1.2 Catálogo de módulos

| # | Módulo | Porta | Gateway | Dependências |
|---|--------|-------|---------|-------------|
| 1 | `aurix-investimento` | 8113 | `/api/investimento/**` | — |
| 2 | `aurix-consignado` | 8114 | `/api/consignado/**` | `aurix-salario`, SRCC |
| 3 | `aurix-cartoes` | 8115 | `/api/cartoes/**` | `aurix-ml` (antifraude) |
| 4 | `aurix-cambio` | 8116 | `/api/cambio/**` | `aurix-compliance` |
| 5 | `aurix-seguros` | 8117 | `/api/seguros/**` | `aurix-credit`, `aurix-consignado` |
| 6 | `aurix-financiamento` | 8118 | `/api/financiamento/**` | `aurix-core` (contas) |

### 1.3 Testes

- `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `RestTemplate` com `@LocalServerPort`
- H2 para testes de repositório
- Testcontainers para testes de integração (quando necessário)
- Pelo menos um teste de controller (fluxo feliz + validação) por endpoint público

---

## 2. `aurix-investimento` — Conta Investimento

### 2.1 Propósito

Permite ao cliente investir em produtos de renda fixa e variável. Conta pode operar no modo **vinculado** (exige conta corrente AURIX) ou **standalone** (conta independente — útil para CNPJ separado), configurável por tenant via `investimento.conta-independente: true`.

### 2.2 Entidades

| Entidade | Atributos principais |
|----------|---------------------|
| `ContaInvestimento` | id, tenantId, clienteId, contaCorrenteId (nullable), numero, agencia, saldoTotal, dataAbertura, ativa, modo (VINCULADA, STANDALONE) |
| `ProdutoInvestimento` | id, tenantId, codigo, nome, tipo (CDB, LCI, LCA, TESOURO, FUNDO, ACAO), categoria (RENDA_FIXA, RENDA_VARIAVEL), emissaor, vencimento, taxaRemuneracao, carencia, valorMinimoAplicacao, ativo |
| `OrdemInvestimento` | id, tenantId, contaId, produtoId, tipo (APLICACAO, RESGATE), valor, quantidade, status (PENDENTE, EXECUTADA, CANCELADA, FALHADA), dataOrdem, dataExecucao, observacao |
| `Carteira` | id, tenantId, contaId, produtoId, saldo, quantidade, dataUltimaAtualizacao |

### 2.3 API Surface

```
GET    /api/investimento/contas                    → listar (paginado, filter tenant)
POST   /api/investimento/contas                    → criar (aceita contaCorrenteId ou null)
GET    /api/investimento/contas/{id}               → obter
PATCH  /api/investimento/contas/{id}/encerrar      → encerrar
GET    /api/investimento/contas/{id}/carteira      → saldo por produto
GET    /api/investimento/produtos                  → catálogo (filter tenant, ativos)
POST   /api/investimento/produtos                  → criar produto (admin)
PUT    /api/investimento/produtos/{id}             → atualizar produto
POST   /api/investimento/ordens/aplicar            → aplicar (débito conta corrente se vinculada)
POST   /api/investimento/ordens/resgatar           → resgatar (crédito conta corrente se vinculada)
GET    /api/investimento/ordens/{contaId}          → extrato de ordens
```

### 2.4 Fluxos

**Aplicação vinculada:**
1. POST `/api/investimento/ordens/aplicar` com `contaId produtoId valor`
2. Se conta vinculada → `ContaCorrenteClient.debitar(contaCorrenteId, valor)`
3. Cria `OrdemInvestimento` com status `PENDENTE`
4. Executa em background → atualiza `Carteira` + status `EXECUTADA`
5. Publica `OrdemExecutadaEvent`
6. Se falha → status `FALHADA`, tenta estorno

**Resgate vinculado:**
1. POST `/api/investimento/ordens/resgatar` com `contaId produtoId valor`
2. Se conta vinculada → `ContaCorrenteClient.creditar(contaCorrenteId, valor)`
3. Diminui `Carteira`, publica `ResgateProcessadoEvent`

**Modo standalone:** ignora chamadas a `ContaCorrenteClient`. Movimentação é interna.

### 2.5 Kafka Events

- `ContaInvestimentoCriadaEvent` — conta aberta
- `OrdemExecutadaEvent` — ordem de aplicação executada
- `ResgateProcessadoEvent` — resgate processado

### 2.6 HTTP Clients

- `ContaCorrenteClient` → `aurix-core` (débito/crédito na conta corrente, modo vinculado)

---

## 3. `aurix-consignado` — Crédito Consignado

### 3.1 Propósito

Empréstimo com desconto em folha de pagamento. Módulo independente que consulta `aurix-salario` para validar vínculo empregatício. Suporta três fontes de margem: **INSS** (Dataprev), **Servidor Público** (SIAFI), **Empresa Privada Conveniada** (eSocial ou manual). Integrado com **SRCC (CIP)** para consulta centralizada de contratos ativos entre instituições.

### 3.2 Entidades

| Entidade | Atributos principais |
|----------|---------------------|
| `ContratoConsignado` | id, tenantId, clienteId, contaSalarioId, valorTotal, taxaJuros, prazoMeses, valorParcela, margemUtilizada, fonteMargem (INSS, SIAFI, EMPRESA), status (PROPOSTA, ASSINADO, ATIVO, LIQUIDADO, INADIMPLENTE), dataContratacao |
| `Parcela` | id, contratoId, numero, valor, dataVencimento, dataPagamento, status (PENDENTE, PAGA, ATRASADA, CANCELADA) |
| `MargemConsignavel` | id, tenantId, clienteId, fonteMargem, margemTotal, margemDisponivel, margemUtilizada, dataAtualizacao |
| `ConvenioConsignado` | id, tenantId, nome, tipo (INSS, SIAFI, EMPRESA), codigoFonte, ativo |
| `ConsignadoSource` | id, tenantId, tipo, endpoint, credenciais, config (JSON) |

### 3.3 API Surface

```
GET    /api/consignado/convenios                     → listar convênios (fontes de margem)
POST   /api/consignado/convenios                     → criar convênio
GET    /api/consignado/margem/{clienteId}            → margem disponível (consolida fontes + SRCC)
POST   /api/consignado/contratos                     → contratar
GET    /api/consignado/contratos/{id}                → obter contrato
GET    /api/consignado/contratos/{id}/parcelas       → parcela
GET    /api/consignado/contratos/cliente/{clienteId} → listar contratos do cliente
POST   /api/consignado/contratos/{id}/liquidar       → liquidar antecipadamente
PATCH  /api/consignado/contratos/{id}/renegociar     → renegociar
```

### 3.4 Fluxo de contratação

1. GET `margem/{clienteId}` → consulta `MargemConsignavel` para cada `ConvenioConsignado` ativo
2. Se SRCC habilitado → consulta CIP/SRCC para contratos em outras instituições → subtrai da margem disponível
3. POST `/contratos` com `contaSalarioId valor prazoMeses`:
   - Valida margem disponível
   - Cria `ContratoConsignado` + `Parcela`s
   - Publica `ContratoAssinadoEvent`
   - Se SRCC → registra contrato na CIP
4. Job `@Scheduled` diário: processa parcelas vencidas → debita via `aurix-salario`
5. Publica `ParcelaDebitadaEvent`, `MargemAtualizadaEvent`

### 3.5 Kafka Events

- `ContratoAssinadoEvent` — novo contrato
- `ParcelaDebitadaEvent` — parcela paga
- `MargemAtualizadaEvent` — margem recalculada
- `ContratoLiquidadoEvent` — quitação antecipada

### 3.6 HTTP Clients

- `ContaSalarioClient` → `aurix-salario` (valida vínculo, debita parcela)
- `SrccClient` → CIP/SRCC (consulta margem interestadual, registra contrato)
- `DataprevClient` → INSS (consulta margem Dataprev)
- `SiafiClient` → servidor público (quando aplicável)
- `ESocialClient` → eSocial empresa privada (quando aplicável)

---

## 4. `aurix-cartoes` — Cartões de Crédito/Débito

### 4.1 Propósito

Motor de cartões configurável por bandeira e adquirente. Suporta **bandeiras parceiras** (Visa, Mastercard, Elo) e **private label** (bandeira própria, processamento interno). **Adquirentes parceiros** (Rede, Stone, GetNet) e **own-acquirer** (processamento full issuer). Antifraude integrado com `aurix-ml`.

Configurável por produto/tenant: `brandPartner` e `acquirerPartner` definem qual parceiro ou se é processamento próprio.

### 4.2 Entidades

| Entidade | Atributos principais |
|----------|---------------------|
| `Cartao` | id, tenantId, contaId, produtoId, bandeiraParceiroId, numero (truncado), cvv (hash), validade, titular, tipo (CREDITO, DEBITO, MULTIFUNCIONAL), status (ATIVO, BLOQUEADO, CANCELADO, EXPIRADO), dataEmissao, dataBloqueio, motivoBloqueio |
| `ProdutoCartao` | id, tenantId, nome, bandeira, adquirente, anuidade, taxaJuros, taxaMora, limiteMinimo, limiteMaximo, programaPontos, ativo |
| `Fatura` | id, cartaoId, mesReferencia, dataFechamento, dataVencimento, valorTotal, valorPago, status (ABERTA, FECHADA, PAGA, ATRASADA) |
| `TransacaoCartao` | id, cartaoId, faturaId, dataTransacao, valor, estabelecimento, codigoAutorizacao, tipo (COMPRA, ESTORNO, TAXA), status (AUTORIZADA, NEGADA, ESTORNADA, CANCELADA), modo (CREDITO, DEBITO), parcelas |
| `LimiteCartao` | id, cartaoId, limiteTotal, limiteDisponivel, limiteUtilizado, dataAtualizacao |
| `ParceiroBandeira` | id, tenantId, nome (VISA, MASTERCARD, ELO, PRIVATE_LABEL), tipoEndpoint, config (JSON), ativo |
| `ParceiroAdquirente` | id, tenantId, nome (REDE, STONE, GETNET, OWN_ACQUIRER), tipoEndpoint, config (JSON), ativo |
| `LancamentoFatura` | id, faturaId, descricao, valor, dataLancamento, categoria |

### 4.3 API Surface

```
### Emissão e Gestão
POST   /api/cartoes/emitir                       → emitir cartão (produtoId, contaId, titular)
GET    /api/cartoes/{id}                         → obter dados do cartão
PATCH  /api/cartoes/{id}/bloquear                → bloquear (com motivo)
PATCH  /api/cartoes/{id}/desbloquear             → desbloquear
PATCH  /api/cartoes/{id}/cancelar                → cancelar

### Limites
PATCH  /api/cartoes/{id}/limite                  → ajustar limite
GET    /api/cartoes/{id}/limite                  → consultar limite

### Transações (issuer — usado quando private-label ou own-acquirer)
POST   /api/cartoes/transacoes/autorizar         → autorizar transação (fluxo issuer)
POST   /api/cartoes/transacoes/estornar          → estornar transação
GET    /api/cartoes/transacoes/{cartaoId}        → extrato de transações

### Faturas
GET    /api/cartoes/faturas/{cartaoId}           → faturas (aberta/histórico)
GET    /api/cartoes/faturas/{id}                 → fatura detalhada com lançamentos
POST   /api/cartoes/faturas/{id}/pagar           → pagar fatura (débito conta corrente)

### Admin
GET    /api/cartoes/produtos                     → listar produtos
POST   /api/cartoes/produtos                     → criar produto
PUT    /api/cartoes/produtos/{id}                → atualizar produto
GET    /api/cartoes/parceiros-bandeira           → listar parceiros
POST   /api/cartoes/parceiros-bandeira           → configurar bandeira
GET    /api/cartoes/parceiros-adquirente         → listar adquirentes
POST   /api/cartoes/parceiros-adquirente         → configurar adquirente
```

### 4.4 Fluxo de autorização (issuer — private label / own acquirer)

1. POST `/transacoes/autorizar` recebe `cartaoId, valor, estabelecimento, modo`
2. `AntifraudeService` avalia risco → se alto risco, NEGADA
3. `LimiteCartaoService` verifica disponibilidade
4. Se aprovado → debita limite, cria `TransacaoCartao` com status `AUTORIZADA`
5. Publica `TransacaoAutorizadaEvent`

### 4.5 Fluxo de faturamento

1. Job `@Scheduled` diário: agrega `TransacaoCartao` do período por cartão
2. Na data de fechamento → gera `LancamentoFatura` para cada transação
3. Seta status `FATURA` → `FECHADA`
4. Publica `FaturaFechadaEvent`
5. Pagamento → POST `/faturas/{id}/pagar` → debita conta corrente via `ContaCorrenteClient`

### 4.6 Kafka Events

- `CartaoEmitidoEvent` — novo cartão emitido
- `TransacaoAutorizadaEvent` — transação autorizada (ou negada)
- `TransacaoEstornadaEvent` — estorno processado
- `FaturaFechadaEvent` — fatura mensal fechada
- `FaturaPagaEvent` — pagamento de fatura

### 4.7 HTTP Clients

- `ContaCorrenteClient` → `aurix-core` (débito de faturas, crédito de estornos)
- `VisaClient`, `MastercardClient`, `EloClient` → parceiros bandeira
- `RedeClient`, `StoneClient`, `GetNetClient` → parceiros adquirente
- `MlFraudClient` → `aurix-ml` (antifraude, se disponível)

---

## 5. `aurix-cambio` — Câmbio

### 5.1 Propósito

Operações de câmbio completas: câmbio contratual BACEN (Decreto 23.258), plataforma digital de câmbio (tipo Remessa Online), remessa internacional (SWIFT), câmbio turismo. Compliance cambial integrado com `aurix-compliance` (ROE, limites por CPF/CNPJ, finalidade de remessa). Configurável por tenant.

### 5.2 Entidades

| Entidade | Atributos principais |
|----------|---------------------|
| `ContratoCambio` | id, tenantId, clienteId, tipo (COMPRA, VENDA), moedaOrigem, moedaDestino, valorOrigem, valorDestino, taxaCambio, dataContratacao, dataLiquidacao, finalidade, status (COTADO, CONTRATADO, LIQUIDADO, CANCELADO) |
| `Cotacao` | id, moeda, taxaCompra, taxaVenda, dataCotacao, fonte (BACEN, PARCEIRO, PROPRIO) |
| `OperacaoCambio` | id, contratoId, clienteId, tipo, valorMoedaEstrangeira, valorMoedaNacional, taxa, dataOperacao, registroBACEN |
| `Remessa` | id, contratoId, clienteId, valor, moeda, bancoDestino, contaDestino, codigoSwift, finalidade, status (PENDENTE, ENVIADA, CONFIRMADA, FALHADA), dataSolicitacao, dataConfirmacao |
| `ContaCambio` | id, tenantId, clienteId, saldoPorMoeda (JSON), dataAtualizacao |
| `ClienteCambio` | id, clienteId, documentacao (JSON), limiteRemessaMensal, limiteRemessaAnual, categoriasAutorizadas |

### 5.3 API Surface

```
### Cotações
GET    /api/cambio/cotacoes                       → listar cotações atuais (todas moedas)
GET    /api/cambio/cotacoes/{moeda}               → cotação atual de uma moeda
POST   /api/cambio/cotacoes                       → atualizar cotação (admin ou job)

### Contratos
POST   /api/cambio/contratos                      → fechar contrato de câmbio
GET    /api/cambio/contratos/{id}                 → obter contrato
GET    /api/cambio/contratos/cliente/{clienteId}  → listar contratos do cliente
PATCH  /api/cambio/contratos/{id}/liquidar        → liquidar contrato
PATCH  /api/cambio/contratos/{id}/cancelar        → cancelar

### Remessas
POST   /api/cambio/remessas                       → solicitar remessa internacional
GET    /api/cambio/remessas/{id}                  → status da remessa
GET    /api/cambio/remessas/cliente/{clienteId}   → histórico de remessas
PATCH  /api/cambio/remessas/{id}/cancelar         → cancelar remessa

### Operações
GET    /api/cambio/operacoes/{clienteId}          → extrato cambial do cliente
GET    /api/cambio/operacoes/{clienteId}/limites  → limites de câmbio disponíveis

### Admin
GET    /api/cambio/clientes                       → listar clientes de câmbio
POST   /api/cambio/clientes                       → habilitar cliente para câmbio
PUT    /api/cambio/clientes/{id}/limites          → ajustar limites
```

### 5.4 Fluxo de contrato de câmbio (BACEN)

1. Simulação → GET `/cotacoes/{moeda}` mostra taxa
2. POST `/contratos` → cria `ContratoCambio` com status `COTADO`
3. Valida compliance (`aurix-compliance`): ROE, finalidade, limite cliente
4. Aceita → status `CONTRATADO`, registra no SISBACEN se aplicável
5. Liquidação → PATCH `/liquidar` → debita/credita conforme tipo
6. Registro BACEN e publicação de `ContratoFechadoEvent`

### 5.5 Fluxo de remessa internacional

1. POST `/remessas` → valida compliance cambial (limite BACEN 281/2023)
2. Cria `Remessa` com status `PENDENTE`
3. Job ou processamento assíncrono: conecta SWIFT → envia remessa
4. Confirmação → status `CONFIRMADA`, publica `RemessaProcessadaEvent`
5. Falha → status `FALHADA`, estorno automático

### 5.6 Jobs

- `CotacaoJob` (`@Scheduled`) — atualiza cotações de fontes externas
- `FechamentoDiarioJob` — fecha contratos pendentes, gera registros BACEN
- `RemessaJob` — processa remessas pendentes

### 5.7 Kafka Events

- `CotacaoAtualizadaEvent` — nova cotação publicada
- `ContratoFechadoEvent` — contrato fechado (com registro BACEN)
- `ContratoLiquidadoEvent` — liquidação concluída
- `RemessaProcessadaEvent` — remessa enviada/confirmada

### 5.8 HTTP Clients

- `BacenClient` → SISBACEN (registro contratos, consulta taxas)
- `SwiftClient` → SWIFT (envio de remessas)
- `ComplianceClient` → `aurix-compliance` (validação ROE, limites)
- `ParceiroCambioClient` → provedor de liquidez (se parceiro)

---

## 6. `aurix-seguros` — Seguros

### 6.1 Propósito

Módulo completo de seguros: Vida (individual/grupo), Prestamista (vinculado a crédito), Residencial, Automóvel, Empresarial. Precificação atuarial, comissionamento de corretores, registro SUSEP, cessão a resseguradoras. Prestamista integrado com `aurix-credit` e `aurix-consignado`.

### 6.2 Entidades

| Entidade | Atributos principais |
|----------|---------------------|
| `ProdutoSeguro` | id, tenantId, nome, tipo (VIDA, PRESTAMISTA, RESIDENCIAL, AUTO, EMPRESARIAL), coberturasBase (JSON), coberturasAdicionais (JSON), premioMinimo, comissaoCorretor, tabelaAtuarialId, ativo |
| `CotacaoSeguro` | id, tenantId, clienteId, produtoId, coberturas (JSON), premioCalculado, dataValidade, status (ATIVA, EXPIRADA, CONVERTIDA) |
| `Apolice` | id, tenantId, numero, clienteId, cotacaoId, produtoId, coberturasContratadas (JSON), premioTotal, formaPagamento, dataInicio, dataFim, status (ATIVA, CANCELADA, VENCIDA), endosso |
| `Tomador` | id, apoliceId, nome, documento, tipoPessoa, endereco, contato |
| `ParcelaPremio` | id, apoliceId, numero, valor, dataVencimento, dataPagamento, status (PENDENTE, PAGA, ATRASADA) |
| `Sinistro` | id, apoliceId, clienteId, dataOcorrencia, dataComunicacao, tipo, descricao, valorSolicitado, valorAprovado, status (ABERTO, EM_ANALISE, APROVADO, NEGADO, LIQUIDADO), peritoResponsavel |
| `Comissao` | id, apoliceId, corretorId, valor, percentual, dataPagamento |
| `Corretor` | id, tenantId, nome, documento, susepCode, comissaoPadrao, ativo |

### 6.3 API Surface

```
### Cotação e Emissão
POST   /api/seguros/cotacoes                      → simular/cotar seguro
GET    /api/seguros/cotacoes/{id}                 → obter cotação
POST   /api/seguros/cotacoes/{id}/emitir          → emitir apólice
GET    /api/seguros/apolices/{id}                 → dados da apólice
GET    /api/seguros/apolices/cliente/{clienteId}  → apólices do cliente
PATCH  /api/seguros/apolices/{id}/cancelar        → cancelar apólice

### Sinistros
POST   /api/seguros/sinistros                     → registrar sinistro
GET    /api/seguros/sinistros/{id}                → acompanhamento
PATCH  /api/seguros/sinistros/{id}/analisar       → análise (aprovar/negar)
PATCH  /api/seguros/sinistros/{id}/liquidar       → liquidar (autorizar pagamento)
GET    /api/seguros/sinistros/apolice/{apoliceId} → sinistros da apólice

### Parcelas
GET    /api/seguros/apolices/{id}/parcelas        → parcelas do prêmio
POST   /api/seguros/apolices/{id}/parcelas/pagar  → pagar parcela

### Admin
GET    /api/seguros/produtos                      → listar produtos
POST   /api/seguros/produtos                      → criar produto
PUT    /api/seguros/produtos/{id}                 → atualizar produto
GET    /api/seguros/corretores                    → listar corretores
POST   /api/seguros/corretores                    → cadastrar corretor
GET    /api/seguros/produtos/{id}/tabelas-atuariais → tabelas de precificação
```

### 6.4 Fluxo de emissão (prestamista)

1. POST `/cotacoes` → dados do cliente + valor financiado (`aurix-credit`/`aurix-consignado`)
2. Calcula prêmio baseado em tabela atuarial (idade, valor, prazo)
3. POST `/cotacoes/{id}/emitir` → cria `Apolice`, gera `ParcelaPremio`s
4. Publica `ApoliceEmitidaEvent`
5. Se forma pagamento débito → `ContaCorrenteClient.debitar()`

### 6.5 Fluxo de sinistro

1. POST `/sinistros` → registra ocorrência, status `ABERTO`
2. Automático: status `EM_ANALISE`, notifica perito se aplicável
3. PATCH `/analisar` → decisão: `APROVADO` ou `NEGADO`
4. Se `APROVADO` → PATCH `/liquidar` → crédito via `ContaCorrenteClient`
5. Publica `SinistroLiquidadoEvent`

### 6.6 Kafka Events

- `ApoliceEmitidaEvent` — nova apólice
- `PremioPagoEvent` — prêmio recebido
- `SinistroAbertoEvent` — sinistro registrado
- `SinistroLiquidadoEvent` — sinistro pago

### 6.7 HTTP Clients

- `SusepClient` → SUSEP (registro de apólices e sinistros)
- `ResseguradoraClient` → resseguradora (cessão de risco)
- `ContaCorrenteClient` → `aurix-core` (débito prêmio, crédito sinistro)
- `CreditClient` → `aurix-credit` (dados do contrato para prestamista)
- `ConsignadoClient` → `aurix-consignado` (dados do contrato para prestamista)

---

## 7. `aurix-financiamento` — Financiamento

### 7.1 Propósito

Financiamento de bens: imobiliário (SFH/SFI), veicular, e outros bens (máquinas, equipamentos). Cálculo de amortização SAC/Price. Garantias (alienação fiduciária, hipoteca, penhor). Registro de garantia (RGI, Detran).

### 7.2 Entidades

| Entidade | Atributos principais |
|----------|---------------------|
| `ContratoFinanciamento` | id, tenantId, clienteId, tipo (IMOBILIARIO, VEICULAR, OUTROS_BENS), valorFinanciado, valorEntrada, taxaJuros, prazoMeses, sistemaAmortizacao (SAC, PRICE, SACRE), valorParcela, saldoDevedor, dataContratacao, dataPrimeiraParcela, status (PROPOSTA, ASSINADO, ATIVO, LIQUIDADO, INADIMPLENTE) |
| `ParcelaFinanciamento` | id, contratoId, numero, dataVencimento, valorParcela, valorAmortizacao, valorJuros, valorSaldoDevedor, dataPagamento, status (PENDENTE, PAGA, ATRASADA, CANCELADA) |
| `BemFinanciado` | id, contratoId, tipo (IMOVEL, VEICULO, EQUIPAMENTO), descricao, valorAvaliacao, chassi (se veículo), placa (se veículo), matriculaRGI (se imóvel), registroGarantia |
| `Garantia` | id, contratoId, bemId, tipo (ALIENACAO_FIDUCIARIA, HIPOTECA, PENHOR), valor, dataRegistro, dataBaixa, status (ATIVA, LIBERADA), orgaoRegistro |
| `SimulacaoFinanciamento` | id, tenantId, clienteId, tipo, valorFinanciado, prazoMeses, taxaJuros, sistemaAmortizacao, valorParcela, tabelaSAC (JSON), tabelaPrice (JSON), dataSimulacao |

### 7.3 API Surface

```
### Simulação
POST   /api/financiamento/simulacoes              → simular financiamento (SAC/Price)
GET    /api/financiamento/simulacoes/{id}         → obter simulação

### Contratos
POST   /api/financiamento/contratos               → contratar financiamento
GET    /api/financiamento/contratos/{id}          → obter contrato
GET    /api/financiamento/contratos/cliente/{clienteId} → listar contratos
PATCH  /api/financiamento/contratos/{id}/liquidar → liquidar antecipadamente
PATCH  /api/financiamento/contratos/{id}/renegociar → renegociar

### Parcelas
GET    /api/financiamento/contratos/{id}/parcelas → parcela do contrato
POST   /api/financiamento/contratos/{id}/parcelas/pagar → pagar parcela

### Bens e Garantias
GET    /api/financiamento/bens/{contratoId}       → bens do contrato
POST   /api/financiamento/garantias               → registrar garantia
PATCH  /api/financiamento/garantias/{id}/liberar  → liberar garantia

### Admin
GET    /api/financiamento/taxas                   → taxas vigentes
PUT    /api/financiamento/taxas                   → atualizar taxas
```

### 7.4 Fluxo de contratação

1. POST `/simulacoes` com `tipo, valor, prazo, sistemaAmortizacao`
2. Retorna tabela de parcelas (SAC ou Price), valor da parcela, CET
3. POST `/contratos` → valida cadastro, cria `ContratoFinanciamento` + `ParcelaFinanciamento`s + `BemFinanciado`
4. Registra `Garantia` (alienação fiduciária no RGI ou Detran)
5. Credita valor na `ContaCorrente` do cliente
6. Publica `ContratoAssinadoEvent`

### 7.5 Fluxo de amortização

- **SAC**: amortização constante, juros decrescentes
- **Price**: prestação constante, amortização crescente
- **SACRE**: prestação recalculada periodicamente

Job `@Scheduled` diário: gera `ParcelaFinanciamento` com valores calculados. Pagamento → debita conta corrente, atualiza saldo devedor.

### 7.6 Kafka Events

- `SimulacaoRealizadaEvent` — simulação solicitada
- `ContratoFinanciamentoAssinadoEvent` — contrato assinado
- `ParcelaPagaEvent` — parcela paga
- `ContratoLiquidadoEvent` — quitação antecipada
- `GarantiaRegistradaEvent` — garantia registrada

### 7.7 HTTP Clients

- `ContaCorrenteClient` → `aurix-core` (crédito do valor, débito de parcelas)
- `CartorioRgiClient` → registro de garantia imobiliária
- `DetranClient` → registro de garantia veicular
- `BacenClient` → `aurix-bacen` (consulta taxa referencial, TR)

---

## 8. Atualizações na Plataforma

### 8.1 Gateway

Inserir no `application.yml` do `aurix-gateway`:

```yaml
spring.cloud.gateway.routes:
  - id: aurix-investimento
    uri: http://localhost:8113
    predicates:
      - Path=/api/investimento/**
    filters:
      - StripPrefix=0

  - id: aurix-consignado
    uri: http://localhost:8114
    predicates:
      - Path=/api/consignado/**
    filters:
      - StripPrefix=0

  - id: aurix-cartoes
    uri: http://localhost:8115
    predicates:
      - Path=/api/cartoes/**
    filters:
      - StripPrefix=0

  - id: aurix-cambio
    uri: http://localhost:8116
    predicates:
      - Path=/api/cambio/**
    filters:
      - StripPrefix=0

  - id: aurix-seguros
    uri: http://localhost:8117
    predicates:
      - Path=/api/seguros/**
    filters:
      - StripPrefix=0

  - id: aurix-financiamento
    uri: http://localhost:8118
    predicates:
      - Path=/api/financiamento/**
    filters:
      - StripPrefix=0
```

### 8.2 Parent POM

Adicionar no `apps/backend/pom.xml`:

```xml
<module>aurix-investimento</module>
<module>aurix-consignado</module>
<module>aurix-cartoes</module>
<module>aurix-cambio</module>
<module>aurix-seguros</module>
<module>aurix-financiamento</module>
```

### 8.3 API Spec

Após implementação, adicionar tags, paths e schemas no `specs/aurix-core.yaml` para cada módulo.

### 8.4 Frontend

- `aurix-admin`: corrigir `TIPOS_CONTA` (remover `INVESTIMENTO`, adicionar `SALARIO` já existe). Adicionar resources para os novos módulos.
- `aurix-web` (internet banking): adicionar páginas para investimentos, cartões, seguros, financiamento.
- `aurix-mobile` (mobile): adicionar telas correspondentes.

---

## 9. Ordem de Implementação Recomendada

| Ordem | Módulo | Justificativa |
|-------|--------|---------------|
| 1 | `aurix-cartoes` | Já existe esqueleto, expandir é mais rápido. Inconsistência ativa no frontend. |
| 2 | `aurix-investimento` | Inconsistência ativa frontend-backend (`TIPOS_CONTA`). Demanda imediata. |
| 3 | `aurix-consignado` | Depende de `aurix-salario` (já pronto). Produto de alto valor no BR. |
| 4 | `aurix-seguros` | Depende de `aurix-credit` e `aurix-consignado` (prestamista). |
| 5 | `aurix-financiamento` | Maior escopo, faz sentido depois dos produtos de crédito. |
| 6 | `aurix-cambio` | Independente, pode ser feito em paralelo com outros. |

---

## 10. Padrões Transversais

### 10.1 Módulo scaffold (todos seguem)

```
aurix-<modulo>/
├── pom.xml
├── docker/
├── src/main/java/com/aurix/platform/<modulo>/
│   ├── Aurix<Modulo>Application.java
│   ├── entity/
│   ├── repository/
│   ├── dto/
│   ├── event/
│   ├── client/
│   ├── config/
│   ├── service/
│   ├── controller/
│   └── job/                          (se aplicável)
├── src/main/resources/
│   ├── application.yml
│   └── application-prod.yml
└── src/test/java/com/aurix/platform/<modulo>/
    └── controller/
```

### 10.2 Package-level null safety

Todos os módulos usam `@NullMarked` em `package-info.java`.

### 10.3 Testes

- `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `RestTemplate`
- H2 para repo tests
- Ao menos 1 teste de controller feliz + validação por endpoint público
- Testcontainers quando depender de Kafka ou Redis

### 10.4 Observabilidade

- Health endpoint (`/health`) em todos os módulos
- Métricas via Actuator (Prometheus no gateway)
- Logs estruturados (SLF4J + Logback)

---

*Design aprovado em 2026-06-24. Próximo passo: gerar planos de implementação individuais por módulo.*
