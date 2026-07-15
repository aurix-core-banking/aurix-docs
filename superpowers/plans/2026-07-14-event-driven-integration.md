# Event-Driven Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Conectar 41 módulos via Kafka — foundation modules migram de JSON puro para `BaseEvent` tipado, Fraud passa a ouvir tópicos reais, Notification expande para 14 tópicos, Audit vira consumer universal.

**Architecture:** 8 tarefas independentes, ordem definida por dependência de dados. Cada tarefa: criar evento no shared → adicionar constante em Topics → atualizar KafkaConfig → trocar producer → trocar consumer → compilar + testar.

**Tech Stack:** Spring Boot 4.1.0, Java 25, Kafka 3.x, `JsonSerializer`/`JsonDeserializer` (já configurado no shared), `TopicBuilder`.

## Global Constraints

- Eventos tipados vão em `aureus-shared/src/main/java/com/aureus/platform/shared/event/` e extendem `BaseEvent`
- Constantes de tópico vão em `Topics.java` no mesmo pacote
- Foundation modules trocam `KafkaTemplate<String, String>` por `KafkaTemplate<String, Object>` (já existe bean no shared `KafkaConfig`)
- Consumers mudam de `String message` parseado para receber tipo concreto do evento
- Foundation modules mantêm `@Configuration("<nome>KafkaConfig")` para evitar conflito de beans com shared
- Product modules mantêm `KafkaTemplate<String, Object>` (já é o padrão deles)
- Todos os tópicos têm 3 partitions, 1 replica

---

### Task 1: Eventos Tipados + Tópicos no Shared

**Files:**
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/ClienteCriadoEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/ClienteAtualizadoEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/ClienteStatusAlteradoEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/KycAprovadoEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/KycRejeitadoEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/TransacaoBloqueadaEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/OcorrenciaFraudEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/ScoreAlteradoEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/NotificacaoEnviadaEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/NotificacaoFalhouEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/SolicitacaoCreditoCriadaEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/CartaoTransacaoAutorizadaEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/CartaoEmitidoEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/CartaoTransacaoEstornadaEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/CartaoFaturaFechadaEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/CartaoFaturaPagaEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/ConsignadoContratoAssinadoEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/FinanciamentoContratoAssinadoEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/InvestimentoOrdemExecutadaEvent.java`
- Create: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/SegurosApoliceEmitidaEvent.java`
- Modify: `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/Topics.java`

**Interfaces:**
- Cada evento estende `BaseEvent`, segue o padrão `ContaEvent` (subclasses com campos + factory methods)
- Topics.java ganha constantes para cada novo tópico

- [ ] **Step 1: Create ClienteCriadoEvent**

File `backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/ClienteCriadoEvent.java`:

```java
package com.aureus.platform.shared.event;

import java.time.LocalDateTime;

public class ClienteCriadoEvent extends BaseEvent {
    private Long clienteId;
    private String documento;
    private String nome;
    private String tipoPessoa;
    private String segmento;

    public static ClienteCriadoEvent criado(Long clienteId, String documento, String nome, String tipoPessoa, String segmento) {
        ClienteCriadoEvent event = new ClienteCriadoEvent();
        event.setEventId(java.util.UUID.randomUUID().toString());
        event.setEventType("CLIENTE_CRIADO");
        event.setSource("aureus-customer");
        event.setTimestamp(LocalDateTime.now());
        event.setCorrelationId(java.util.UUID.randomUUID().toString());
        event.clienteId = clienteId;
        event.documento = documento;
        event.nome = nome;
        event.tipoPessoa = tipoPessoa;
        event.segmento = segmento;
        return event;
    }

    public ClienteCriadoEvent() {}

    public Long getClienteId() { return clienteId; }
    public void setClienteId(Long clienteId) { this.clienteId = clienteId; }
    public String getDocumento() { return documento; }
    public void setDocumento(String documento) { this.documento = documento; }
    public String getNome() { return nome; }
    public void setNome(String nome) { this.nome = nome; }
    public String getTipoPessoa() { return tipoPessoa; }
    public void setTipoPessoa(String tipoPessoa) { this.tipoPessoa = tipoPessoa; }
    public String getSegmento() { return segmento; }
    public void setSegmento(String segmento) { this.segmento = segmento; }
}
```

- [ ] **Step 2: Create remaining foundation events (same pattern)**

Create `ClienteAtualizadoEvent` (clienteId, documento, status), `ClienteStatusAlteradoEvent` (clienteId, statusAnterior, statusAtual), `KycAprovadoEvent` (clienteId, solicitacaoId, scoreRisco), `KycRejeitadoEvent` (clienteId, solicitacaoId, motivo), `TransacaoBloqueadaEvent` (clienteId, transacaoRef, score, risco, bloqueioId), `OcorrenciaFraudEvent` (clienteId, ocorrenciaId, tipo, status), `ScoreAlteradoEvent` (clienteId, transacaoRef, score, risco), `NotificacaoEnviadaEvent` (notificacaoId, clienteId, canal, templateCodigo, status), `NotificacaoFalhouEvent` (notificacaoId, clienteId, canal, motivo).

- [ ] **Step 3: Create SolicitacaoCreditoCriadaEvent**

```java
package com.aureus.platform.shared.event;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public class SolicitacaoCreditoCriadaEvent extends BaseEvent {
    private Long solicitacaoId;
    private Long clienteId;
    private BigDecimal valor;
    private String tipoCredito;
    private LocalDateTime dataSolicitacao;

    public static SolicitacaoCreditoCriadaEvent criada(Long solicitacaoId, Long clienteId, BigDecimal valor, String tipoCredito) {
        SolicitacaoCreditoCriadaEvent event = new SolicitacaoCreditoCriadaEvent();
        event.setEventId(java.util.UUID.randomUUID().toString());
        event.setEventType("SOLICITACAO_CREDITO_CRIADA");
        event.setSource("aureus-credit");
        event.setTimestamp(LocalDateTime.now());
        event.setCorrelationId(java.util.UUID.randomUUID().toString());
        event.solicitacaoId = solicitacaoId;
        event.clienteId = clienteId;
        event.valor = valor;
        event.tipoCredito = tipoCredito;
        event.dataSolicitacao = LocalDateTime.now();
        return event;
    }

    public SolicitacaoCreditoCriadaEvent() {}

    public Long getSolicitacaoId() { return solicitacaoId; }
    public void setSolicitacaoId(Long solicitacaoId) { this.solicitacaoId = solicitacaoId; }
    public Long getClienteId() { return clienteId; }
    public void setClienteId(Long clienteId) { this.clienteId = clienteId; }
    public BigDecimal getValor() { return valor; }
    public void setValor(BigDecimal valor) { this.valor = valor; }
    public String getTipoCredito() { return tipoCredito; }
    public void setTipoCredito(String tipoCredito) { this.tipoCredito = tipoCredito; }
    public LocalDateTime getDataSolicitacao() { return dataSolicitacao; }
    public void setDataSolicitacao(LocalDateTime dataSolicitacao) { this.dataSolicitacao = dataSolicitacao; }
}
```

- [ ] **Step 4: Create Cartoes shared events**

Create `CartaoTransacaoAutorizadaEvent` (codigoTransacao, cartaoId, valor, estabelecimento, autorizacao, status, tenantId), `CartaoEmitidoEvent` (cartaoId, contaId, nomePortador, bandeira, tipo, tenantId), `CartaoTransacaoEstornadaEvent` (codigoTransacao, cartaoId, valor, tenantId), `CartaoFaturaFechadaEvent` (faturaId, cartaoId, mesReferencia, anoReferencia, valor, tenantId), `CartaoFaturaPagaEvent` (faturaId, cartaoId, valorPago, tenantId).

Cada um com factory method + construtor vazio + getters/setters, seguindo padrão `BaseEvent`.

- [ ] **Step 5: Create product module shared events**

Create `ConsignadoContratoAssinadoEvent` (contratoId, clienteId, valorTotal, prazoMeses, valorParcela, tenantId), `FinanciamentoContratoAssinadoEvent` (id, clienteId, contaCorrenteId, tipo, valorFinanciado, prazoMeses, taxaJuros, tenantId), `InvestimentoOrdemExecutadaEvent` (id, contaId, produtoId, valor, quantidade, tenantId), `SegurosApoliceEmitidaEvent` (apoliceId, numero, clienteId, produtoId, premioTotal, dataInicio, dataFim, tenantId).

- [ ] **Step 6: Add all topic constants to Topics.java**

Add these constants after existing ones:

```java
// ===== customer =====
public static final String CUSTOMER_CLIENTE_CRIADO = "customer.cliente.criado.v1";
public static final String CUSTOMER_CLIENTE_ATUALIZADO = "customer.cliente.atualizado.v1";
public static final String CUSTOMER_CLIENTE_STATUS_ALTERADO = "customer.cliente.status.alterado.v1";

// ===== kyc =====
public static final String KYC_SOLICITACAO_APROVADA = "kyc.solicitacao.aprovada.v1";
public static final String KYC_SOLICITACAO_REJEITADA = "kyc.solicitacao.rejeitada.v1";

// ===== fraud =====
public static final String FRAUD_TRANSACAO_BLOQUEADA = "fraud.transacao.bloqueada.v1";
public static final String FRAUD_OCORRENCIA_CRIADA = "fraud.ocorrencia.criada.v1";
public static final String FRAUD_SCORE_ALTERADO = "fraud.score.alterado.v1";

// ===== notification =====
public static final String NOTIFICATION_NOTIFICACAO_ENVIADA = "notification.notificacao.enviada.v1";
public static final String NOTIFICATION_NOTIFICACAO_FALHOU = "notification.notificacao.falhou.v1";

// ===== credit =====
public static final String CREDIT_SOLICITACAO_CRIADA = "credit.solicitacao.criada.v1";

// ===== cartoes =====
public static final String CARTOES_CARTAO_EMITIDO = "cartoes.cartao.emitido.v1";
public static final String CARTOES_TRANSACAO_AUTORIZADA = "cartoes.transacao.autorizada.v1";
public static final String CARTOES_TRANSACAO_ESTORNADA = "cartoes.transacao.estornada.v1";
public static final String CARTOES_FATURA_FECHADA = "cartoes.fatura.fechada.v1";
public static final String CARTOES_FATURA_PAGA = "cartoes.fatura.paga.v1";

// ===== consignado =====
public static final String CONSIGNADO_CONTRATO_ASSINADO = "consignado.contrato.assinado.v1";

// ===== financiamento =====
public static final String FINANCIAMENTO_CONTRATO_ASSINADO = "financiamento.contrato.assinado.v1";

// ===== investimento =====
public static final String INVESTIMENTO_ORDEM_EXECUTADA = "investimento.ordem.executada.v1";

// ===== seguros =====
public static final String SEGUROS_APOLICE_EMITIDA = "seguros.apolice.emitida.v1";

// ===== cambio =====
public static final String CAMBIO_COTACAO_ATUALIZADA = "cambio.cotacao.atualizada.v1";
public static final String CAMBIO_CONTRATO_FECHADO = "cambio.contrato.fechado.v1";
public static final String CAMBIO_CONTRATO_LIQUIDADO = "cambio.contrato.liquidado.v1";
public static final String CAMBIO_REMESSA_PROCESSADA = "cambio.remessa.processada.v1";

// ===== consignado (full) =====
public static final String CONSIGNADO_PARCELA_DEBITADA = "consignado.parcela.debitada.v1";
public static final String CONSIGNADO_MARGEM_ATUALIZADA = "consignado.margem.atualizada.v1";
public static final String CONSIGNADO_CONTRATO_LIQUIDADO = "consignado.contrato.liquidado.v1";

// ===== financiamento (full) =====
public static final String FINANCIAMENTO_SIMULACAO_REALIZADA = "financiamento.simulacao.realizada.v1";
public static final String FINANCIAMENTO_PARCELA_PAGA = "financiamento.parcela.paga.v1";
public static final String FINANCIAMENTO_CONTRATO_LIQUIDADO = "financiamento.contrato.liquidado.v1";
public static final String FINANCIAMENTO_GARANTIA_REGISTRADA = "financiamento.garantia.registrada.v1";

// ===== investimento (full) =====
public static final String INVESTIMENTO_CONTA_CRIADA = "investimento.conta.criada.v1";
public static final String INVESTIMENTO_RESGATE_PROCESSADO = "investimento.resgate.processado.v1";

// ===== poupanca =====
public static final String POUPANCA_CONTA_CRIADA = "poupanca.conta.criada.v1";
public static final String POUPANCA_DEPOSITO_REALIZADO = "poupanca.deposito.realizado.v1";
public static final String POUPANCA_SAQUE_REALIZADO = "poupanca.saque.realizado.v1";
public static final String POUPANCA_RENDIMENTO_CREDITADO = "poupanca.rendimento.creditado.v1";

// ===== salario =====
public static final String SALARIO_CONTA_CRIADA = "salario.conta.criada.v1";
public static final String SALARIO_CREDITADO = "salario.creditado.v1";
public static final String SALARIO_PORTABILIDADE_SOLICITADA = "salario.portabilidade.solicitada.v1";

// ===== seguros (full) =====
public static final String SEGUROS_PREMIO_PAGO = "seguros.premio.pago.v1";
public static final String SEGUROS_SINISTRO_ABERTO = "seguros.sinistro.aberto.v1";
public static final String SEGUROS_SINISTRO_LIQUIDADO = "seguros.sinistro.liquidado.v1";
```

- [ ] **Step 7: Compile and test**

```bash
mvn -pl backend/aureus-shared -am compile -DskipTests
```
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add backend/aureus-shared/src/main/java/com/aureus/platform/shared/event/
git commit -m "feat(shared): add typed event classes and ADR-0001 topic constants"
```

---

### Task 2: Aureus-Customer — JSON → Typed Events + ADR Topics

**Files:**
- Modify: `backend/aureus-customer/src/main/java/com/aureus/platform/customer/service/ClienteProducer.java`
- Modify: `backend/aureus-customer/src/main/java/com/aureus/platform/customer/config/KafkaConfig.java`

**Depends on:** Task 1 (event classes + Topics constants)

- [ ] **Step 1: Update ClienteProducer**

Replace `KafkaTemplate<String, String>` with `KafkaTemplate<String, Object>`. Remove `ObjectMapper`. Use typed events:

```java
package com.aureus.platform.customer.service;

import com.aureus.platform.customer.entity.Cliente;
import com.aureus.platform.shared.event.ClienteCriadoEvent;
import com.aureus.platform.shared.event.ClienteAtualizadoEvent;
import com.aureus.platform.shared.event.ClienteStatusAlteradoEvent;
import com.aureus.platform.shared.event.Topics;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class ClienteProducer {
    private static final Logger log = LoggerFactory.getLogger(ClienteProducer.class);
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public ClienteProducer(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void clienteCriado(Cliente cliente) {
        ClienteCriadoEvent event = ClienteCriadoEvent.criado(
            cliente.getId(), cliente.getDocumento(), cliente.getNomeCompleto(),
            cliente.getTipoPessoa(), cliente.getSegmento());
        kafkaTemplate.send(Topics.CUSTOMER_CLIENTE_CRIADO, String.valueOf(cliente.getId()), event);
        log.info("Evento {} publicado para clienteId={}", event.getEventType(), cliente.getId());
    }

    public void clienteAtualizado(Cliente cliente) {
        ClienteAtualizadoEvent event = ClienteAtualizadoEvent.atualizado(
            cliente.getId(), cliente.getDocumento(), cliente.getStatus());
        kafkaTemplate.send(Topics.CUSTOMER_CLIENTE_ATUALIZADO, String.valueOf(cliente.getId()), event);
    }

    public void clienteStatusAlterado(Cliente cliente, String statusAnterior) {
        ClienteStatusAlteradoEvent event = ClienteStatusAlteradoEvent.alterado(
            cliente.getId(), statusAnterior, cliente.getStatus());
        kafkaTemplate.send(Topics.CUSTOMER_CLIENTE_STATUS_ALTERADO, String.valueOf(cliente.getId()), event);
    }
}
```

- [ ] **Step 2: Update Customer KafkaConfig**

Replace old topic names with new ADR-0001 constants:

```java
package com.aureus.platform.customer.config;

import com.aureus.platform.shared.event.Topics;
import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration("customerKafkaConfig")
public class KafkaConfig {
    @Bean
    public NewTopic clienteCriadoTopic() {
        return TopicBuilder.name(Topics.CUSTOMER_CLIENTE_CRIADO).partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic clienteAtualizadoTopic() {
        return TopicBuilder.name(Topics.CUSTOMER_CLIENTE_ATUALIZADO).partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic clienteStatusAlteradoTopic() {
        return TopicBuilder.name(Topics.CUSTOMER_CLIENTE_STATUS_ALTERADO).partitions(3).replicas(1).build();
    }
}
```

- [ ] **Step 3: Compile and test**

```bash
mvn -pl backend/aureus-customer -am test -Dtest="*Test"
```
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add backend/aureus-customer/
git commit -m "feat(customer): migrate to typed events + ADR-0001 topics"
```

---

### Task 3: Aureus-KYC — JSON → Typed Events + ADR Topics

**Files:**
- Modify: `backend/aureus-kyc/src/main/java/com/aureus/platform/kyc/service/KycProducer.java`
- Modify: `backend/aureus-kyc/src/main/java/com/aureus/platform/kyc/service/KycConsumer.java`
- Modify: `backend/aureus-kyc/src/main/java/com/aureus/platform/kyc/config/KafkaConfig.java`

**Depends on:** Task 1

- [ ] **Step 1: Update KycProducer**

Replace `KafkaTemplate<String, String>` with `KafkaTemplate<String, Object>`. Use `KycAprovadoEvent` / `KycRejeitadoEvent` and `Topics.KYC_SOLICITACAO_APROVADA` / `Topics.KYC_SOLICITACAO_REJEITADA`.

```java
public void kycAprovado(SolicitacaoKYC solicitacao) {
    KycAprovadoEvent event = KycAprovadoEvent.aprovado(
        solicitacao.getClienteId(), solicitacao.getId(), solicitacao.getScoreRisco());
    kafkaTemplate.send(Topics.KYC_SOLICITACAO_APROVADA, String.valueOf(solicitacao.getClienteId()), event);
}

public void kycRejeitado(SolicitacaoKYC solicitacao, String motivo) {
    KycRejeitadoEvent event = KycRejeitadoEvent.rejeitado(
        solicitacao.getClienteId(), solicitacao.getId(), motivo);
    kafkaTemplate.send(Topics.KYC_SOLICITACAO_REJEITADA, String.valueOf(solicitacao.getClienteId()), event);
}
```

- [ ] **Step 2: Update KycConsumer**

Change from `@KafkaListener(topics = "cliente.criado") String message` to `@KafkaListener(topics = Topics.CUSTOMER_CLIENTE_CRIADO) ClienteCriadoEvent event`. Replace manual JSON parsing with `event.getClienteId()` etc.

- [ ] **Step 3: Update KYC KafkaConfig**

Replace `"kyc.aprovado"` → `Topics.KYC_SOLICITACAO_APROVADA`, `"kyc.rejeitado"` → `Topics.KYC_SOLICITACAO_REJEITADA`.

- [ ] **Step 4: Compile and test**

```bash
mvn -pl backend/aureus-kyc -am test -Dtest="*Test"
```
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add backend/aureus-kyc/
git commit -m "feat(kyc): migrate to typed events + ADR-0001 topics"
```

---

### Task 4: Aureus-Fraud — JSON → Typed Events + Fix Consumer Topics

**Files:**
- Modify: `backend/aureus-fraud/src/main/java/com/aureus/platform/fraud/service/FraudProducer.java`
- Modify: `backend/aureus-fraud/src/main/java/com/aureus/platform/fraud/service/FraudConsumer.java`
- Modify: `backend/aureus-fraud/src/main/java/com/aureus/platform/fraud/config/KafkaConfig.java`

**Depends on:** Task 1

- [ ] **Step 1: Update FraudProducer**

Replace JSON strings with typed events (`TransacaoBloqueadaEvent`, `OcorrenciaFraudEvent`, `ScoreAlteradoEvent`). Use `Topics.FRAUD_TRANSACAO_BLOQUEADA`, `Topics.FRAUD_OCORRENCIA_CRIADA`, `Topics.FRAUD_SCORE_ALTERADO`.

- [ ] **Step 2: Update FraudConsumer — fix topic names**

Change consumers to listen on **real** topics:

```java
@KafkaListener(topics = Topics.CUSTOMER_CLIENTE_CRIADO, groupId = "aureus-fraud-group")
public void onClienteCriado(ClienteCriadoEvent event) {
    criarOcorrencia(event.getClienteId(), "CLIENTE_CRIADO", null);
}

@KafkaListener(topics = Topics.KYC_SOLICITACAO_APROVADA, groupId = "aureus-fraud-group")
public void onKycAprovado(KycAprovadoEvent event) {
    criarOcorrencia(event.getClienteId(), "KYC_APROVADO", null);
}

@KafkaListener(topics = Topics.CORE_TRANSACAO_REALIZADA, groupId = "aureus-fraud-group")
public void onPixTransferencia(TransacaoEvent event) {
    criarOcorrencia(Long.valueOf(event.getClienteId()), "PIX_TRANSFERENCIA", event.getTransacaoId());
}

@KafkaListener(topics = Topics.CREDIT_SOLICITACAO_CRIADA, groupId = "aureus-fraud-group")
public void onCreditoSolicitacao(SolicitacaoCreditoCriadaEvent event) {
    criarOcorrencia(event.getClienteId(), "CREDITO_SOLICITACAO", null);
}

@KafkaListener(topics = Topics.CARTOES_TRANSACAO_AUTORIZADA, groupId = "aureus-fraud-group")
public void onCartoesTransacao(CartaoTransacaoAutorizadaEvent event) {
    criarOcorrencia(event.getCartaoId(), "CARTOES_TRANSACAO", event.getCodigoTransacao());
}
```

Remove `ObjectMapper` and manual JSON parsing. Change `processEvent(String message, String tipo)` to `criarOcorrencia(Long clienteId, String tipo, String transacaoRef)`.

- [ ] **Step 3: Update Fraud KafkaConfig**

Replace old topic names with `Topics.FRAUD_TRANSACAO_BLOQUEADA`, `Topics.FRAUD_OCORRENCIA_CRIADA`, `Topics.FRAUD_SCORE_ALTERADO`.

- [ ] **Step 4: Compile and test**

```bash
mvn -pl backend/aureus-fraud -am test -Dtest="*Test"
```
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add backend/aureus-fraud/
git commit -m "feat(fraud): migrate to typed events, fix consumer topic names"
```

---

### Task 5: Aureus-Notification — JSON → Typed Events + ADR Topics

**Files:**
- Modify: `backend/aureus-notification/src/main/java/com/aureus/platform/notification/service/NotificacaoProducer.java`
- Modify: `backend/aureus-notification/src/main/java/com/aureus/platform/notification/service/NotificacaoConsumer.java`
- Modify: `backend/aureus-notification/src/main/java/com/aureus/platform/notification/config/KafkaConfig.java`

**Depends on:** Task 1

- [ ] **Step 1: Update NotificacaoProducer**

Replace JSON strings with `NotificacaoEnviadaEvent` / `NotificacaoFalhouEvent`. Use `Topics.NOTIFICATION_NOTIFICACAO_ENVIADA`, `Topics.NOTIFICATION_NOTIFICACAO_FALHOU`.

- [ ] **Step 2: Update NotificacaoConsumer**

Change consumers from `String message` to typed events. `onClienteCriado` receives `ClienteCriadoEvent`, `onKycAprovado` receives `KycAprovadoEvent`, etc. Update `processEvent` to accept typed events.

- [ ] **Step 3: Update Notification KafkaConfig**

Replace old topic names with `Topics.NOTIFICATION_NOTIFICACAO_ENVIADA`, `Topics.NOTIFICATION_NOTIFICACAO_FALHOU`.

- [ ] **Step 4: Compile and test**

```bash
mvn -pl backend/aureus-notification -am test -Dtest="*Test"
```
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add backend/aureus-notification/
git commit -m "feat(notification): migrate to typed events + ADR-0001 topics"
```

---

### Task 6: Aureus-Cartões — Migrar Tópicos para ADR-0001

**Files:**
- Modify: `backend/aureus-cartoes/src/main/java/com/aureus/platform/cartoes/config/CartoesKafkaConfig.java`
- Modify: `backend/aureus-cartoes/src/main/java/com/aureus/platform/cartoes/service/EmissaoService.java`
- Modify: `backend/aureus-cartoes/src/main/java/com/aureus/platform/cartoes/service/TransacaoService.java`
- Modify: `backend/aureus-cartoes/src/main/java/com/aureus/platform/cartoes/service/FaturaService.java`
- Remove (local events): `backend/aureus-cartoes/src/main/java/com/aureus/platform/cartoes/event/CartaoEmitidoEvent.java`
- Remove (local events): `backend/aureus-cartoes/src/main/java/com/aureus/platform/cartoes/event/TransacaoAutorizadaEvent.java`
- Remove (local events): `backend/aureus-cartoes/src/main/java/com/aureus/platform/cartoes/event/TransacaoEstornadaEvent.java`
- Remove (local events): `backend/aureus-cartoes/src/main/java/com/aureus/platform/cartoes/event/FaturaFechadaEvent.java`
- Remove (local events): `backend/aureus-cartoes/src/main/java/com/aureus/platform/cartoes/event/FaturaPagaEvent.java`

**Depends on:** Task 1 (shared events for cartões exist)

- [ ] **Step 1: Update CartoesKafkaConfig**

Replace local constants with `Topics.*` imports. Use `Topics.CARTOES_CARTAO_EMITIDO`, `Topics.CARTOES_TRANSACAO_AUTORIZADA`, `Topics.CARTOES_TRANSACAO_ESTORNADA`, `Topics.CARTOES_FATURA_FECHADA`, `Topics.CARTOES_FATURA_PAGA`. Remove local `TOPICO_*` constants.

```java
package com.aureus.platform.cartoes.config;

import com.aureus.platform.shared.event.Topics;
import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class CartoesKafkaConfig {
    @Bean
    public NewTopic cartaoEmitidoTopic() {
        return TopicBuilder.name(Topics.CARTOES_CARTAO_EMITIDO).partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic transacaoAutorizadaTopic() {
        return TopicBuilder.name(Topics.CARTOES_TRANSACAO_AUTORIZADA).partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic transacaoEstornadaTopic() {
        return TopicBuilder.name(Topics.CARTOES_TRANSACAO_ESTORNADA).partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic faturaFechadaTopic() {
        return TopicBuilder.name(Topics.CARTOES_FATURA_FECHADA).partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic faturaPagaTopic() {
        return TopicBuilder.name(Topics.CARTOES_FATURA_PAGA).partitions(3).replicas(1).build();
    }
}
```

- [ ] **Step 2: Update EmissaoService imports**

Change `import com.aureus.platform.cartoes.config.CartoesKafkaConfig.TOPICO_CARTAO_EMITIDO` to `import com.aureus.platform.shared.event.Topics.CARTOES_CARTAO_EMITIDO`. Change `import com.aureus.platform.cartoes.event.CartaoEmitidoEvent` to `import com.aureus.platform.shared.event.CartaoEmitidoEvent`.

Update `kafkaTemplate.send(CartoesKafkaConfig.TOPICO_CARTAO_EMITIDO, ...)` to `kafkaTemplate.send(Topics.CARTOES_CARTAO_EMITIDO, ...)`.

- [ ] **Step 3: Update TransacaoService imports**

Same pattern: replace local event imports with shared ones, replace `CartoesKafkaConfig.TOPICO_TRANSACAO_AUTORIZADA` with `Topics.CARTOES_TRANSACAO_AUTORIZADA`, `TOPICO_TRANSACAO_ESTORNADA` with `CARTOES_TRANSACAO_ESTORNADA`.

- [ ] **Step 4: Update FaturaService imports**

Same pattern: replace `TOPICO_FATURA_FECHADA` → `Topics.CARTOES_FATURA_FECHADA`, `TOPICO_FATURA_PAGA` → `Topics.CARTOES_FATURA_PAGA`.

- [ ] **Step 5: Remove local event files**

Delete the 5 local event records from `backend/aureus-cartoes/src/main/java/com/aureus/platform/cartoes/event/`.

Also remove `package-info.java` from event dir if it only references local events.

- [ ] **Step 6: Compile and test**

```bash
mvn -pl backend/aureus-cartoes -am test -Dtest="*Test"
```
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add backend/aureus-cartoes/
git commit -m "feat(cartoes): migrate topics to ADR-0001, use shared event classes"
```

---

### Task 7: Aureus-Credit — Novo Producer Kafka

**Files:**
- Create: `backend/aureus-credit/src/main/java/com/aureus/platform/credit/config/CreditKafkaConfig.java`
- Modify: `backend/aureus-credit/pom.xml` (add spring-kafka dependency if missing)
- Find/create producer service in credit module

**Depends on:** Task 1 (SolicitacaoCreditoCriadaEvent + Topics constant)

- [ ] **Step 1: Find credit service that needs producer**

Grep for the credit solicitation creation service:

```bash
rg -l "SolicitacaoCredito" backend/aureus-credit/src/main/java/ --type java
```
Expected: find service class that creates credit solicitations

- [ ] **Step 2: Add spring-kafka to pom.xml** (if missing)

Check: `rg "spring-kafka" backend/aureus-credit/pom.xml`. If not present, add:

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

- [ ] **Step 3: Create CreditKafkaConfig**

```java
package com.aureus.platform.credit.config;

import com.aureus.platform.shared.event.Topics;
import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration("creditKafkaConfig")
public class CreditKafkaConfig {
    @Bean
    public NewTopic solicitacaoCreditoCriadaTopic() {
        return TopicBuilder.name(Topics.CREDIT_SOLICITACAO_CRIADA)
            .partitions(3).replicas(1).build();
    }
}
```

- [ ] **Step 4: Add producer to credit service**

In the credit solicitation service, inject `KafkaTemplate<String, Object>` and after creating a solicitation:

```java
import com.aureus.platform.shared.event.SolicitacaoCreditoCriadaEvent;
import com.aureus.platform.shared.event.Topics;

// In method that creates solicitation:
SolicitacaoCreditoCriadaEvent event = SolicitacaoCreditoCriadaEvent.criada(
    solicitacao.getId(), solicitacao.getClienteId(), solicitacao.getValor(), solicitacao.getTipoCredito());
kafkaTemplate.send(Topics.CREDIT_SOLICITACAO_CRIADA, String.valueOf(solicitacao.getClienteId()), event);
```

- [ ] **Step 5: Compile and test**

```bash
mvn -pl backend/aureus-credit -am compile -DskipTests
```
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add backend/aureus-credit/
git commit -m "feat(credit): add Kafka producer for credit.solicitacao.criada.v1"
```

---

### Task 8: Tax, Billing, Bacen — Producers Faltantes

**Files:**
- Modify: `backend/aureus-tax/` — adicionar producer no serviço de cálculo de imposto
- Modify: `backend/aureus-billing/` — adicionar producers para fatura emitida/paga
- Modify: `backend/aureus-bacen/` — adicionar producers para relatório gerado/enviado

- [ ] **Step 1: Add spring-kafka dependency** to `aureus-tax/pom.xml`, `aureus-billing/pom.xml`, `aureus-bacen/pom.xml` (check each first with `rg "spring-kafka"`)

- [ ] **Step 2: Create TaxKafkaConfig** with topic bean for `Topics.IMPOSTO_CALCULADO`

- [ ] **Step 3: Add producer to tax calculation service** — inject `KafkaTemplate<String, Object>`, create `ImpostoEvent.impostoCalculado(...)` and send to `Topics.IMPOSTO_CALCULADO`

- [ ] **Step 4: Create BillingKafkaConfig** with topic beans for `Topics.FATURA_EMITIDA`, `Topics.FATURA_PAGA`

- [ ] **Step 5: Add producers to billing services** — after emitir fatura and after pagar fatura

- [ ] **Step 6: Create BacenKafkaConfig** with topic beans for `Topics.RELATORIO_GERADO`, `Topics.RELATORIO_ENVIADO`

- [ ] **Step 7: Add producers to bacen services** — after gerar relatório and after enviar relatório

- [ ] **Step 8: Compile and test all**

```bash
mvn -pl backend/aureus-tax,backend/aureus-billing,backend/aureus-bacen -am compile -DskipTests
```
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add backend/aureus-tax/ backend/aureus-billing/ backend/aureus-bacen/
git commit -m "feat(tax,billing,bacen): add missing Kafka producers"
```

---

### Task 9: Product Modules — Migrar Tópicos para ADR-0001

**Files:**
- Modify: `backend/aureus-cambio/src/main/java/com/aureus/platform/cambio/config/CambioKafkaConfig.java`
- Modify: `backend/aureus-consignado/src/main/java/com/aureus/platform/consignado/config/ConsignadoKafkaConfig.java`
- Modify: `backend/aureus-financiamento/src/main/java/com/aureus/platform/financiamento/config/FinanciamentoKafkaConfig.java`
- Modify: `backend/aureus-investimento/src/main/java/com/aureus/platform/investimento/config/InvestimentoKafkaConfig.java`
- Modify: `backend/aureus-poupanca/src/main/java/com/aureus/platform/poupanca/config/PoupancaKafkaConfig.java`
- Modify: `backend/aureus-salario/src/main/java/com/aureus/platform/salario/config/SalarioKafkaConfig.java`
- Modify: `backend/aureus-seguros/src/main/java/com/aureus/platform/seguros/config/SegurosKafkaConfig.java`

**Depends on:** Task 1 (Topics constants exist)

- [ ] **Step 1: Update CambioKafkaConfig**

Replace local `TOPICO_*` string constants with `Topics.CAMBIO_COTACAO_ATUALIZADA`, `Topics.CAMBIO_CONTRATO_FECHADO`, `Topics.CAMBIO_CONTRATO_LIQUIDADO`, `Topics.CAMBIO_REMESSA_PROCESSADA`.

Update all services that reference `CambioKafkaConfig.TOPICO_*` to use `Topics.*` instead. Grep:

```bash
rg "TOPICO_" backend/aureus-cambio/src/main/java/ --type java
```

If any service uses the local constants, update them.

- [ ] **Step 2: Repeat for consignado**

Replace with `Topics.CONSIGNADO_CONTRATO_ASSINADO`, `Topics.CONSIGNADO_PARCELA_DEBITADA`, `Topics.CONSIGNADO_MARGEM_ATUALIZADA`, `Topics.CONSIGNADO_CONTRATO_LIQUIDADO`.

- [ ] **Step 3: Repeat for financiamento**

Replace with `Topics.FINANCIAMENTO_SIMULACAO_REALIZADA`, `Topics.FINANCIAMENTO_CONTRATO_ASSINADO`, `Topics.FINANCIAMENTO_PARCELA_PAGA`, `Topics.FINANCIAMENTO_CONTRATO_LIQUIDADO`, `Topics.FINANCIAMENTO_GARANTIA_REGISTRADA`.

- [ ] **Step 4: Repeat for investimento**

Replace with `Topics.INVESTIMENTO_CONTA_CRIADA`, `Topics.INVESTIMENTO_ORDEM_EXECUTADA`, `Topics.INVESTIMENTO_RESGATE_PROCESSADO`.

- [ ] **Step 5: Repeat for poupanca**

Replace with `Topics.POUPANCA_CONTA_CRIADA`, `Topics.POUPANCA_DEPOSITO_REALIZADO`, `Topics.POUPANCA_SAQUE_REALIZADO`, `Topics.POUPANCA_RENDIMENTO_CREDITADO`.

- [ ] **Step 6: Repeat for salario**

Replace with `Topics.SALARIO_CONTA_CRIADA`, `Topics.SALARIO_CREDITADO`, `Topics.SALARIO_PORTABILIDADE_SOLICITADA`.

- [ ] **Step 7: Repeat for seguros**

Replace with `Topics.SEGUROS_APOLICE_EMITIDA`, `Topics.SEGUROS_PREMIO_PAGO`, `Topics.SEGUROS_SINISTRO_ABERTO`, `Topics.SEGUROS_SINISTRO_LIQUIDADO`.

- [ ] **Step 8: Compile all product modules**

```bash
mvn -pl backend/aureus-cambio,backend/aureus-consignado,backend/aureus-financiamento,backend/aureus-investimento,backend/aureus-poupanca,backend/aureus-salario,backend/aureus-seguros -am compile -DskipTests
```
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add backend/aureus-cambio/ backend/aureus-consignado/ backend/aureus-financiamento/ backend/aureus-investimento/ backend/aureus-poupanca/ backend/aureus-salario/ backend/aureus-seguros/
git commit -m "feat: migrate all product module topics to ADR-0001 constants"
```

---

### Task 10: Notification — Expandir Consumers

**Files:**
- Modify: `backend/aureus-notification/src/main/java/com/aureus/platform/notification/service/NotificacaoConsumer.java`
- Modify: `backend/aureus-notification/src/main/java/com/aureus/platform/notification/config/KafkaConfig.java`

**Depends on:** Task 1, Task 5, Task 6, Task 9

- [ ] **Step 1: Add topic beans to Notification KafkaConfig**

```java
@Bean
public NewTopic transacaoRealizadaTopic() {
    return TopicBuilder.name(Topics.CORE_TRANSACAO_REALIZADA).partitions(3).replicas(1).build();
}
// ... same for CARTOES_TRANSACAO_AUTORIZADA, CARTOES_FATURA_FECHADA, CARTOES_FATURA_PAGA,
// SETTLEMENT_LIQUIDEZ_PROCESSADA, SETTLEMENT_LIQUIDEZ_REJEITADA,
// CONSIGNADO_CONTRATO_ASSINADO, FINANCIAMENTO_CONTRATO_ASSINADO,
// INVESTIMENTO_ORDEM_EXECUTADA, SEGUROS_APOLICE_EMITIDA
```

- [ ] **Step 2: Add consumer methods to NotificacaoConsumer**

```java
@KafkaListener(topics = Topics.CORE_TRANSACAO_REALIZADA, groupId = "aureus-notification-group")
public void onTransacaoRealizada(TransacaoEvent event) {
    Map<String, String> vars = new HashMap<>();
    vars.put("clienteId", event.getClienteId());
    vars.put("valor", event.getValor().toString());
    vars.put("descricao", event.getDescricao());
    notificacaoService.enviar(Long.valueOf(event.getClienteId()), "transacao_realizada",
        "cliente-" + event.getClienteId(), vars);
}

@KafkaListener(topics = Topics.CARTOES_TRANSACAO_AUTORIZADA, groupId = "aureus-notification-group")
public void onCartaoTransacaoAutorizada(CartaoTransacaoAutorizadaEvent event) {
    Map<String, String> vars = new HashMap<>();
    vars.put("cartaoId", String.valueOf(event.getCartaoId()));
    vars.put("valor", event.getValor().toString());
    vars.put("estabelecimento", event.getEstabelecimento());
    notificacaoService.enviar(event.getCartaoId(), "cartao_compra_aprovada",
        "cliente-" + event.getCartaoId(), vars);
}

// ... same pattern for CARTOES_FATURA_FECHADA, CARTOES_FATURA_PAGA,
// SETTLEMENT_LIQUIDEZ_PROCESSADA, SETTLEMENT_LIQUIDEZ_REJEITADA,
// CONSIGNADO_CONTRATO_ASSINADO, FINANCIAMENTO_CONTRATO_ASSINADO,
// INVESTIMENTO_ORDEM_EXECUTADA, SEGUROS_APOLICE_EMITIDA
```

- [ ] **Step 3: Compile and test**

```bash
mvn -pl backend/aureus-notification -am test -Dtest="*Test"
```
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add backend/aureus-notification/
git commit -m "feat(notification): expand consumers to 14 topics across core, cartoes, settlement, consignado, financiamento, investimento, seguros"
```

---

### Task 11: Aureus-Audit — Consumer Universal

**Files:**
- Create: `backend/aureus-audit/src/main/java/com/aureus/platform/audit/config/AuditKafkaConfig.java`
- Create: `backend/aureus-audit/src/main/java/com/aureus/platform/audit/service/AuditEventConsumer.java`
- Modify: `backend/aureus-audit/pom.xml` (add spring-kafka if missing)
- Modify: `backend/aureus-audit/src/main/java/com/aureus/platform/audit/service/LogAuditoriaService.java` (add method to save from event)

**Depends on:** Task 1

- [ ] **Step 1: Add spring-kafka to aureus-audit pom.xml** (if missing)

- [ ] **Step 2: Create AuditKafkaConfig**

```java
package com.aureus.platform.audit.config;

import com.aureus.platform.shared.event.Topics;
import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration("auditKafkaConfig")
public class AuditKafkaConfig {
    // Audit reads from all topics, no NewTopic beans needed (they're declared elsewhere)
}
```

- [ ] **Step 3: Create AuditEventConsumer**

```java
package com.aureus.platform.audit.service;

import com.aureus.platform.audit.entity.LogAuditoria;
import com.aureus.platform.audit.repository.LogAuditoriaRepository;
import com.aureus.platform.shared.event.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;
import java.time.LocalDateTime;

@Service
public class AuditEventConsumer {
    private static final Logger log = LoggerFactory.getLogger(AuditEventConsumer.class);
    private final LogAuditoriaRepository repository;

    public AuditEventConsumer(LogAuditoriaRepository repository) {
        this.repository = repository;
    }

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
        Topics.TAX_IMPOSTO_CALCULADO,
        Topics.BILLING_FATURA_EMITIDA,
        Topics.BILLING_FATURA_PAGA,
        Topics.BACEN_RELATORIO_GERADO,
        Topics.BACEN_RELATORIO_ENVIADO,
        Topics.CREDIT_SOLICITACAO_CRIADA,
        Topics.CARTOES_CARTAO_EMITIDO,
        Topics.CARTOES_TRANSACAO_AUTORIZADA,
        Topics.CARTOES_TRANSACAO_ESTORNADA,
        Topics.CARTOES_FATURA_FECHADA,
        Topics.CARTOES_FATURA_PAGA,
        Topics.CONSIGNADO_CONTRATO_ASSINADO,
        Topics.FINANCIAMENTO_CONTRATO_ASSINADO,
        Topics.INVESTIMENTO_ORDEM_EXECUTADA,
        Topics.SEGUROS_APOLICE_EMITIDA,
        Topics.NOTIFICATION_NOTIFICACAO_ENVIADA,
        Topics.NOTIFICATION_NOTIFICACAO_FALHOU
    }, groupId = "aureus-audit-group")
    public void onEvent(BaseEvent event) {
        LogAuditoria logAud = new LogAuditoria();
        logAud.setTipoEvento(event.getEventType());
        logAud.setOrigem(event.getSource());
        logAud.setIdCorrelacao(event.getCorrelationId());
        logAud.setIdEvento(event.getEventId());
        logAud.setTimestamp(LocalDateTime.now());
        logAud.setPayload(event.toString());
        repository.save(logAud);
        log.debug("Audit registered: type={}, source={}", event.getEventType(), event.getSource());
    }
}
```

- [ ] **Step 4: Check LogAuditoria entity** — ensure it has fields: tipoEvento, origem, idCorrelacao, idEvento, timestamp, payload

Read: `backend/aureus-audit/src/main/java/com/aureus/platform/audit/entity/LogAuditoria.java`

If missing fields, add them.

- [ ] **Step 5: Compile and test**

```bash
mvn -pl backend/aureus-audit -am compile -DskipTests
```
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add backend/aureus-audit/
git commit -m "feat(audit): add universal Kafka consumer for all platform events"
```

---

### Task 12: Build Final + Verify

- [ ] **Step 1: Full compile check**

```bash
mvn clean compile -DskipTests -q
```
Expected: BUILD SUCCESS (no output with -q)

- [ ] **Step 2: Run tests for all changed modules**

```bash
mvn test -pl backend/aureus-shared,backend/aureus-customer,backend/aureus-kyc,backend/aureus-fraud,backend/aureus-notification,backend/aureus-cartoes,backend/aureus-credit,backend/aureus-audit -am
```
Expected: BUILD SUCCESS

- [ ] **Step 3: Review git status**

```bash
git status
git log --oneline -10
```

- [ ] **Step 4: Final commit if any pending changes**

```bash
git commit -am "chore: final adjustments after event-driven integration"
```
