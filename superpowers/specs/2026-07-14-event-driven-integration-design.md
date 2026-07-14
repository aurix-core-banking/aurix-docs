# Event-Driven Integration — Design Spec (Fase 2, Sub-projeto 1)

> **Objetivo:** Conectar os 41 módulos da plataforma AUREUS via Kafka, eliminando tópicos fantasmas, padronizando nomenclatura ADR-0001, migrando foundation modules de JSON puro para `BaseEvent`, e criando consumers faltantes (Notification expandido, Audit universal).
> **Arquitetura:** Kafka como espinha dorsal de eventos. Foundation modules migram de `ObjectMapper.writeValueAsString()` para `JsonSerializer` com tipos concretos.

## Diagrama Alvo

```
[customer]──customer.cliente.criado.v1──▶[kyc]
[customer]──customer.cliente.criado.v1──▶[fraud]
[customer]──customer.cliente.criado.v1──▶[notification]
[kyc]──────kyc.solicitacao.aprovada.v1──▶[fraud]
[kyc]──────kyc.solicitacao.aprovada.v1──▶[notification]
[kyc]──────kyc.solicitacao.rejeitada.v1─▶[notification]
[fraud]────fraud.transacao.bloqueada.v1─▶[notification]
[core/pix]─core.transacao.realizada.v1──▶[fraud]
[core/pix]─core.transacao.realizada.v1──▶[notification]
[core/pix]─core.transacao.realizada.v1──▶[audit]
[cartoes]──cartoes.transacao.autorizada.v1▶[fraud]
[cartoes]──cartoes.transacao.autorizada.v1▶[notification]
[cartoes]──cartoes.cartao.emitido.v1────▶[audit]
[cartoes]──cartoes.fatura.fechada.v1────▶[notification]
[cartoes]──cartoes.fatura.paga.v1───────▶[notification]
[credit]───credit.solicitacao.criada.v1──▶[fraud]
[settlement]settlement.liquidez.processada.v1─▶[notification, audit]
[settlement]settlement.liquidez.rejeitada.v1──▶[notification, audit]
[tax]──────tax.imposto.calculado.v1─────▶[audit]
[billing]──billing.fatura.emitida.v1────▶[audit]
[billing]──billing.fatura.paga.v1───────▶[audit]
[bacen]────bacen.relatorio.gerado.v1────▶[audit]
[...]─────*.*.*.v1──────────────────────▶[audit] (universal consumer)
```

## 1. Standardização de Tópicos Foundation

Foundation modules migram de nomes flat para convenção ADR-0001 `<dominio>.<entidade>.<evento>.<versao>`:

| Atual (string literal) | Novo (constante Topics.java) |
|---|---|
| `cliente.criado` | `customer.cliente.criado.v1` |
| `cliente.atualizado` | `customer.cliente.atualizado.v1` |
| `cliente.status.alterado` | `customer.cliente.status.alterado.v1` |
| `kyc.aprovado` | `kyc.solicitacao.aprovada.v1` |
| `kyc.rejeitado` | `kyc.solicitacao.rejeitada.v1` |
| `fraude.transacao.bloqueada` | `fraud.transacao.bloqueada.v1` |
| `fraude.ocorrencia.criada` | `fraud.ocorrencia.criada.v1` |
| `fraude.score.alterado` | `fraud.score.alterado.v1` |
| `notificacao.enviada` | `notification.notificacao.enviada.v1` |
| `notificacao.falhou` | `notification.notificacao.falhou.v1` |

**Arquivos alterados:**
- `Topics.java` — adicionar 10 constantes novas
- `KafkaConfig.java` em cada foundation module — atualizar nome do tópico
- Producer/Consumer services — usar constantes de `Topics.java`

## 2. Migração Foundation: JSON Puro → BaseEvent Tipado

Cada foundation module hoje faz:
```java
String json = objectMapper.writeValueAsString(payload);
kafkaTemplate.send(topic, json);
```

Passa a fazer:
```java
ClienteCriadoEvent event = ClienteCriadoEvent.criado(clienteId, documento, nome, tipoPessoa, segmento);
kafkaTemplate.send(Topics.CUSTOMER_CLIENTE_CRIADO, event);
```

### Eventos novos no shared (`com.aureus.platform.shared.event`)

Cada evento estende `BaseEvent` e segue o padrão `ContaEvent` (com factory methods estáticos + getters):

- `ClienteCriadoEvent` — clienteId, documento, nome, tipoPessoa, segmento
- `ClienteAtualizadoEvent` — clienteId, documento, status
- `ClienteStatusAlteradoEvent` — clienteId, statusAnterior, statusAtual
- `KycAprovadoEvent` — clienteId, solicitacaoId, scoreRisco
- `KycRejeitadoEvent` — clienteId, solicitacaoId, motivo
- `TransacaoBloqueadaEvent` — clienteId, transacaoRef, score, risco, bloqueioId
- `OcorrenciaFraudEvent` — clienteId, ocorrenciaId, tipo, status
- `ScoreAlteradoEvent` — clienteId, score
- `NotificacaoEnviadaEvent` — notificacaoId, clienteId, canal, templateCodigo, status
- `NotificacaoFalhouEvent` — notificacaoId, clienteId, canal, motivo

### Consumers migram

De `@KafkaListener(topics = "cliente.criado")` com `String message` parseado manualmente para:
```java
@KafkaListener(topics = Topics.CUSTOMER_CLIENTE_CRIADO)
public void onClienteCriado(ClienteCriadoEvent event) { ... }
```

Requer `JsonDeserializer` tipado na config (já existe no `KafkaConfig` do shared, basta confiar no pacote).

## 3. Fix FraudConsumer: Tópicos Reais

FraudConsumer hoje escuta tópicos que não existem:

| Consumer atual (inexistente) | Producer real | Novo tópico alvo |
|---|---|---|
| `pix.transferencia.criada` | `aureus-pix` → `core.transacao.realizada.v1` | `core.transacao.realizada.v1` |
| `cartoes.transacao.autorizada` | `aureus-cartoes` → `aureus.cartao.transacao.autorizada` | `cartoes.transacao.autorizada.v1` (após migração) |
| `credito.solicitacao.criada` | `aureus-credit` → (não produz) | `credit.solicitacao.criada.v1` (novo producer) |

FraudConsumer adapta `TransacaoBloqueadaEvent` para receber `TransacaoEvent` (shared) para PIX, `CartaoTransacaoAutorizadaEvent` para cartões, e `SolicitacaoCreditoCriadaEvent` para crédito.

## 4. Migração Cartões: Tópicos ADR-0001

| Atual | Novo |
|---|---|
| `aureus.cartao.emitido` | `cartoes.cartao.emitido.v1` |
| `aureus.cartao.transacao.autorizada` | `cartoes.transacao.autorizada.v1` |
| `aureus.cartao.transacao.estornada` | `cartoes.transacao.estornada.v1` |
| `aureus.cartao.fatura.fechada` | `cartoes.fatura.fechada.v1` |
| `aureus.cartao.fatura.paga` | `cartoes.fatura.paga.v1` |

### Eventos migrados para shared

Eventos Java records locais viram `BaseEvent` subclasses no shared:

- `CartaoEmitidoEvent` → `CartaoEmitidoEvent extends BaseEvent` — cartaoId, contaId, nomePortador, bandeira, tipo, tenantId
- `TransacaoAutorizadaEvent` → `CartaoTransacaoAutorizadaEvent extends BaseEvent` — codigoTransacao, cartaoId, valor, estabelecimento, autorizacao, status, tenantId
- `TransacaoEstornadaEvent` → `CartaoTransacaoEstornadaEvent extends BaseEvent` — codigoTransacao, cartaoId, valor, tenantId
- `FaturaFechadaEvent` → `CartaoFaturaFechadaEvent extends BaseEvent` — faturaId, cartaoId, mesReferencia, anoReferencia, valor, tenantId
- `FaturaPagaEvent` → `CartaoFaturaPagaEvent extends BaseEvent` — faturaId, cartaoId, valorPago, tenantId

KafkaConfig do cartões usa constantes de `Topics.java` e template tipado.

## 5. Credit: Producer para `credit.solicitacao.criada.v1`

`aureus-credit` hoje não produz Kafka. Adicionar:

- Dependência `spring-kafka` no `pom.xml`
- `CreditKafkaConfig` — `KafkaTemplate<String, SolicitacaoCreditoCriadaEvent>`
- `SolicitacaoCreditoCriadaEvent extends BaseEvent` no shared — solicitacaoId, clienteId, valor, tipoCredito, dataSolicitacao, tenantId
- Producer no `SolicitacaoCreditoService` (ou serviço equivalente) — publicar evento após criar solicitação
- Novo tópico `credit.solicitacao.criada.v1` em `Topics.java`

## 6. Tax, Billing, Bacen: Producers Faltantes

Tópicos já definidos em `Topics.java` mas sem producer:

| Tópico | Módulo | Onde produzir |
|---|---|---|
| `tax.imposto.calculado.v1` | `aureus-tax` | No serviço de cálculo de imposto, após `calcularImposto()` |
| `billing.fatura.emitida.v1` | `aureus-billing` | Após emitir fatura |
| `billing.fatura.paga.v1` | `aureus-billing` | Após registrar pagamento de fatura |
| `bacen.relatorio.gerado.v1` | `aureus-bacen` | Após gerar relatório BACEN |
| `bacen.relatorio.enviado.v1` | `aureus-bacen` | Após enviar relatório ao BACEN |

Cada módulo precisa: dependência `spring-kafka`, KafkaConfig, evento tipado no shared (ou usar `BaseEvent` genérico), producer no service.

## 7. Notification: Expandir Consumers

Notification hoje consome 4 tópicos. Passa a consumir:

| Tópico | Evento | Notificação gerada |
|---|---|---|
| `core.transacao.realizada.v1` (já existe) | `TransacaoEvent` | "Transferência PIX recebida" |
| `cartoes.transacao.autorizada.v1` (novo) | `CartaoTransacaoAutorizadaEvent` | "Compra no cartão aprovada" |
| `cartoes.fatura.fechada.v1` (novo) | `CartaoFaturaFechadaEvent` | "Fatura do cartão fechada" |
| `cartoes.fatura.paga.v1` (novo) | `CartaoFaturaPagaEvent` | "Fatura do cartão paga" |
| `settlement.liquidez.processada.v1` (já existe) | `LiquidezEvent` | "Liquidação processada" |
| `settlement.liquidez.rejeitada.v1` (já existe) | `LiquidezEvent` | "Liquidação rejeitada" |
| `consignado.contrato.assinado.v1` (novo) | `ConsignadoContratoAssinadoEvent` | "Contrato consignado assinado" |
| `financiamento.contrato.assinado.v1` (novo) | `FinanciamentoContratoAssinadoEvent` | "Contrato financiamento assinado" |
| `investimento.ordem.executada.v1` (novo) | `InvestimentoOrdemExecutadaEvent` | "Ordem de investimento executada" |
| `seguros.apolice.emitida.v1` (novo) | `SegurosApoliceEmitidaEvent` | "Apólice de seguro emitida" |

## 8. Audit: Consumer Universal

`aureus-audit` não tem Kafka hoje. Adicionar `AuditEventConsumer` que escuta **todos** os tópicos e registra cada evento recebido como `LogAuditoria`:

```java
@Component
public class AuditEventConsumer {
    @KafkaListener(topics = {
        Topics.CUSTOMER_CLIENTE_CRIADO,
        Topics.CUSTOMER_CLIENTE_ATUALIZADO,
        Topics.CUSTOMER_CLIENTE_STATUS_ALTERADO,
        Topics.KYC_SOLICITACAO_APROVADA,
        Topics.KYC_SOLICITACAO_REJEITADA,
        Topics.FRAUD_TRANSACAO_BLOQUEADA,
        Topics.FRAUD_OCORRENCIA_CRIADA,
        Topics.FRAUD_SCORE_ALTERADO,
        Topics.CORE_CONTA_CRIADA,
        Topics.CORE_CONTA_ATUALIZADA,
        Topics.CORE_CONTA_BLOQUEADA,
        Topics.CORE_TRANSACAO_REALIZADA,
        Topics.CORE_TRANSACAO_LIQUIDADA,
        Topics.CORE_TRANSACAO_CONCILIADA,
        Topics.SETTLEMENT_LIQUIDEZ_PROCESSADA,
        Topics.SETTLEMENT_LIQUIDEZ_REJEITADA,
        Topics.TAX_IMPOSTO_CALCULADO
    })
    public void onEvent(BaseEvent event) {
        log.info("Audit event: type={}, source={}, id={}", event.getEventType(), event.getSource(), event.getEventId());
        // Persistir como LogAuditoria no banco do audit
    }
}
```

Requer: dependência `spring-kafka`, KafkaConfig, deserializador para `BaseEvent` (já existe no shared).

## 9. Standardização Produtos Restantes: Tópicos ADR-0001

| Módulo | Atual | Novo |
|---|---|---|
| `aureus-cambio` | `cambio-cotacao-atualizada` | `cambio.cotacao.atualizada.v1` |
| `aureus-cambio` | `cambio-contrato-fechado` | `cambio.contrato.fechado.v1` |
| `aureus-cambio` | `cambio-contrato-liquidado` | `cambio.contrato.liquidado.v1` |
| `aureus-cambio` | `cambio-remessa-processada` | `cambio.remessa.processada.v1` |
| `aureus-consignado` | `consignado-contrato-assinado` | `consignado.contrato.assinado.v1` |
| `aureus-consignado` | `consignado-parcela-debitada` | `consignado.parcela.debitada.v1` |
| `aureus-consignado` | `consignado-margem-atualizada` | `consignado.margem.atualizada.v1` |
| `aureus-consignado` | `consignado-contrato-liquidado` | `consignado.contrato.liquidado.v1` |
| `aureus-financiamento` | `financiamento-simulacao-realizada` | `financiamento.simulacao.realizada.v1` |
| `aureus-financiamento` | `financiamento-contrato-assinado` | `financiamento.contrato.assinado.v1` |
| `aureus-financiamento` | `financiamento-parcela-paga` | `financiamento.parcela.paga.v1` |
| `aureus-financiamento` | `financiamento-contrato-liquidado` | `financiamento.contrato.liquidado.v1` |
| `aureus-financiamento` | `financiamento-garantia-registrada` | `financiamento.garantia.registrada.v1` |
| `aureus-investimento` | `investimento-conta-criada` | `investimento.conta.criada.v1` |
| `aureus-investimento` | `investimento-ordem-executada` | `investimento.ordem.executada.v1` |
| `aureus-investimento` | `investimento-resgate-processado` | `investimento.resgate.processado.v1` |
| `aureus-poupanca` | `poupanca-conta-criada` | `poupanca.conta.criada.v1` |
| `aureus-poupanca` | `poupanca-deposito-realizado` | `poupanca.deposito.realizado.v1` |
| `aureus-poupanca` | `poupanca-saque-realizado` | `poupanca.saque.realizado.v1` |
| `aureus-poupanca` | `poupanca-rendimento-creditado` | `poupanca.rendimento.creditado.v1` |
| `aureus-salario` | `salario-conta-criada` | `salario.conta.criada.v1` |
| `aureus-salario` | `salario-creditado` | `salario.creditado.v1` |
| `aureus-salario` | `salario-portabilidade-solicitada` | `salario.portabilidade.solicitada.v1` |
| `aureus-seguros` | `seguros-apolice-emitida` | `seguros.apolice.emitida.v1` |
| `aureus-seguros` | `seguros-premio-pago` | `seguros.premio.pago.v1` |
| `aureus-seguros` | `seguros-sinistro-aberto` | `seguros.sinistro.aberto.v1` |
| `aureus-seguros` | `seguros-sinistro-liquidado` | `seguros.sinistro.liquidado.v1` |

Cada evento record local vira `BaseEvent` subclass no shared, adicionado em lote.

## 10. Topics.java: Constantes Finais

Após a migração, `Topics.java` conterá ~60 constantes organizadas por domínio:

```java
// ===== customer =====
CUSTOMER_CLIENTE_CRIADO = "customer.cliente.criado.v1"
CUSTOMER_CLIENTE_ATUALIZADO = "customer.cliente.atualizado.v1"
CUSTOMER_CLIENTE_STATUS_ALTERADO = "customer.cliente.status.alterado.v1"

// ===== kyc =====
KYC_SOLICITACAO_APROVADA = "kyc.solicitacao.aprovada.v1"
KYC_SOLICITACAO_REJEITADA = "kyc.solicitacao.rejeitada.v1"

// ===== fraud =====
FRAUD_TRANSACAO_BLOQUEADA = "fraud.transacao.bloqueada.v1"
FRAUD_OCORRENCIA_CRIADA = "fraud.ocorrencia.criada.v1"
FRAUD_SCORE_ALTERADO = "fraud.score.alterado.v1"

// ===== notification =====
NOTIFICATION_NOTIFICACAO_ENVIADA = "notification.notificacao.enviada.v1"
NOTIFICATION_NOTIFICACAO_FALHOU = "notification.notificacao.falhou.v1"

// ===== credit =====
CREDIT_SOLICITACAO_CRIADA = "credit.solicitacao.criada.v1"

// ===== cartoes (migrado) =====
CARTOES_CARTAO_EMITIDO = "cartoes.cartao.emitido.v1"
CARTOES_TRANSACAO_AUTORIZADA = "cartoes.transacao.autorizada.v1"
CARTOES_TRANSACAO_ESTORNADA = "cartoes.transacao.estornada.v1"
CARTOES_FATURA_FECHADA = "cartoes.fatura.fechada.v1"
CARTOES_FATURA_PAGA = "cartoes.fatura.paga.v1"

// ===== cambio, consignado, financiamento, investimento, poupanca, salario, seguros =====
// (constantes para todos os tópicos listados na seção 9)
```

## Estratégia de Migração

Cada módulo segue o mesmo padrão:
1. Adicionar evento tipado no shared
2. Adicionar constante em `Topics.java`
3. Atualizar KafkaConfig para usar template tipado
4. Substituir `ObjectMapper.writeValueAsString()` por `kafkaTemplate.send(topic, event)`
5. Atualizar consumer para receber tipo concreto
6. Remover constante/string literal antigo
7. Compilar + testar

Ordem de execução:
1. Shared (eventos + Topics)
2. Foundation (Customer → KYC → Fraud → Notification)
3. Cartões (eventos + tópicos)
4. Credit (novo producer)
5. Tax, Billing, Bacen (producers faltantes)
6. Product modules (cambio, consignado, financiamento, investimento, poupanca, salario, seguros)
7. Notification (expandir consumers)
8. Audit (consumer universal)

## Riscos

- **Mudança de nome de tópico**: producers e consumers devem ser alterados no mesmo deploy para evitar perda de mensagens. Usar tópico antigo + novo em paralelo durante transição (fase de dual-write) ou fazer deploy dos consumers primeiro.
- **Quebra de consumer existente**: `aureus-core` consome seus próprios eventos — não alterar nome desses tópicos (já seguem ADR-0001).
- **Eventos records → BaseEvent**: muda serialização. Garantir que `JsonDeserializer` confia no pacote `com.aureus.platform.shared.event` (já configurado).
