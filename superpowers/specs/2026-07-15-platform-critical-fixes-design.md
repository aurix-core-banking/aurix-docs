# Platform Critical Fixes — Design Spec (Fase 3)

> **Objetivo:** Corrigir os gaps críticos de produção identificados na análise de negócio de Julho/2026.
> **Prioridade:** P0 e P1 — bloqueantes de produção e de receita.
> **Dependências:** Fase 1 (Foundation: Customer, KYC, Fraud, Notification) e Fase 2 (Revenue: Acquirer, Collections, Guarantee) devem estar implementadas.

---

## Resumo dos Gaps Corrigidos

| # | Gap | Prioridade | Tipo |
|---|-----|-----------|------|
| 1 | Conflito de porta 8090 (`aurix-ai`, `aurix-catalog`, `aurix-controller`) | 🔴 P0 | Infra |
| 2 | `aurix-fraud`, `aurix-kyc`, `aurix-customer`, `aurix-notification` fora do Maven parent | 🔴 P0 | Build |
| 3 | `aurix-fraud` não integrado com PIX, Cartões e Financial | 🔴 P0 | Integração |
| 4 | `aurix-financiamento` sem implementação real | 🟠 P1 | Backend |
| 5 | Frontend web sem páginas para Câmbio, Seguros, Consignado | 🟠 P1 | Frontend |
| 6 | `TIPOS_CONTA` no admin tem `INVESTIMENTO`; backend tem `SALARIO` | 🔴 P0 | Frontend/Backend |
| 7 | `aurix-ai` sem nenhum controller exposto | 🟠 P1 | Backend |

---

## Gap 1 — Conflito de Porta 8090

### Problema

Três serviços apontam para a porta 8090 no `README.md` e em `traefik/dynamic.yml`:
- `aurix-ai` → 8090
- `aurix-catalog` → 8090
- `aurix-controller` → 8090

O `aurix-controller` é o correto na porta 8090 (já declarado no `docker-compose.yml`).

### Correção

#### `README.md` — tabela de módulos

| Módulo | Porta correta |
|--------|--------------|
| `aurix-ai` | **8116** (próxima disponível após 8115/cartões) |
| `aurix-catalog` | **8120** (após 8119/câmbio) |
| `aurix-controller` | **8090** (mantido) |

#### `infra/traefik/dynamic.yml`

```yaml
# Corrigir serviços ai e catalog
ai:
  loadBalancer:
    servers:
      - url: "http://aurix-ai:8116"   # era 8090

catalog:
  loadBalancer:
    servers:
      - url: "http://aurix-catalog:8120"  # era 8090
```

#### `infra/docker-compose.yml` — atualizar portas

**aurix-ai:**
```yaml
aurix-ai:
  ports:
    - "8116:8116"    # era 8090:8090
  environment:
    SERVER_PORT: 8116
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:8116/api/ai/health"]
```

**aurix-catalog:**
```yaml
aurix-catalog:
  ports:
    - "8120:8120"    # era 8090:8090
  environment:
    SERVER_PORT: 8120
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:8120/api/catalog/health"]
```

#### `apps/backend/aurix-ai/src/main/resources/application.yml`

```yaml
server:
  port: 8116
```

#### `apps/backend/aurix-catalog/src/main/resources/application.yml`

```yaml
server:
  port: 8120
```

#### `tests/e2e/config.py`

```python
SERVICES = {
    # ...
    "aurix-ai":      "http://localhost:8116/api/ai/health",
    "aurix-catalog": "http://localhost:8120/api/catalog/health",
}
```

---

## Gap 2 — Módulos fora do Maven Parent

### Problema

Os seguintes módulos existem no filesystem e no Docker Compose mas **não constam no `apps/backend/pom.xml`** como módulos Maven:

- `aurix-customer`
- `aurix-kyc`
- `aurix-fraud`
- `aurix-notification`

Consequência: `mvn clean install` ignora esses módulos. CI não os compila nem testa.

### Correção — `apps/backend/pom.xml`

Localizar a seção `<modules>` e adicionar os 4 módulos em ordem alfabética:

```xml
<modules>
    <module>aurix-accounting</module>
    <module>aurix-ai</module>
    <module>aurix-analytics</module>
    <module>aurix-audit</module>
    <module>aurix-baas</module>
    <module>aurix-bacen</module>
    <module>aurix-billing</module>
    <module>aurix-budget</module>
    <module>aurix-cambio</module>
    <module>aurix-cartoes</module>
    <module>aurix-catalog</module>
    <module>aurix-compliance</module>
    <module>aurix-consignado</module>
    <module>aurix-controller</module>
    <module>aurix-core</module>
    <module>aurix-cost</module>
    <module>aurix-credit</module>
    <!-- NOVOS: Foundation modules -->
    <module>aurix-customer</module>
    <module>aurix-financial</module>
    <module>aurix-financiamento</module>
    <!-- NOVO -->
    <module>aurix-fraud</module>
    <module>aurix-gateway</module>
    <module>aurix-internet-banking</module>
    <module>aurix-investimento</module>
    <!-- NOVO -->
    <module>aurix-kyc</module>
    <module>aurix-mobile-banking</module>
    <!-- NOVO -->
    <module>aurix-notification</module>
    <module>aurix-onboarding</module>
    <module>aurix-openfinance</module>
    <module>aurix-organization</module>
    <module>aurix-pix</module>
    <module>aurix-poupanca</module>
    <module>aurix-pricing</module>
    <module>aurix-provisioning</module>
    <module>aurix-salario</module>
    <module>aurix-security</module>
    <module>aurix-seguros</module>
    <module>aurix-settlement</module>
    <module>aurix-shared</module>
    <module>aurix-tax</module>
    <module>aurix-treasury</module>
    <module>aurix-webhooks</module>
</modules>
```

### Verificação de `pom.xml` individuais

Cada módulo novo deve ter no seu `pom.xml`:

```xml
<parent>
    <groupId>com.aurix.platform</groupId>
    <artifactId>aurix-platform-parent</artifactId>
    <version>1.2.0</version>
    <relativePath>../pom.xml</relativePath>
</parent>
```

Verificar e corrigir nos 4 módulos. Se estiver ausente, adicionar herdando do parent raiz.

---

## Gap 3 — Integração Fraud com PIX, Cartões e Financial

### Problema

`aurix-fraud` tem `FraudConsumer.java` e `FraudScoringService.java`, mas os módulos `aurix-pix`, `aurix-cartoes` e `aurix-financial` **não publicam eventos para os tópicos que o Fraud consome**.

### Tópicos a adicionar em `aurix-shared` — `Topics.java`

```java
// ===== fraud integration =====
String PIX_TRANSFERENCIA_INICIADA      = "pix.transferencia.iniciada.v1";
String PIX_CHAVE_CRIADA                = "pix.chave.criada.v1";
String CARTOES_TRANSACAO_AUTORIZADA    = "cartoes.transacao.autorizada.v1";
String CARTOES_TRANSACAO_NEGADA        = "cartoes.transacao.negada.v1";
String FINANCIAL_MOVIMENTACAO_CRIADA   = "financial.movimentacao.criada.v1";
String FRAUD_TRANSACAO_BLOQUEADA       = "fraude.transacao.bloqueada.v1";
String FRAUD_OCORRENCIA_CRIADA         = "fraude.ocorrencia.criada.v1";
String FRAUD_SCORE_ALTERADO            = "fraude.score.alterado.v1";
```

### Eventos tipados a criar em `com.aurix.platform.shared.event`

**`PixTransferenciaIniciadaEvent`**
```java
public class PixTransferenciaIniciadaEvent extends BaseEvent {
    private Long transacaoId;
    private String chavePix;
    private BigDecimal valor;
    private Long pagadorId;
    private Long recebedorId;
    private String tipo; // IMEDIATO / AGENDADO
}
```

**`CartaoTransacaoAutorizadaEvent`**
```java
public class CartaoTransacaoAutorizadaEvent extends BaseEvent {
    private Long transacaoId;
    private Long cartaoId;
    private Long clienteId;
    private BigDecimal valor;
    private String bandeira;
    private String estabelecimento;
    private String tipoCaptura; // PRESENCIAL / ONLINE / CONTACTLESS
}
```

**`FinancialMovimentacaoCriadaEvent`**
```java
public class FinancialMovimentacaoCriadaEvent extends BaseEvent {
    private Long movimentacaoId;
    private Long contaId;
    private Long clienteId;
    private BigDecimal valor;
    private String tipo; // CREDITO / DEBITO
    private String origem; // PIX / TED / DOC / BOLETO / INTERNO
}
```

### Produtor em `aurix-pix`

Criar `PagamentoPixProducer.java` em `com.aurix.platform.pix.event`:

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class PagamentoPixProducer {

    private final KafkaTemplate<String, PixTransferenciaIniciadaEvent> kafkaTemplate;

    public void publicarTransferenciaIniciada(PagamentoPix pix) {
        var event = PixTransferenciaIniciadaEvent.builder()
            .transacaoId(pix.getId())
            .chavePix(pix.getChavePix())
            .valor(pix.getValor())
            .pagadorId(pix.getPagadorId())
            .recebedorId(pix.getRecebedorId())
            .build();
        kafkaTemplate.send(Topics.PIX_TRANSFERENCIA_INICIADA, String.valueOf(pix.getId()), event);
        log.info("PIX transferencia iniciada publicada: id={}", pix.getId());
    }
}
```

Injetar `PagamentoPixProducer` em `PagamentoPixService` e chamar após persistir a transação.

### Produtor em `aurix-cartoes`

Criar `CartaoTransacaoProducer.java` em `com.aurix.platform.cartoes.event`:

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class CartaoTransacaoProducer {

    private final KafkaTemplate<String, CartaoTransacaoAutorizadaEvent> kafkaTemplate;

    public void publicarTransacaoAutorizada(TransacaoCartao transacao) {
        var event = CartaoTransacaoAutorizadaEvent.builder()
            .transacaoId(transacao.getId())
            .cartaoId(transacao.getCartaoId())
            .clienteId(transacao.getClienteId())
            .valor(transacao.getValor())
            .bandeira(transacao.getBandeira())
            .build();
        kafkaTemplate.send(Topics.CARTOES_TRANSACAO_AUTORIZADA, String.valueOf(transacao.getId()), event);
    }
}
```

### Consumer em `aurix-fraud`

Atualizar `FraudConsumer.java` para consumir os tópicos reais:

```java
@KafkaListener(topics = {
    Topics.PIX_TRANSFERENCIA_INICIADA,
    Topics.CARTOES_TRANSACAO_AUTORIZADA,
    Topics.FINANCIAL_MOVIMENTACAO_CRIADA
}, groupId = "aurix-fraud-group")
public void onTransacao(BaseEvent event, @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
    log.info("Fraude recebeu evento do tópico: {}", topic);
    fraudScoringService.avaliarRisco(event);
}
```

### `FraudScoringService` — lógica de scoring por tipo de evento

```java
public ScoreTransacao avaliarRisco(BaseEvent event) {
    if (event instanceof PixTransferenciaIniciadaEvent pix) {
        return avaliarPix(pix);
    } else if (event instanceof CartaoTransacaoAutorizadaEvent cartao) {
        return avaliarCartao(cartao);
    } else if (event instanceof FinancialMovimentacaoCriadaEvent mov) {
        return avaliarMovimentacao(mov);
    }
    return ScoreTransacao.padrao();
}

private ScoreTransacao avaliarPix(PixTransferenciaIniciadaEvent pix) {
    double score = 0.0;
    List<String> regrasDisparadas = new ArrayList<>();

    // Regra 1: valor alto (>10k) em horário suspeito (00:00–05:00)
    if (pix.getValor().compareTo(BigDecimal.valueOf(10_000)) > 0) {
        int hora = LocalTime.now().getHour();
        if (hora >= 0 && hora < 5) {
            score += 0.40;
            regrasDisparadas.add("VALOR_ALTO_HORARIO_SUSPEITO");
        }
    }
    // Regra 2: chave PIX nova (menos de 24h de cadastro)
    if (chavePixService.isChaveRecente(pix.getChavePix())) {
        score += 0.30;
        regrasDisparadas.add("CHAVE_PIX_RECENTE");
    }
    // Regra 3: cliente com bloqueio preventivo ativo
    if (bloqueioService.temBloqueioAtivo(pix.getPagadorId())) {
        score = 1.0;
        regrasDisparadas.add("CLIENTE_BLOQUEADO");
    }

    String decisao = score >= 0.7 ? "BLOQUEAR" : score >= 0.4 ? "REVISAR" : "APROVAR";
    return salvarScore(pix.getTransacaoId(), "pix", pix.getPagadorId(), score, regrasDisparadas, decisao);
}
```

Se decisão = `BLOQUEAR`, publicar `fraude.transacao.bloqueada.v1` e registrar `OcorrenciaFraude`.

---

## Gap 4 — `aurix-financiamento` Implementation

### Problema

O módulo `aurix-financiamento` tem Dockerfile, Docker Compose e spec OpenAPI, mas o `src/main/java` está vazio — sem entidades, serviços ou controllers.

### Entidades

**`ContratoFinanciamento`** (`financiamento.contratos`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| clienteId | Long | FK → customer.cliente |
| tipo | String | IMOBILIARIO / VEICULAR / PESSOAL |
| valorFinanciado | BigDecimal | Valor total financiado |
| valorEntrada | BigDecimal | Valor da entrada |
| numeroParcelas | Integer | Número de parcelas |
| valorParcela | BigDecimal | Valor calculado da parcela |
| taxaJurosMensal | BigDecimal | Taxa de juros % ao mês |
| sistemaAmortizacao | String | PRICE / SAC / SACRE |
| dataContrato | LocalDate | Data de assinatura |
| dataVencimentoPrimeira | LocalDate | Vencimento da 1ª parcela |
| status | String | PROPOSTA / APROVADO / ATIVO / QUITADO / INADIMPLENTE |
| tenantId | String | Tenant ID |

**`ParcelaFinanciamento`** (`financiamento.parcelas`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| contratoId | Long | FK → contrato |
| numeroParcela | Integer | Número da parcela |
| dataVencimento | LocalDate | Data de vencimento |
| valorParcela | BigDecimal | Valor nominal |
| valorJuros | BigDecimal | Componente de juros |
| valorAmortizacao | BigDecimal | Componente de amortização |
| saldoDevedor | BigDecimal | Saldo devedor após parcela |
| status | String | PENDENTE / PAGA / ATRASADA |
| dataPagamento | LocalDate | Data do pagamento (se paga) |

**`ImovelFinanciado`** (`financiamento.imoveis`) — exclusivo tipo IMOBILIARIO
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| contratoId | Long | FK → contrato |
| endereco | String | Endereço completo |
| cep | String | CEP |
| valorAvaliacao | BigDecimal | Valor de avaliação |
| matricula | String | Número da matrícula |
| cartorio | String | Cartório de registro |
| status | String | DISPONIVEL / ALIENADO / LIBERADO |

**`VeiculoFinanciado`** (`financiamento.veiculos`) — exclusivo tipo VEICULAR
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| contratoId | Long | FK → contrato |
| marca | String | Marca |
| modelo | String | Modelo |
| anoFabricacao | Integer | Ano de fabricação |
| chassi | String | Número do chassi |
| placa | String | Placa |
| valorFipe | BigDecimal | Valor FIPE |
| status | String | DISPONIVEL / ALIENADO / LIBERADO |

### Services

**`FinanciamentoService`**
```java
// Calcular parcelas por sistema de amortização
public List<ParcelaFinanciamento> calcularTabela(BigDecimal valor, int parcelas,
    BigDecimal taxaMensal, SistemaAmortizacao sistema);

// Price (parcela fixa, juros decrescentes)
private List<ParcelaFinanciamento> calcularPrice(BigDecimal pv, int n, BigDecimal i);

// SAC (amortização fixa, parcelas decrescentes)
private List<ParcelaFinanciamento> calcularSAC(BigDecimal pv, int n, BigDecimal i);

// Criar proposta de financiamento
public ContratoFinanciamento criarProposta(CriarPropostaRequest request);

// Aprovar proposta (requer análise de crédito prévia)
public ContratoFinanciamento aprovarProposta(Long contratoId, AprovarPropostaRequest request);

// Processar pagamento de parcela
public ParcelaFinanciamento pagarParcela(Long contratoId, Integer numeroParcela,
    PagarParcelaRequest request);

// Calcular quitação antecipada (com desconto de juros futuros)
public SimulacaoQuitacaoResponse simularQuitacao(Long contratoId, LocalDate dataQuitacao);
```

**`AmortizacaoService`** — lógica pura de cálculo financeiro (testável isoladamente):
```java
// Fator de parcela Price: PMT = PV * [i(1+i)^n / ((1+i)^n - 1)]
public BigDecimal calcularPmtPrice(BigDecimal pv, int n, BigDecimal i);

// SAC: amortização = PV / n; juros = saldo * i; parcela = amort + juros
public BigDecimal calcularParcelaSAC(BigDecimal pv, int n, BigDecimal i, int parcelaAtual);
```

### REST API

| Método | Path | Descrição |
|--------|------|-----------|-|
| POST | `/api/financiamento/propostas` | Criar proposta de financiamento |
| GET | `/api/financiamento/propostas/{id}` | Consultar proposta |
| POST | `/api/financiamento/propostas/{id}/aprovar` | Aprovar proposta |
| POST | `/api/financiamento/propostas/{id}/rejeitar` | Rejeitar proposta |
| GET | `/api/financiamento/contratos` | Listar contratos (filtro: cliente, tipo, status) |
| GET | `/api/financiamento/contratos/{id}` | Detalhe do contrato |
| GET | `/api/financiamento/contratos/{id}/parcelas` | Tabela de parcelas (price/SAC) |
| POST | `/api/financiamento/contratos/{id}/parcelas/{n}/pagar` | Registrar pagamento |
| GET | `/api/financiamento/contratos/{id}/quitacao` | Simular quitação antecipada |
| POST | `/api/financiamento/simulacoes` | Simular sem criar contrato |
| GET | `/api/financiamento/health` | Health check |

### Kafka Events

- `financiamento.proposta.criada.v1` — proposta iniciada
- `financiamento.contrato.aprovado.v1` — crédito aprovado
- `financiamento.parcela.paga.v1` — parcela quitada
- `financiamento.contrato.inadimplente.v1` — atraso detectado (job diário)

### Job de inadimplência

```java
@Scheduled(cron = "0 0 8 * * *")  // diariamente às 08:00
public void verificarInadimplencia() {
    // Buscar parcelas vencidas há mais de 3 dias
    // Atualizar status parcela para ATRASADA
    // Se 3+ parcelas atrasadas → contrato INADIMPLENTE
    // Publicar financiamento.contrato.inadimplente.v1
}
```

---

## Gap 5 — Frontend Web: Páginas Câmbio, Seguros, Consignado

### Arquitetura

Seguir exatamente o padrão de `Cartoes.js` e `Investimentos.js`:
- Functional components com `export default function`
- MUI 5 (Box, Card, Grid, TextField, Button, Table, Dialog, Stepper)
- `apiService` para chamadas de API (com fallback para mock em dev)
- `numeral` para formatação de moeda
- `date-fns` com locale `ptBR`
- Labels em português

### Nova página: `Cambio.js`

**Path:** `apps/frontend/aurix-web/src/pages/Cambio.js`

**Seções da página:**
1. **Cotação ao vivo** — Card com USD, EUR, GBP em tempo real (polling a cada 30s)
2. **Nova Operação** — Stepper com 3 etapas:
   - Etapa 1: Tipo (compra/venda), moeda, valor em BRL ou moeda estrangeira
   - Etapa 2: Revisão (cotação do momento, spread, IOF)
   - Etapa 3: Confirmação e comprovante
3. **Histórico** — Tabela de operações anteriores: data, tipo, moeda, valor, taxa, status

**Integrações de API:**
```javascript
apiService.getCotacoes()                       // GET /api/cambio/cotacoes
apiService.simularCambio(params)              // POST /api/cambio/simulacoes
apiService.realizarOperacaoCambio(params)     // POST /api/cambio/operacoes
apiService.getOperacoesCambio(clienteId)      // GET /api/cambio/operacoes
```

**Mock data (fallback dev):**
```javascript
const mockCotacoes = [
  { moeda: 'USD', compra: 5.1250, venda: 5.1520, variacao: +0.15 },
  { moeda: 'EUR', compra: 5.5840, venda: 5.6120, variacao: -0.08 },
  { moeda: 'GBP', compra: 6.4320, venda: 6.4780, variacao: +0.22 },
];
```

### Nova página: `Seguros.js`

**Path:** `apps/frontend/aurix-web/src/pages/Seguros.js`

**Seções da página:**
1. **Meus Seguros** — Cards com apólices ativas (nome, tipo, prêmio mensal, vencimento, status)
2. **Contratar Seguro** — Stepper:
   - Etapa 1: Categoria (VIDA / AUTO / RESIDENCIAL / PRESTAMISTA)
   - Etapa 2: Cotação (dados do bem segurado, coberturas, franquia)
   - Etapa 3: Revisão do plano e valor do prêmio
   - Etapa 4: Contratação
3. **Sinistros** — Tabela com ocorrências, status, data abertura

**Integrações de API:**
```javascript
apiService.getApolicesAtivas(clienteId)        // GET /api/seguros/apolices
apiService.cotarSeguro(params)                 // POST /api/seguros/cotacoes
apiService.contratarSeguro(cotacaoId)          // POST /api/seguros/apolices
apiService.registrarSinistro(params)           // POST /api/seguros/sinistros
apiService.getSinistros(clienteId)             // GET /api/seguros/sinistros
```

### Nova página: `Consignado.js`

**Path:** `apps/frontend/aurix-web/src/pages/Consignado.js`

**Seções da página:**
1. **Margem Consignável** — Card com margem disponível, margem utilizada, limite por lei (35% da renda)
2. **Simular Empréstimo** — Form com valor desejado e prazo; exibe parcela, taxa e CET
3. **Meus Contratos** — Tabela: número do contrato, valor, parcelas, saldo devedor, próximo desconto
4. **Solicitar Portabilidade** — Form para migrar consignado de outro banco

**Integrações de API:**
```javascript
apiService.getMargemConsignavel(clienteId)     // GET /api/consignado/margem/{clienteId}
apiService.simularConsignado(params)           // POST /api/consignado/simulacoes
apiService.solicitarConsignado(params)         // POST /api/consignado/propostas
apiService.getContratosConsignado(clienteId)   // GET /api/consignado/contratos
```

### Roteamento — `App.js`

```jsx
import Cambio from './pages/Cambio';
import Seguros from './pages/Seguros';
import Consignado from './pages/Consignado';

// Adicionar nas rotas protegidas:
<Route path="/cambio"     element={<Cambio user={user} />} />
<Route path="/seguros"    element={<Seguros user={user} />} />
<Route path="/consignado" element={<Consignado user={user} />} />
```

### Sidebar — novos itens de menu

```jsx
// Em Sidebar.js, adicionar após "Cartões":
{ path: '/cambio',     label: 'Câmbio',      icon: <CurrencyExchangeIcon /> },
{ path: '/seguros',    label: 'Seguros',     icon: <SecurityIcon /> },
{ path: '/consignado', label: 'Consignado',  icon: <AccountBalanceIcon /> },
```

### Testes (co-localizados)

| Arquivo | Cobertura mínima |
|---------|-----------------|
| `Cambio.test.js` | renderiza título, exibe cotações mock, submete operação |
| `Seguros.test.js` | renderiza título, exibe apólices mock, abre modal de sinistro |
| `Consignado.test.js` | renderiza título, exibe margem, calcula simulação |

---

## Gap 6 — Inconsistência `TIPOS_CONTA` Frontend vs Backend

### Problema

`apps/frontend/aurix-admin/src/constants/index.js` define:
```javascript
export const TIPOS_CONTA = [
  { id: 'CORRENTE', name: 'Corrente' },
  { id: 'POUPANCA', name: 'Poupança' },
  { id: 'INVESTIMENTO', name: 'Investimento' },   // ← não existe no backend
];
```

O backend (`aurix-core`) aceita: `CORRENTE`, `POUPANCA`, `SALARIO`.

O tipo `INVESTIMENTO` está em `aurix-investimento` como entidade separada (`ContaInvestimento`), não como tipo de conta no core.

### Correção — `constants/index.js`

```javascript
export const TIPOS_CONTA = [
  { id: 'CORRENTE',    name: 'Corrente' },
  { id: 'POUPANCA',   name: 'Poupança' },
  { id: 'SALARIO',    name: 'Salário' },         // adicionar
  // INVESTIMENTO removido — gerenciado pelo módulo aurix-investimento
];
```

### Correção — `TIPOS_TRANSACAO`

O admin usa: `DEPOSITO`, `SAQUE`, `TRANSFERENCIA`, `PAGAMENTO`, `PIX`
O backend usa: `PIX`, `TED`, `DOC`, `BOLETO`, `DEBITO`, `CREDITO`

Alinhar ao vocabulário do backend:

```javascript
export const TIPOS_TRANSACAO = [
  { id: 'PIX',          name: 'PIX' },
  { id: 'TED',          name: 'TED' },
  { id: 'DOC',          name: 'DOC' },
  { id: 'BOLETO',       name: 'Boleto' },
  { id: 'DEBITO',       name: 'Débito' },
  { id: 'CREDITO',      name: 'Crédito' },
];
```

### Correção — `STATUS_COMPLIANCE`

O admin usa: `PENDENTE`, `EM_ANALISE`, `APROVADO`, `REJEITADO`
O backend usa: `LIMITE`, `BLOQUEIO`, `ALERTA`, `KYCA`

Esses são semanticamente diferentes — o backend trata de **tipos de alerta**, não status de processo. Solução: manter ambos os enums com semântica clara:

```javascript
// Status de processo (onboarding, KYC)
export const STATUS_PROCESSO = [
  { id: 'PENDENTE',    name: 'Pendente' },
  { id: 'EM_ANALISE', name: 'Em Análise' },
  { id: 'APROVADO',   name: 'Aprovado' },
  { id: 'REJEITADO',  name: 'Rejeitado' },
];

// Tipos de alerta de compliance (backend)
export const TIPOS_ALERTA_COMPLIANCE = [
  { id: 'LIMITE',    name: 'Limite' },
  { id: 'BLOQUEIO', name: 'Bloqueio' },
  { id: 'ALERTA',   name: 'Alerta' },
  { id: 'KYCA',     name: 'KYC/AML' },
];
```

---

## Gap 7 — `aurix-ai`: Expor Endpoints Mínimos

### Problema

`aurix-ai` tem Spring AI 1.0.0 e LangChain4j mas **zero controllers**. A rota `/api/ai` existe no Traefik, mas não retorna nada.

### Solução: Controller mínimo com capacidades reais

Criar `AiController.java` em `com.aurix.platform.ai.controller`:

#### Endpoint 1: Análise de extrato (ML + LLM)

```java
@PostMapping("/analise/extrato")
public ResponseEntity<AnaliseExtratoResponse> analisarExtrato(
    @RequestBody AnaliseExtratoRequest request) {
    // 1. Buscar transações do clienteId (via CoreApiClient)
    // 2. Calcular métricas (maior gasto, categoria dominante, tendência)
    // 3. Gerar insights em português com Spring AI (prompt template)
    // 4. Retornar insights + recomendações
}
```

#### Endpoint 2: Chatbot financeiro

```java
@PostMapping("/chat")
public ResponseEntity<ChatResponse> chat(@RequestBody ChatRequest request) {
    // Stateless Q&A sobre produtos AURIX e finanças pessoais
    // Spring AI ChatClient com system prompt pré-definido
}
```

#### Endpoint 3: Score de crédito com explicabilidade

```java
@GetMapping("/credito/score/{clienteId}")
public ResponseEntity<ScoreCreditoResponse> scoreCredito(@PathVariable Long clienteId) {
    // 1. Buscar dados do cliente (customer, histórico de crédito)
    // 2. Calcular score com modelo de regras
    // 3. Gerar explicação em português via LLM
    // 4. Retornar score + explicação + fatores de impacto
}
```

### `AiController` — estrutura completa

```java
@RestController
@RequestMapping("/api/ai")
@RequiredArgsConstructor
@Slf4j
public class AiController {

    private final AiService aiService;

    @GetMapping("/health")
    public ResponseEntity<Map<String, String>> health() {
        return ResponseEntity.ok(Map.of("status", "UP", "service", "aurix-ai"));
    }

    @PostMapping("/analise/extrato")
    public ResponseEntity<AnaliseExtratoResponse> analisarExtrato(
            @RequestBody AnaliseExtratoRequest request) {
        return ResponseEntity.ok(aiService.analisarExtrato(request));
    }

    @PostMapping("/chat")
    public ResponseEntity<ChatResponse> chat(@RequestBody ChatRequest request) {
        return ResponseEntity.ok(aiService.chat(request.getMensagem(), request.getClienteId()));
    }

    @GetMapping("/credito/score/{clienteId}")
    public ResponseEntity<ScoreCreditoResponse> scoreCredito(@PathVariable Long clienteId) {
        return ResponseEntity.ok(aiService.calcularScoreCredito(clienteId));
    }
}
```

### `AiService` — implementação inicial (regras + Spring AI)

```java
@Service
@Slf4j
public class AiService {

    // @Value("${spring.ai.openai.api-key:none}")
    // private String aiApiKey;
    // Uso do ChatClient do Spring AI — configurar com chave real em prod

    public AnaliseExtratoResponse analisarExtrato(AnaliseExtratoRequest request) {
        // Fase 1: lógica de regras (sem LLM)
        List<String> insights = new ArrayList<>();
        if (request.getTransacoes() == null || request.getTransacoes().isEmpty()) {
            return AnaliseExtratoResponse.vazio();
        }
        BigDecimal totalGastos = calcularTotalGastos(request.getTransacoes());
        String categoriaTop = identificarCategoriaDominante(request.getTransacoes());
        insights.add("Gasto total no período: " + formatarMoeda(totalGastos));
        insights.add("Categoria com maior gasto: " + categoriaTop);
        return AnaliseExtratoResponse.of(insights, totalGastos, categoriaTop);
    }

    public ScoreCreditoResponse calcularScoreCredito(Long clienteId) {
        // Regras simples — substituir por modelo ML quando disponível
        // Score base 500 + ajustes por comportamento
        return ScoreCreditoResponse.builder()
            .score(500)
            .faixa("MEDIO_RISCO")
            .explicacao("Score calculado com base no histórico de movimentações.")
            .build();
    }

    public ChatResponse chat(String mensagem, Long clienteId) {
        // Stub inicial — integrar com Spring AI ChatClient
        return ChatResponse.of("Olá! Sou o assistente AURIX. Em breve estarei totalmente integrado.");
    }
}
```

---

## Infraestrutura — Resumo das Alterações

| Arquivo | Alterações |
|---------|-----------|
| `apps/backend/pom.xml` | Adicionar 4 módulos: customer, kyc, fraud, notification |
| `infra/docker-compose.yml` | Corrigir porta aurix-ai para 8116, aurix-catalog para 8120 |
| `infra/traefik/dynamic.yml` | Corrigir URLs aurix-ai e aurix-catalog |
| `aurix-shared/Topics.java` | Adicionar 8 novos tópicos de fraude + financial |
| `aurix-shared/event/` | Adicionar 3 novos eventos tipados |
| `aurix-pix` | Criar `PagamentoPixProducer.java` |
| `aurix-cartoes` | Criar `CartaoTransacaoProducer.java` |
| `aurix-fraud` | Atualizar `FraudConsumer.java` com tópicos reais |
| `aurix-financiamento` | Implementar entities, services e controllers |
| `aurix-ai` | Criar `AiController.java` + `AiService.java` |
| `apps/frontend/aurix-web/src/pages/` | Criar `Cambio.js`, `Seguros.js`, `Consignado.js` + testes |
| `apps/frontend/aurix-admin/src/constants/index.js` | Alinhar `TIPOS_CONTA`, `TIPOS_TRANSACAO`, `STATUS_COMPLIANCE` |

---

## Tabela de Portas Atualizada

| Módulo | Porta | Status |
|--------|-------|--------|
| aurix-controller | 8090 | ✅ Correto |
| aurix-ai | **8116** | 🔧 Corrigido (era 8090) |
| aurix-catalog | **8120** | 🔧 Corrigido (era 8090) |
| aurix-customer | 8123 | ✅ OK |
| aurix-kyc | 8124 | ✅ OK |
| aurix-fraud | 8125 | ✅ OK |
| aurix-notification | 8126 | ✅ OK |
| aurix-acquirer | 8127 | 📋 Fase 2 |
| aurix-collections | 8128 | 📋 Fase 2 |
| aurix-guarantee | 8129 | 📋 Fase 2 |

---

## Ordem de Implementação

1. **Gap 2** (Maven parent) — sem dependências, desbloqueia CI imediatamente
2. **Gap 1** (conflito de portas) — alterações de config, sem dependência de código
3. **Gap 6** (constantes frontend) — 5 linhas de JS, alto impacto de UX
4. **Gap 3** (Fraud → PIX/Cartões) — adicionar produtores aos módulos existentes
5. **Gap 4** (Financiamento) — novo backend, mais trabalho
6. **Gap 5** (Frontend web) — 3 novas páginas, paralelizável com Gap 4
7. **Gap 7** (aurix-ai controller) — baixo risco, entrega valor imediato de produto

---

## Testes

- **Gap 1 e 2:** Validar com `mvn clean install` e `docker-compose up` — verificar que todos os módulos buildaram e health checks passam nas portas corretas
- **Gap 3:** `FraudConsumerIntegrationTest` com Testcontainers (Kafka + PostgreSQL): publicar evento PIX → verificar `ScoreTransacao` persistida
- **Gap 4:** `FinanciamentoServiceTest` com cálculo de tabela Price e SAC (valores verificáveis matematicamente)
- **Gap 5:** Testes de renderização co-localizados (Jest + React Testing Library)
- **Gap 6:** Nenhum teste adicional — mudança de constante simples
- **Gap 7:** `AiControllerTest` — endpoint health, chat e score retornam 200

*Análise base: business-gap-analysis.md — Julho 2026.*
