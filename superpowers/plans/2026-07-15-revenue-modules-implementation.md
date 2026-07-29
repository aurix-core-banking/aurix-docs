# Revenue Modules (Fase 2) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement 3 revenue modules (Acquirer, Collections, Guarantee) — Guarantee extracted from finanziamento, Acquirer and Collections built from scratch.

**Architecture:** 3 independent Spring Boot microservices (ports 8127-8129), same pattern as Foundation modules. Guarantee is extracted from `aurix-financiamento` with a Feign client for backwards compatibility.

**Tech Stack:** Java 25, Spring Boot 4.1.0, Maven, PostgreSQL, Kafka, Feign, Testcontainers

## Global Constraints

- All Java code: Project Lombok (`@Slf4j`), constructor injection, `TenantContext.getTenantId()`
- All Java exceptions: `IllegalArgumentException` for business errors, `IllegalStateException` for state errors
- Kafka events extend `BaseEvent` in `com.aurix.platform.shared.event`
- Topic constants in `Topics.java` following ADR-0001: `<dominio>.<entidade>.<evento>.v1`
- New modules added to `backend/pom.xml` `<modules>` list
- Each module gets: Dockerfile, SecurityConfig, application.yml, docker-compose entry, traefik route, e2e config
- `@Configuration("<nome>KafkaConfig")` for Kafka configs to avoid bean name conflicts
- Ports: 8127 (acquirer), 8128 (collections), 8129 (guarantee)
- Context paths: `/api/acquirer`, `/api/collections`, `/api/guarantee`
- Package naming: `com.aurix.platform.acquirer`, `.collections`, `.guarantee`

---

## File Map

### Guarantee (extracted)
- Create: `backend/aurix-guarantee/pom.xml`
- Create: `backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/AurixGuaranteeApplication.java`
- Create: `backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/config/SecurityConfig.java`
- Create: `backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/config/GuaranteeKafkaConfig.java`
- Move (from finanziamento): `Garantia.java`, `GarantiaRepository.java`, `GarantiaService.java`, `GarantiaRequest.java`, `GarantiaResponse.java`, `GarantiaController.java`, `AtualizacaoGarantiasJob.java`, `GarantiaControllerTest.java`
- Create: `backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/client/GarantiaClient.java` (Feign)
- Create: `backend/aurix-guarantee/src/main/resources/application.yml`
- Create: `backend/aurix-guarantee/Dockerfile`
- Modify: `backend/aurix-financiamento/pom.xml` (add guarantee dependency)
- Modify: `backend/aurix-financiamento/src/main/java/.../service/ContratoFinanciamentoService.java` (use GarantiaClient)

### Acquirer (new)
- Create: `backend/aurix-acquirer/pom.xml`
- Create: `backend/aurix-acquirer/src/main/java/com/aurix/platform/acquirer/AurixAcquirerApplication.java`
- Create: entity classes (Estabelecimento, Terminal, TransacaoCaptura, Liquidacao, TaxaAcquirer)
- Create: repository interfaces
- Create: services (TransacaoService, LiquidacaoService, EstabelecimentoService)
- Create: controllers (TransacaoController, EstabelecimentoController, LiquidacaoController)
- Create: config (SecurityConfig, AcquirerKafkaConfig)
- Create: `application.yml`, `Dockerfile`

### Collections (new)
- Create: `backend/aurix-collections/pom.xml`
- Create: `backend/aurix-collections/src/main/java/com/aurix/platform/collections/AurixCollectionsApplication.java`
- Create: entity classes (Cobranca, CarnetParcela, Acordo, Negativacao, RegistroCobranca)
- Create: repository interfaces
- Create: services (BoletoService, CarnetService, AcordoService, NegativacaoService)
- Create: controllers (BoletoController, CarnetController, AcordoController, NegativacaoController)
- Create: config (SecurityConfig, CollectionsKafkaConfig)
- Create: `application.yml`, `Dockerfile`

### Shared events (for all 3)
- Modify: `backend/aurix-shared/src/main/java/.../event/Topics.java`
- Create: 10 event classes in `...shared.event`

### Infra
- Modify: `infrastructure/docker-compose.yml` (add 3 services)
- Modify: `infrastructure/traefik/dynamic.yml` (add 3 routers/services)
- Modify: `aurix-tests/e2e/config.py` (add 3 health endpoints)
- Modify: `backend/pom.xml` (add 3 modules)

---

### Task 1: Shared — Add events and topic constants for all 3 modules

**Files:**
- Modify: `backend/aurix-shared/src/main/java/com/aurix/platform/shared/event/Topics.java`
- Create: `TransacaoAutorizadaEvent.java`
- Create: `TransacaoCapturadaEvent.java`
- Create: `TransacaoLiquidadaEvent.java`
- Create: `TransacaoEstornadaEvent.java`
- Create: `BoletoEmitidoEvent.java`
- Create: `CobrancaPagaEvent.java`
- Create: `CobrancaNegativadaEvent.java`
- Create: `CobrancaCanceladaEvent.java`
- Create: `GarantiaRegistradaEvent.java`
- Create: `GarantiaLiberadaEvent.java`

**Interfaces:**
- Consumes: `BaseEvent` (existing), existing `Topics.java` pattern
- Produces: 10 event classes + 10 topic constants used by all 3 modules

- [ ] **Step 1: Add topic constants to Topics.java**

Add after existing constants:

```java
// ===== acquirer =====
public static final String ACQUIRER_TRANSACAO_AUTORIZADA = "acquirer.transacao.autorizada.v1";
public static final String ACQUIRER_TRANSACAO_CAPTURADA = "acquirer.transacao.capturada.v1";
public static final String ACQUIRER_TRANSACAO_LIQUIDADA = "acquirer.transacao.liquidada.v1";
public static final String ACQUIRER_TRANSACAO_ESTORNADA = "acquirer.transacao.estornada.v1";

// ===== collections =====
public static final String COLLECTIONS_BOLETO_EMITIDO = "collections.boleto.emitido.v1";
public static final String COLLECTIONS_COBRANCA_PAGA = "collections.cobranca.paga.v1";
public static final String COLLECTIONS_COBRANCA_NEGATIVADA = "collections.cobranca.negativada.v1";
public static final String COLLECTIONS_COBRANCA_CANCELADA = "collections.cobranca.cancelada.v1";

// ===== guarantee =====
public static final String GUARANTEE_GARANTIA_REGISTRADA = "guarantee.garantia.registrada.v1";
public static final String GUARANTEE_GARANTIA_LIBERADA = "guarantee.garantia.liberada.v1";
```

- [ ] **Step 2: Create acquirer event classes**

Create `TransacaoAutorizadaEvent.java` in `com.aurix.platform.shared.event`:

```java
package com.aurix.platform.shared.event;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public class TransacaoAutorizadaEvent extends BaseEvent {
    private Long transacaoId;
    private Long terminalId;
    private BigDecimal valor;
    private String bandeira;
    private String codigoAutorizacao;
    private String nsu;

    public static TransacaoAutorizadaEvent autorizada(Long transacaoId, Long terminalId, BigDecimal valor, String bandeira, String codigoAutorizacao, String nsu) {
        TransacaoAutorizadaEvent e = new TransacaoAutorizadaEvent();
        e.setEventId(java.util.UUID.randomUUID().toString());
        e.setEventType("TRANSACAO_AUTORIZADA");
        e.setSource("aurix-acquirer");
        e.setTimestamp(LocalDateTime.now());
        e.setCorrelationId(java.util.UUID.randomUUID().toString());
        e.transacaoId = transacaoId;
        e.terminalId = terminalId;
        e.valor = valor;
        e.bandeira = bandeira;
        e.codigoAutorizacao = codigoAutorizacao;
        e.nsu = nsu;
        return e;
    }

    public TransacaoAutorizadaEvent() {}

    public Long getTransacaoId() { return transacaoId; }
    public void setTransacaoId(Long v) { this.transacaoId = v; }
    public Long getTerminalId() { return terminalId; }
    public void setTerminalId(Long v) { this.terminalId = v; }
    public BigDecimal getValor() { return valor; }
    public void setValor(BigDecimal v) { this.valor = v; }
    public String getBandeira() { return bandeira; }
    public void setBandeira(String v) { this.bandeira = v; }
    public String getCodigoAutorizacao() { return codigoAutorizacao; }
    public void setCodigoAutorizacao(String v) { this.codigoAutorizacao = v; }
    public String getNsu() { return nsu; }
    public void setNsu(String v) { this.nsu = v; }
}
```

Then create `TransacaoCapturadaEvent.java`, `TransacaoLiquidadaEvent.java`, `TransacaoEstornadaEvent.java` with the same pattern (factory method + empty constructor + getters/setters).

TransacaoCapturadaEvent: transacaoId, valor, parcelas, bandeira (source "aurix-acquirer", eventType "TRANSACAO_CAPTURADA")
TransacaoLiquidadaEvent: liquidacaoId, transacaoId, valorLiquido, valorRepasse (source "aurix-acquirer", eventType "TRANSACAO_LIQUIDADA")
TransacaoEstornadaEvent: transacaoId, valor, motivo (source "aurix-acquirer", eventType "TRANSACAO_ESTORNADA")

- [ ] **Step 3: Create collections event classes**

Create `BoletoEmitidoEvent.java`:
```java
package com.aurix.platform.shared.event;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

public class BoletoEmitidoEvent extends BaseEvent {
    private Long cobrancaId;
    private String nossoNumero;
    private BigDecimal valor;
    private LocalDate dataVencimento;
    private Long clienteId;

    public static BoletoEmitidoEvent emitido(Long cobrancaId, String nossoNumero, BigDecimal valor, LocalDate dataVencimento, Long clienteId) {
        BoletoEmitidoEvent e = new BoletoEmitidoEvent();
        e.setEventId(java.util.UUID.randomUUID().toString());
        e.setEventType("BOLETO_EMITIDO");
        e.setSource("aurix-collections");
        e.setTimestamp(LocalDateTime.now());
        e.setCorrelationId(java.util.UUID.randomUUID().toString());
        e.cobrancaId = cobrancaId;
        e.nossoNumero = nossoNumero;
        e.valor = valor;
        e.dataVencimento = dataVencimento;
        e.clienteId = clienteId;
        return e;
    }

    public BoletoEmitidoEvent() {}

    public Long getCobrancaId() { return cobrancaId; }
    public void setCobrancaId(Long v) { this.cobrancaId = v; }
    public String getNossoNumero() { return nossoNumero; }
    public void setNossoNumero(String v) { this.nossoNumero = v; }
    public BigDecimal getValor() { return valor; }
    public void setValor(BigDecimal v) { this.valor = v; }
    public LocalDate getDataVencimento() { return dataVencimento; }
    public void setDataVencimento(LocalDate v) { this.dataVencimento = v; }
    public Long getClienteId() { return clienteId; }
    public void setClienteId(Long v) { this.clienteId = v; }
}
```

Create `CobrancaPagaEvent.java`, `CobrancaNegativadaEvent.java`, `CobrancaCanceladaEvent.java` with the same pattern.

CobrancaPagaEvent: cobrancaId, valorPago (BigDecimal), dataPagamento (LocalDate), source "aurix-collections", eventType "COBRANCA_PAGA"
CobrancaNegativadaEvent: cobrancaId, orgao (String), dataEnvio (LocalDate), source "aurix-collections", eventType "COBRANCA_NEGATIVADA"
CobrancaCanceladaEvent: cobrancaId, motivo (String), source "aurix-collections", eventType "COBRANCA_CANCELADA"

- [ ] **Step 4: Create guarantee event classes**

Create `GarantiaRegistradaEvent.java`:
```java
package com.aurix.platform.shared.event;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public class GarantiaRegistradaEvent extends BaseEvent {
    private Long garantiaId;
    private Long contratoId;
    private Long clienteId;
    private String tipo;
    private BigDecimal valor;

    public static GarantiaRegistradaEvent registrada(Long garantiaId, Long contratoId, Long clienteId, String tipo, BigDecimal valor) {
        GarantiaRegistradaEvent e = new GarantiaRegistradaEvent();
        e.setEventId(java.util.UUID.randomUUID().toString());
        e.setEventType("GARANTIA_REGISTRADA");
        e.setSource("aurix-guarantee");
        e.setTimestamp(LocalDateTime.now());
        e.setCorrelationId(java.util.UUID.randomUUID().toString());
        e.garantiaId = garantiaId;
        e.contratoId = contratoId;
        e.clienteId = clienteId;
        e.tipo = tipo;
        e.valor = valor;
        return e;
    }

    public GarantiaRegistradaEvent() {}

    public Long getGarantiaId() { return garantiaId; }
    public void setGarantiaId(Long v) { this.garantiaId = v; }
    public Long getContratoId() { return contratoId; }
    public void setContratoId(Long v) { this.contratoId = v; }
    public Long getClienteId() { return clienteId; }
    public void setClienteId(Long v) { this.clienteId = v; }
    public String getTipo() { return tipo; }
    public void setTipo(String v) { this.tipo = v; }
    public BigDecimal getValor() { return valor; }
    public void setValor(BigDecimal v) { this.valor = v; }
}
```

Create `GarantiaLiberadaEvent.java`: garantiaId, contratoId, dataBaixa (LocalDate), source "aurix-guarantee", eventType "GARANTIA_LIBERADA"

- [ ] **Step 5: Compile shared module**

```bash
mvn -pl backend/aurix-shared -am compile -DskipTests
```

Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add backend/aurix-shared/src/main/java/com/aurix/platform/shared/event/TransacaoAutorizadaEvent.java backend/aurix-shared/src/main/java/com/aurix/platform/shared/event/TransacaoCapturadaEvent.java backend/aurix-shared/src/main/java/com/aurix/platform/shared/event/TransacaoLiquidadaEvent.java backend/aurix-shared/src/main/java/com/aurix/platform/shared/event/TransacaoEstornadaEvent.java backend/aurix-shared/src/main/java/com/aurix/platform/shared/event/BoletoEmitidoEvent.java backend/aurix-shared/src/main/java/com/aurix/platform/shared/event/CobrancaPagaEvent.java backend/aurix-shared/src/main/java/com/aurix/platform/shared/event/CobrancaNegativadaEvent.java backend/aurix-shared/src/main/java/com/aurix/platform/shared/event/CobrancaCanceladaEvent.java backend/aurix-shared/src/main/java/com/aurix/platform/shared/event/GarantiaRegistradaEvent.java backend/aurix-shared/src/main/java/com/aurix/platform/shared/event/GarantiaLiberadaEvent.java
git commit -m "feat(shared): add revenue module event classes and ADR-0001 topic constants"
```

---

### Task 2: Guarantee — Create module skeleton

**Files:**
- Create: `backend/aurix-guarantee/pom.xml`
- Create: `backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/AurixGuaranteeApplication.java`
- Create: `backend/aurix-guarantee/src/main/resources/application.yml`
- Create: `backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/config/SecurityConfig.java`
- Create: `backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/config/GuaranteeKafkaConfig.java`
- Create: `backend/aurix-guarantee/Dockerfile`
- Modify: `backend/pom.xml`

**Interfaces:**
- Consumes: shared module, Spring Boot starters
- Produces: compilable module skeleton

- [ ] **Step 1: Create pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.aurix.platform</groupId>
        <artifactId>aurix-platform</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>aurix-guarantee</artifactId>
    <packaging>jar</packaging>
    <name>AURIX Guarantee</name>
    <description>Módulo de Garantias - Alienação Fiduciária, Penhor, Hipoteca</description>

    <dependencies>
        <dependency>
            <groupId>com.aurix.platform</groupId>
            <artifactId>aurix-shared</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

- [ ] **Step 2: Create application main class**

```java
package com.aurix.platform.guarantee;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableFeignClients
public class AurixGuaranteeApplication {
    public static void main(String[] args) {
        SpringApplication.run(AurixGuaranteeApplication.class, args);
    }
}
```

- [ ] **Step 3: Create application.yml**

```yaml
server:
  port: 8129
  servlet:
    context-path: /api/guarantee

spring:
  application:
    name: aurix-guarantee
  datasource:
    url: jdbc:postgresql://localhost:5432/aurix
    username: aurix
    password: aurix
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      group-id: aurix-guarantee-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.aurix.platform.shared.event"

logging:
  level:
    com.aurix.platform: DEBUG
```

- [ ] **Step 4: Create SecurityConfig**

```java
package com.aurix.platform.guarantee.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
            .csrf(csrf -> csrf.disable());
        return http.build();
    }
}
```

- [ ] **Step 5: Create GuaranteeKafkaConfig**

```java
package com.aurix.platform.guarantee.config;

import com.aurix.platform.shared.event.Topics;
import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration("guaranteeKafkaConfig")
public class GuaranteeKafkaConfig {
    @Bean
    public NewTopic garantiaRegistradaTopic() {
        return TopicBuilder.name(Topics.GUARANTEE_GARANTIA_REGISTRADA)
            .partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic garantiaLiberadaTopic() {
        return TopicBuilder.name(Topics.GUARANTEE_GARANTIA_LIBERADA)
            .partitions(3).replicas(1).build();
    }
}
```

- [ ] **Step 6: Create Dockerfile**

```dockerfile
FROM eclipse-temurin:25-jdk-jammy AS build
WORKDIR /app
COPY mvnw pom.xml ./
RUN chmod +x mvnw
COPY . .
RUN ./mvnw -pl aurix-guarantee -am package -DskipTests -q

FROM eclipse-temurin:25-jre-jammy
WORKDIR /app
COPY --from=build /app/backend/aurix-guarantee/target/*.jar app.jar
EXPOSE 8129
ENTRYPOINT ["java", "-jar", "app.jar"]
```

- [ ] **Step 7: Add module to backend/pom.xml**

Read `backend/pom.xml`, find the `<modules>` section, and add:
```xml
<module>aurix-guarantee</module>
```

- [ ] **Step 8: Compile to verify**

```bash
mvn -pl backend/aurix-guarantee -am compile -DskipTests
```

Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add backend/aurix-guarantee/ backend/pom.xml
git commit -m "feat(guarantee): create module skeleton with pom.xml, application, security, kafka config"
```

---

### Task 3: Guarantee — Extract entities, repositories, DTOs from finanziamento

**Files:**
- Move: `Garantia.java` from `...financiamento.entity` → `...guarantee.entity`
- Move: `GarantiaRepository.java` from `...financiamento.repository` → `...guarantee.repository`
- Move: `GarantiaRequest.java` from `...financiamento.dto.request` → `...guarantee.dto`
- Move: `GarantiaResponse.java` from `...financiamento.dto.response` → `...guarantee.dto`
- Create: `Bem.java`, `Avaliacao.java`, `RegistroGarantia.java` in `...guarantee.entity`
- Create: `BemRepository.java`, `AvaliacaoRepository.java`, `RegistroGarantiaRepository.java` in `...guarantee.repository`

**Interfaces:**
- Consumes: existing Garantia entity/Repository/DTOs from finanziamento
- Produces: full entity + repository layer for Guarantee module

- [ ] **Step 1: Read existing Garantia.java from finanziamento**

Read `backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/entity/Garantia.java` to understand its structure.

- [ ] **Step 2: Copy and adapt Garantia.java to guarantee module**

Create `backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/entity/Garantia.java`:

Use the same fields as the finanziamento version but update the package and add any missing fields from the spec (clienteId, contratoId as generic Long, tenantId).

The updated Garantia entity should have:
- id (Long, PK auto)
- contratoId (Long — generic, can reference any contract)
- clienteId (Long)
- bemId (Long, FK → bem)
- tipo (String — ALIENACAO_FIDUCIARIA/PENHOR/HIPOTECA/FIANCA)
- valor (BigDecimal)
- dataRegistro (LocalDate)
- dataVencimento (LocalDate)
- status (String — ATIVA/LIBERADA/VENCIDA)
- dataBaixa (LocalDate)
- tenantId (String)

Use `@Entity @Table(name = "garantias", schema = "aurix")` and appropriate JPA annotations.

- [ ] **Step 3: Copy GarantiaRepository**

Create `backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/repository/GarantiaRepository.java`:

```java
package com.aurix.platform.guarantee.repository;

import com.aurix.platform.guarantee.entity.Garantia;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface GarantiaRepository extends JpaRepository<Garantia, Long> {
    List<Garantia> findByContratoId(Long contratoId);
    List<Garantia> findByClienteId(Long clienteId);
    List<Garantia> findByStatus(String status);
}
```

- [ ] **Step 4: Copy GarantiaRequest and GarantiaResponse DTOs**

Create `backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/dto/GarantiaRequest.java` and `GarantiaResponse.java`:

GarantiaRequest: contratoId, clienteId, bemId, tipo, valor, dataVencimento
GarantiaResponse: id, tipo, valor, dataRegistro, dataBaixa, status, orgaoRegistro (same as existing from finanziamento)

- [ ] **Step 5: Create Bem entity and repository**

```java
package com.aurix.platform.guarantee.entity;

import jakarta.persistence.*;
import java.math.BigDecimal;

@Entity
@Table(name = "bens", schema = "aurix")
public class Bem {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String tipo; // IMOVEL/VEICULO/EQUIPAMENTO/TITULO
    private String descricao;
    private BigDecimal valorAvaliacao;
    private String registroCartorio;
    private String chassi;
    private String placa;
    private String tenantId;

    public Bem() {}
    // getters and setters for all fields
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getTipo() { return tipo; }
    public void setTipo(String tipo) { this.tipo = tipo; }
    public String getDescricao() { return descricao; }
    public void setDescricao(String descricao) { this.descricao = descricao; }
    public BigDecimal getValorAvaliacao() { return valorAvaliacao; }
    public void setValorAvaliacao(BigDecimal v) { this.valorAvaliacao = v; }
    public String getRegistroCartorio() { return registroCartorio; }
    public void setRegistroCartorio(String v) { this.registroCartorio = v; }
    public String getChassi() { return chassi; }
    public void setChassi(String chassi) { this.chassi = chassi; }
    public String getPlaca() { return placa; }
    public void setPlaca(String placa) { this.placa = placa; }
    public String getTenantId() { return tenantId; }
    public void setTenantId(String v) { this.tenantId = v; }
}
```

Create `BemRepository.java`:
```java
package com.aurix.platform.guarantee.repository;

import com.aurix.platform.guarantee.entity.Bem;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface BemRepository extends JpaRepository<Bem, Long> {
    List<Bem> findByTipo(String tipo);
}
```

- [ ] **Step 6: Create Avaliacao entity and repository**

```java
package com.aurix.platform.guarantee.entity;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDate;

@Entity
@Table(name = "avaliacoes", schema = "aurix")
public class Avaliacao {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long bemId;
    private LocalDate data;
    private BigDecimal valor;
    private String metodo;
    private Long avaliadorId;
    private LocalDate validadeAte;

    public Avaliacao() {}
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Long getBemId() { return bemId; }
    public void setBemId(Long v) { this.bemId = v; }
    public LocalDate getData() { return data; }
    public void setData(LocalDate v) { this.data = v; }
    public BigDecimal getValor() { return valor; }
    public void setValor(BigDecimal v) { this.valor = v; }
    public String getMetodo() { return metodo; }
    public void setMetodo(String v) { this.metodo = v; }
    public Long getAvaliadorId() { return avaliadorId; }
    public void setAvaliadorId(Long v) { this.avaliadorId = v; }
    public LocalDate getValidadeAte() { return validadeAte; }
    public void setValidadeAte(LocalDate v) { this.validadeAte = v; }
}
```

Create `AvaliacaoRepository.java`:
```java
package com.aurix.platform.guarantee.repository;

import com.aurix.platform.guarantee.entity.Avaliacao;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface AvaliacaoRepository extends JpaRepository<Avaliacao, Long> {
    List<Avaliacao> findByBemIdOrderByDataDesc(Long bemId);
}
```

- [ ] **Step 7: Create RegistroGarantia entity and repository**

```java
package com.aurix.platform.guarantee.entity;

import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "registros_garantia", schema = "aurix")
public class RegistroGarantia {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long garantiaId;
    private String orgao; // CARTORIO/DETRAN/JUCESP
    private LocalDateTime dataRegistro;
    private String protocolo;
    private String status; // PENDENTE/REGISTRADO/REJEITADO

    public RegistroGarantia() {}
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Long getGarantiaId() { return garantiaId; }
    public void setGarantiaId(Long v) { this.garantiaId = v; }
    public String getOrgao() { return orgao; }
    public void setOrgao(String v) { this.orgao = v; }
    public LocalDateTime getDataRegistro() { return dataRegistro; }
    public void setDataRegistro(LocalDateTime v) { this.dataRegistro = v; }
    public String getProtocolo() { return protocolo; }
    public void setProtocolo(String v) { this.protocolo = v; }
    public String getStatus() { return status; }
    public void setStatus(String v) { this.status = v; }
}
```

Create `RegistroGarantiaRepository.java`:
```java
package com.aurix.platform.guarantee.repository;

import com.aurix.platform.guarantee.entity.RegistroGarantia;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface RegistroGarantiaRepository extends JpaRepository<RegistroGarantia, Long> {
    List<RegistroGarantia> findByGarantiaId(Long garantiaId);
}
```

- [ ] **Step 8: Compile to verify**

```bash
mvn -pl backend/aurix-guarantee -am compile -DskipTests
```

Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/entity/ backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/repository/ backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/dto/
git commit -m "feat(guarantee): add entities, repositories, DTOs (extracted from finanziamento + new)"
```

---

### Task 4: Guarantee — Create services + Feign client

**Files:**
- Create: `backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/service/GarantiaService.java`
- Create: `backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/service/BemService.java`
- Create: `backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/client/GarantiaClient.java`
- Create: test files

**Interfaces:**
- Consumes: GarantiaRepository, BemRepository, AvaliacaoRepository, KafkaTemplate
- Produces: GarantiaService (used by GarantiaController and GarantiaClient)

- [ ] **Step 1: Read existing GarantiaService from finanziamento**

Read `backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/service/GarantiaService.java` to understand its methods.

- [ ] **Step 2: Create GarantiaService in guarantee module**

```java
package com.aurix.platform.guarantee.service;

import com.aurix.platform.guarantee.dto.GarantiaRequest;
import com.aurix.platform.guarantee.dto.GarantiaResponse;
import com.aurix.platform.guarantee.entity.Garantia;
import com.aurix.platform.guarantee.repository.GarantiaRepository;
import com.aurix.platform.shared.event.GarantiaRegistradaEvent;
import com.aurix.platform.shared.event.GarantiaLiberadaEvent;
import com.aurix.platform.shared.event.Topics;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDate;
import java.util.List;

@Service
public class GarantiaService {
    private static final Logger log = LoggerFactory.getLogger(GarantiaService.class);
    private final GarantiaRepository garantiaRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public GarantiaService(GarantiaRepository garantiaRepository, KafkaTemplate<String, Object> kafkaTemplate) {
        this.garantiaRepository = garantiaRepository;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Transactional
    public GarantiaResponse registrar(GarantiaRequest request, Long contratoId, Long bemId) {
        Garantia garantia = new Garantia();
        garantia.setContratoId(contratoId);
        garantia.setClienteId(request.getClienteId());
        garantia.setBemId(bemId);
        garantia.setTipo(request.getTipo());
        garantia.setValor(request.getValor());
        garantia.setDataRegistro(LocalDate.now());
        garantia.setDataVencimento(request.getDataVencimento());
        garantia.setStatus("ATIVA");
        garantia = garantiaRepository.save(garantia);

        log.info("Garantia registrada: id={}, contratoId={}", garantia.getId(), contratoId);

        GarantiaRegistradaEvent event = GarantiaRegistradaEvent.registrada(
            garantia.getId(), contratoId, request.getClienteId(), request.getTipo(), request.getValor());
        kafkaTemplate.send(Topics.GUARANTEE_GARANTIA_REGISTRADA, String.valueOf(contratoId), event);

        return toResponse(garantia);
    }

    @Transactional
    public GarantiaResponse liberar(Long id, LocalDate dataBaixa) {
        Garantia garantia = garantiaRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Garantia não encontrada: " + id));
        garantia.setStatus("LIBERADA");
        garantia.setDataBaixa(dataBaixa != null ? dataBaixa : LocalDate.now());
        garantia = garantiaRepository.save(garantia);

        log.info("Garantia liberada: id={}", garantia.getId());

        GarantiaLiberadaEvent event = GarantiaLiberadaEvent.liberada(
            garantia.getId(), garantia.getContratoId(), garantia.getDataBaixa());
        kafkaTemplate.send(Topics.GUARANTEE_GARANTIA_LIBERADA, String.valueOf(garantia.getContratoId()), event);

        return toResponse(garantia);
    }

    public GarantiaResponse buscarPorId(Long id) {
        Garantia garantia = garantiaRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Garantia não encontrada: " + id));
        return toResponse(garantia);
    }

    public List<GarantiaResponse> listarPorContrato(Long contratoId) {
        return garantiaRepository.findByContratoId(contratoId).stream()
            .map(this::toResponse).toList();
    }

    public List<GarantiaResponse> listarPorCliente(Long clienteId) {
        return garantiaRepository.findByClienteId(clienteId).stream()
            .map(this::toResponse).toList();
    }

    private GarantiaResponse toResponse(Garantia g) {
        return new GarantiaResponse(g.getId(), g.getTipo(), g.getValor(),
            g.getDataRegistro(), g.getDataBaixa(), g.getStatus(), null);
    }
}
```

- [ ] **Step 3: Create GarantiaClient (Feign)**

```java
package com.aurix.platform.guarantee.client;

import com.aurix.platform.guarantee.dto.GarantiaRequest;
import com.aurix.platform.guarantee.dto.GarantiaResponse;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@FeignClient(name = "aurix-guarantee", path = "/api/guarantee")
public interface GarantiaClient {
    @PostMapping("/garantias")
    ResponseEntity<GarantiaResponse> registrar(@RequestBody GarantiaRequest request);

    @PatchMapping("/garantias/{id}/liberar")
    ResponseEntity<GarantiaResponse> liberar(@PathVariable Long id, @RequestBody LiberarGarantiaRequest request);

    @GetMapping("/garantias/{id}")
    ResponseEntity<GarantiaResponse> buscar(@PathVariable Long id);

    @GetMapping("/garantias")
    ResponseEntity<List<GarantiaResponse>> listar(@RequestParam(required = false) Long contratoId,
                                                   @RequestParam(required = false) Long clienteId);
}
```

You'll also need `LiberarGarantiaRequest.java` in `com.aurix.platform.guarantee.dto`:
```java
package com.aurix.platform.guarantee.dto;

import java.time.LocalDate;

public class LiberarGarantiaRequest {
    private LocalDate dataBaixa;

    public LocalDate getDataBaixa() { return dataBaixa; }
    public void setDataBaixa(LocalDate dataBaixa) { this.dataBaixa = dataBaixa; }
}
```

- [ ] **Step 4: Compile to verify**

```bash
mvn -pl backend/aurix-guarantee -am compile -DskipTests
```

Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/service/ backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/client/
git commit -m "feat(guarantee): add GarantiaService, BemService, and GarantiaClient Feign"
```

---

### Task 5: Guarantee — Update finanziamento to use GarantiaClient

**Files:**
- Modify: `backend/aurix-financiamento/pom.xml` (add aurix-guarantee dependency)
- Modify: `backend/aurix-financiamento/src/main/java/.../service/ContratoFinanciamentoService.java` (replace GarantiaRepository injections with GarantiaClient)
- Modify: `backend/aurix-financiamento/src/main/java/.../service/GarantiaService.java` (remove — now in guarantee module)
- Remove: Garantia entity, repository, DTOs from finanziamento (now in guarantee module)

**Interfaces:**
- Consumes: GarantiaClient from aurix-guarantee
- Produces: finanziamento compiles without local Garantia classes

- [ ] **Step 1: Add aurix-guarantee dependency to finanziamento pom.xml**

Read `backend/aurix-financiamento/pom.xml` and add:
```xml
<dependency>
    <groupId>com.aurix.platform</groupId>
    <artifactId>aurix-guarantee</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 2: Update ContratoFinanciamentoService**

Read the file, replace `GarantiaRepository` injection with `GarantiaClient` injection. Update method calls:

```java
// Before:
private final GarantiaRepository garantiaRepository;

// After:
private final GarantiaClient garantiaClient;
```

Replace `garantiaRepository.findByContratoId(id)` with `garantiaClient.listar(null, null).getBody()` etc. Adjust the logic to match the Feign client interface.

- [ ] **Step 3: Remove old Garantia classes from finanziamento**

```bash
rm backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/entity/Garantia.java
rm backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/repository/GarantiaRepository.java
rm backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/service/GarantiaService.java
rm backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/dto/request/GarantiaRequest.java
rm backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/dto/response/GarantiaResponse.java
rm backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/controller/GarantiaController.java
rm backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/job/AtualizacaoGarantiasJob.java
```

- [ ] **Step 4: Compile both modules**

```bash
mvn -pl backend/aurix-financiamento,backend/aurix-guarantee -am compile -DskipTests
```

Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add backend/aurix-financiamento/pom.xml backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/service/ContratoFinanciamentoService.java && git rm backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/entity/Garantia.java backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/repository/GarantiaRepository.java backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/service/GarantiaService.java backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/dto/request/GarantiaRequest.java backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/dto/response/GarantiaResponse.java backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/controller/GarantiaController.java backend/aurix-financiamento/src/main/java/com/aurix/platform/financiamento/job/AtualizacaoGarantiasJob.java
git commit -m "refactor(financiamento): replace local Garantia with aurix-guarantee Feign client"
```

---

### Task 6: Guarantee — Create controller and tests

**Files:**
- Create: `backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/controller/GarantiaController.java`
- Create: `backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/controller/BemController.java`
- Create: `backend/aurix-guarantee/src/test/java/com/aurix/platform/guarantee/controller/GarantiaControllerTest.java`
- Create: `backend/aurix-guarantee/src/test/java/com/aurix/platform/guarantee/service/GarantiaServiceTest.java`

**Interfaces:**
- Consumes: GarantiaService, BemService
- Produces: REST endpoints for Guarantee module

- [ ] **Step 1: Create GarantiaController**

```java
package com.aurix.platform.guarantee.controller;

import com.aurix.platform.guarantee.dto.GarantiaRequest;
import com.aurix.platform.guarantee.dto.GarantiaResponse;
import com.aurix.platform.guarantee.dto.LiberarGarantiaRequest;
import com.aurix.platform.guarantee.service.GarantiaService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/garantias")
@Tag(name = "Garantias", description = "Gestão de garantias")
public class GarantiaController {
    private static final Logger log = LoggerFactory.getLogger(GarantiaController.class);
    private final GarantiaService garantiaService;

    public GarantiaController(GarantiaService garantiaService) {
        this.garantiaService = garantiaService;
    }

    @PostMapping
    @Operation(summary = "Registrar garantia")
    public ResponseEntity<GarantiaResponse> registrar(@Valid @RequestBody GarantiaRequest request) {
        log.info("Registrando garantia para contratoId={}", request.getContratoId());
        GarantiaResponse response = garantiaService.registrar(request, request.getContratoId(), request.getBemId());
        return ResponseEntity.ok(response);
    }

    @PatchMapping("/{id}/liberar")
    @Operation(summary = "Liberar garantia")
    public ResponseEntity<GarantiaResponse> liberar(@PathVariable Long id, @RequestBody LiberarGarantiaRequest request) {
        log.info("Liberando garantia id={}", id);
        GarantiaResponse response = garantiaService.liberar(id, request.getDataBaixa());
        return ResponseEntity.ok(response);
    }

    @GetMapping("/{id}")
    @Operation(summary = "Consultar garantia por ID")
    public ResponseEntity<GarantiaResponse> buscar(@PathVariable Long id) {
        return ResponseEntity.ok(garantiaService.buscarPorId(id));
    }

    @GetMapping
    @Operation(summary = "Listar garantias")
    public ResponseEntity<List<GarantiaResponse>> listar(
            @RequestParam(required = false) Long contratoId,
            @RequestParam(required = false) Long clienteId) {
        if (contratoId != null) {
            return ResponseEntity.ok(garantiaService.listarPorContrato(contratoId));
        }
        if (clienteId != null) {
            return ResponseEntity.ok(garantiaService.listarPorCliente(clienteId));
        }
        return ResponseEntity.ok(List.of());
    }
}
```

- [ ] **Step 2: Create BemController**

```java
package com.aurix.platform.guarantee.controller;

import com.aurix.platform.guarantee.entity.Bem;
import com.aurix.platform.guarantee.service.BemService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/bens")
@Tag(name = "Bens", description = "Cadastro de bens")
public class BemController {
    private final BemService bemService;

    public BemController(BemService bemService) {
        this.bemService = bemService;
    }

    @PostMapping
    @Operation(summary = "Cadastrar bem")
    public ResponseEntity<Bem> cadastrar(@Valid @RequestBody Bem bem) {
        return ResponseEntity.ok(bemService.cadastrar(bem));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Consultar bem")
    public ResponseEntity<Bem> buscar(@PathVariable Long id) {
        return ResponseEntity.ok(bemService.buscarPorId(id));
    }

    @GetMapping
    @Operation(summary = "Listar bens")
    public ResponseEntity<List<Bem>> listar(@RequestParam(required = false) String tipo) {
        return ResponseEntity.ok(bemService.listar(tipo));
    }
}
```

- [ ] **Step 3: Create BemService**

```java
package com.aurix.platform.guarantee.service;

import com.aurix.platform.guarantee.entity.Bem;
import com.aurix.platform.guarantee.repository.BemRepository;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class BemService {
    private final BemRepository bemRepository;

    public BemService(BemRepository bemRepository) {
        this.bemRepository = bemRepository;
    }

    public Bem cadastrar(Bem bem) {
        return bemRepository.save(bem);
    }

    public Bem buscarPorId(Long id) {
        return bemRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Bem não encontrado: " + id));
    }

    public List<Bem> listar(String tipo) {
        if (tipo != null) return bemRepository.findByTipo(tipo);
        return bemRepository.findAll();
    }
}
```

- [ ] **Step 4: Create GarantiaServiceTest**

```java
package com.aurix.platform.guarantee.service;

import com.aurix.platform.guarantee.dto.GarantiaRequest;
import com.aurix.platform.guarantee.entity.Garantia;
import com.aurix.platform.guarantee.repository.GarantiaRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.kafka.core.KafkaTemplate;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class GarantiaServiceTest {

    @Mock private GarantiaRepository garantiaRepository;
    @Mock private KafkaTemplate<String, Object> kafkaTemplate;
    private GarantiaService garantiaService;

    @BeforeEach
    void setUp() {
        garantiaService = new GarantiaService(garantiaRepository, kafkaTemplate);
    }

    @Test
    void shouldRegisterGarantia() {
        var request = new GarantiaRequest();
        request.setClienteId(1L);
        request.setTipo("ALIENACAO_FIDUCIARIA");
        request.setValor(BigDecimal.valueOf(100000));
        request.setDataVencimento(LocalDate.now().plusYears(5));

        var saved = new Garantia();
        saved.setId(1L);
        saved.setContratoId(100L);
        saved.setTipo("ALIENACAO_FIDUCIARIA");

        when(garantiaRepository.save(any())).thenReturn(saved);

        var response = garantiaService.registrar(request, 100L, 200L);

        assertNotNull(response);
        assertEquals("ALIENACAO_FIDUCIARIA", response.getTipo());
        verify(garantiaRepository).save(any());
        verify(kafkaTemplate).send(anyString(), anyString(), any());
    }

    @Test
    void shouldLiberateGarantia() {
        var garantia = new Garantia();
        garantia.setId(1L);
        garantia.setContratoId(100L);
        garantia.setStatus("ATIVA");

        when(garantiaRepository.findById(1L)).thenReturn(Optional.of(garantia));
        when(garantiaRepository.save(any())).thenReturn(garantia);

        var response = garantiaService.liberar(1L, LocalDate.now());

        assertNotNull(response);
        assertEquals("LIBERADA", response.getStatus());
        verify(kafkaTemplate).send(eq("guarantee.garantia.liberada.v1"), anyString(), any());
    }

    @Test
    void shouldThrowWhenGarantiaNotFound() {
        when(garantiaRepository.findById(999L)).thenReturn(Optional.empty());
        assertThrows(IllegalArgumentException.class,
            () -> garantiaService.buscarPorId(999L));
    }
}
```

- [ ] **Step 5: Compile and run tests**

```bash
mvn test -pl backend/aurix-guarantee -am -Dtest='!*IntegrationTest' -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add backend/aurix-guarantee/src/main/java/com/aurix/platform/guarantee/controller/ backend/aurix-guarantee/src/test/
git commit -m "feat(guarantee): add controllers and unit tests"
```

---

### Task 7: Acquirer — Create module skeleton + entities + repositories

**Files:**
- Create: `backend/aurix-acquirer/pom.xml`
- Create: `backend/aurix-acquirer/src/main/java/com/aurix/platform/acquirer/AurixAcquirerApplication.java`
- Create: `backend/aurix-acquirer/src/main/resources/application.yml`
- Create: `backend/aurix-acquirer/Dockerfile`
- Create: `backend/aurix-acquirer/src/main/java/com/aurix/platform/acquirer/config/SecurityConfig.java`
- Create: `backend/aurix-acquirer/src/main/java/com/aurix/platform/acquirer/config/AcquirerKafkaConfig.java`
- Create: entity classes (Estabelecimento, Terminal, TransacaoCaptura, Liquidacao, TaxaAcquirer)
- Create: repository interfaces
- Create: DTOs (TransacaoRequest, TransacaoResponse, EstabelecimentoRequest, etc.)
- Modify: `backend/pom.xml` (add aureuis-acquirer module)

**Interfaces:**
- Consumes: shared module
- Produces: compilable module with entities and repositories

- [ ] **Step 1: Create pom.xml** (same pattern as guarantee but with artifactId aureuis-acquirer)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.aurix.platform</groupId>
        <artifactId>aurix-platform</artifactId>
        <version>1.0.0</version>
    </parent>
    <artifactId>aurix-acquirer</artifactId>
    <packaging>jar</packaging>
    <name>AURIX Acquirer</name>
    <description>Módulo de Adquirência - Captura e Liquidação de Transações</description>
    <dependencies>
        <!-- same as guarantee but without spring-cloud-starter-openfeign -->
        <dependency><groupId>com.aurix.platform</groupId><artifactId>aurix-shared</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-jpa</artifactId></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-validation</artifactId></dependency>
        <dependency><groupId>org.springframework.kafka</groupId><artifactId>spring-kafka</artifactId></dependency>
        <dependency><groupId>org.postgresql</groupId><artifactId>postgresql</artifactId><scope>runtime</scope></dependency>
        <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-test</artifactId><scope>test</scope></dependency>
        <dependency><groupId>org.springframework.kafka</groupId><artifactId>spring-kafka-test</artifactId><scope>test</scope></dependency>
    </dependencies>
</project>
```

- [ ] **Step 2: Create application main class, application.yml, SecurityConfig, Dockerfile** (same patterns as guarantee with port 8127, context-path `/api/acquirer`, group-id `aurix-acquirer-group`)

- [ ] **Step 3: Create AcquirerKafkaConfig**

```java
package com.aurix.platform.acquirer.config;

import com.aurix.platform.shared.event.Topics;
import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration("acquirerKafkaConfig")
public class AcquirerKafkaConfig {
    @Bean public NewTopic transacaoAutorizadaTopic() {
        return TopicBuilder.name(Topics.ACQUIRER_TRANSACAO_AUTORIZADA).partitions(3).replicas(1).build();
    }
    @Bean public NewTopic transacaoCapturadaTopic() {
        return TopicBuilder.name(Topics.ACQUIRER_TRANSACAO_CAPTURADA).partitions(3).replicas(1).build();
    }
    @Bean public NewTopic transacaoLiquidadaTopic() {
        return TopicBuilder.name(Topics.ACQUIRER_TRANSACAO_LIQUIDADA).partitions(3).replicas(1).build();
    }
    @Bean public NewTopic transacaoEstornadaTopic() {
        return TopicBuilder.name(Topics.ACQUIRER_TRANSACAO_ESTORNADA).partitions(3).replicas(1).build();
    }
}
```

- [ ] **Step 4: Create entity classes**

Create `Estabelecimento.java`, `Terminal.java`, `TransacaoCaptura.java`, `Liquidacao.java`, `TaxaAcquirer.java` in `com.aurix.platform.acquirer.entity` package, following the field definitions from the spec.

Each entity: `@Entity @Table(name = "acquirer_<nome>", schema = "aurix")`, JPA annotations, empty constructor, getters + setters.

Estabelecimento: id, clienteId, nomeFantasia, cnpj, bandeirasHabilitadas (String), taxaAdmin (BigDecimal), prazoLiquidacao (Integer), status (String), tenantId
Terminal: id, estabelecimentoId, modelo, tipo (String), codigo, status, tenantId
TransacaoCaptura: id, terminalId, bandeira, valor (BigDecimal), parcelas (Integer), dadosCartao (String), status, codigoAutorizacao, nsu, dataHora (LocalDateTime), tenantId
Liquidacao: id, transacaoId, valorLiquido (BigDecimal), taxaAdmin (BigDecimal), valorRepasse (BigDecimal), dataLiquidacao (LocalDateTime), status
TaxaAcquirer: id, bandeira, tipoTransacao, percentual (BigDecimal), valorFixo (BigDecimal), produtoCredito

- [ ] **Step 5: Create repository interfaces**

`EstabelecimentoRepository`, `TerminalRepository`, `TransacaoCapturaRepository`, `LiquidacaoRepository`, `TaxaAcquirerRepository` — basic JpaRepository with findBy methods matching the query requirements.

TransacaoCapturaRepository:
- findByTerminalId(Long)
- findByStatus(String)
- findByDataHoraBetween(LocalDateTime, LocalDateTime)

LiquidacaoRepository:
- findByStatus(String)
- findByTransacaoId(Long)

EstabelecimentoRepository:
- findByClienteId(Long)
- findByCnpj(String)

- [ ] **Step 6: Add module to backend/pom.xml**

```xml
<module>aurix-acquirer</module>
```

- [ ] **Step 7: Compile to verify**

```bash
mvn -pl backend/aurix-acquirer -am compile -DskipTests
```

Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add backend/aurix-acquirer/ backend/pom.xml
git commit -m "feat(acquirer): create module skeleton with entities, repositories, kafka config"
```

---

### Task 8: Acquirer — Create services

**Files:**
- Create: `EstabelecimentoService.java`
- Create: `TransacaoService.java`
- Create: `LiquidacaoService.java`

**Interfaces:**
- Consumes: repositories, KafkaTemplate
- Produces: services consumed by controllers

- [ ] **Step 1: Create TransacaoService**

```java
package com.aurix.platform.acquirer.service;

import com.aurix.platform.acquirer.entity.TransacaoCaptura;
import com.aurix.platform.acquirer.repository.TransacaoCapturaRepository;
import com.aurix.platform.shared.event.TransacaoAutorizadaEvent;
import com.aurix.platform.shared.event.TransacaoCapturadaEvent;
import com.aurix.platform.shared.event.TransacaoEstornadaEvent;
import com.aurix.platform.shared.event.Topics;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

@Service
public class TransacaoService {
    private static final Logger log = LoggerFactory.getLogger(TransacaoService.class);
    private final TransacaoCapturaRepository transacaoRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public TransacaoService(TransacaoCapturaRepository transacaoRepository, KafkaTemplate<String, Object> kafkaTemplate) {
        this.transacaoRepository = transacaoRepository;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Transactional
    public TransacaoCaptura autorizar(Long terminalId, String bandeira, java.math.BigDecimal valor, Integer parcelas, String dadosCartao) {
        TransacaoCaptura t = new TransacaoCaptura();
        t.setTerminalId(terminalId);
        t.setBandeira(bandeira);
        t.setValor(valor);
        t.setParcelas(parcelas);
        t.setDadosCartao(dadosCartao);
        t.setStatus("AUTORIZADA");
        t.setCodigoAutorizacao(UUID.randomUUID().toString().substring(0, 8).toUpperCase());
        t.setNsu(String.valueOf(System.currentTimeMillis()));
        t.setDataHora(LocalDateTime.now());
        t = transacaoRepository.save(t);

        TransacaoAutorizadaEvent event = TransacaoAutorizadaEvent.autorizada(
            t.getId(), terminalId, valor, bandeira, t.getCodigoAutorizacao(), t.getNsu());
        kafkaTemplate.send(Topics.ACQUIRER_TRANSACAO_AUTORIZADA, String.valueOf(terminalId), event);

        log.info("Transação autorizada: id={}, valor={}", t.getId(), valor);
        return t;
    }

    @Transactional
    public TransacaoCaptura capturar(Long id) {
        TransacaoCaptura t = transacaoRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Transação não encontrada: " + id));
        if (!"AUTORIZADA".equals(t.getStatus())) {
            throw new IllegalStateException("Transação " + id + " não está autorizada");
        }
        t.setStatus("CAPTURADA");
        t = transacaoRepository.save(t);

        TransacaoCapturadaEvent event = TransacaoCapturadaEvent.capturada(
            t.getId(), t.getValor(), t.getParcelas(), t.getBandeira());
        kafkaTemplate.send(Topics.ACQUIRER_TRANSACAO_CAPTURADA, String.valueOf(t.getTerminalId()), event);

        log.info("Transação capturada: id={}", t.getId());
        return t;
    }

    @Transactional
    public TransacaoCaptura estornar(Long id, String motivo) {
        TransacaoCaptura t = transacaoRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Transação não encontrada: " + id));
        if ("ESTORNADA".equals(t.getStatus())) {
            throw new IllegalStateException("Transação " + id + " já está estornada");
        }
        t.setStatus("ESTORNADA");
        t = transacaoRepository.save(t);

        TransacaoEstornadaEvent event = TransacaoEstornadaEvent.estornada(t.getId(), t.getValor(), motivo);
        kafkaTemplate.send(Topics.ACQUIRER_TRANSACAO_ESTORNADA, String.valueOf(t.getTerminalId()), event);

        log.info("Transação estornada: id={}, motivo={}", t.getId(), motivo);
        return t;
    }

    public TransacaoCaptura buscar(Long id) {
        return transacaoRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Transação não encontrada: " + id));
    }

    public List<TransacaoCaptura> listar(Long terminalId, String status) {
        if (terminalId != null) return transacaoRepository.findByTerminalId(terminalId);
        if (status != null) return transacaoRepository.findByStatus(status);
        return transacaoRepository.findAll();
    }
}
```

- [ ] **Step 2: Create EstabelecimentoService** (CRUD for estabelecimentos and terminais)

```java
package com.aurix.platform.acquirer.service;

import com.aurix.platform.acquirer.entity.Estabelecimento;
import com.aurix.platform.acquirer.entity.Terminal;
import com.aurix.platform.acquirer.repository.EstabelecimentoRepository;
import com.aurix.platform.acquirer.repository.TerminalRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class EstabelecimentoService {
    private static final Logger log = LoggerFactory.getLogger(EstabelecimentoService.class);
    private final EstabelecimentoRepository estabelecimentoRepository;
    private final TerminalRepository terminalRepository;

    public EstabelecimentoService(EstabelecimentoRepository estabelecimentoRepository, TerminalRepository terminalRepository) {
        this.estabelecimentoRepository = estabelecimentoRepository;
        this.terminalRepository = terminalRepository;
    }

    public Estabelecimento cadastrar(Estabelecimento e) {
        e.setStatus("ATIVO");
        Estabelecimento saved = estabelecimentoRepository.save(e);
        log.info("Estabelecimento cadastrado: id={}", saved.getId());
        return saved;
    }

    public Estabelecimento buscar(Long id) {
        return estabelecimentoRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Estabelecimento não encontrado: " + id));
    }

    public List<Estabelecimento> listar() {
        return estabelecimentoRepository.findAll();
    }

    public Terminal cadastrarTerminal(Terminal t) {
        Terminal saved = terminalRepository.save(t);
        log.info("Terminal cadastrado: id={}, estabelecimentoId={}", saved.getId(), t.getEstabelecimentoId());
        return saved;
    }

    public List<Terminal> listarTerminais(Long estabelecimentoId) {
        return terminalRepository.findByEstabelecimentoId(estabelecimentoId);
    }
}
```

- [ ] **Step 3: Create LiquidacaoService**

```java
package com.aurix.platform.acquirer.service;

import com.aurix.platform.acquirer.entity.Liquidacao;
import com.aurix.platform.acquirer.entity.TransacaoCaptura;
import com.aurix.platform.acquirer.repository.LiquidacaoRepository;
import com.aurix.platform.acquirer.repository.TransacaoCapturaRepository;
import com.aurix.platform.shared.event.TransacaoLiquidadaEvent;
import com.aurix.platform.shared.event.Topics;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Service
public class LiquidacaoService {
    private static final Logger log = LoggerFactory.getLogger(LiquidacaoService.class);
    private final LiquidacaoRepository liquidacaoRepository;
    private final TransacaoCapturaRepository transacaoRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public LiquidacaoService(LiquidacaoRepository liquidacaoRepository,
                             TransacaoCapturaRepository transacaoRepository,
                             KafkaTemplate<String, Object> kafkaTemplate) {
        this.liquidacaoRepository = liquidacaoRepository;
        this.transacaoRepository = transacaoRepository;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Transactional
    public Liquidacao processarLiquidacao(Long transacaoId, BigDecimal taxaAdmin) {
        TransacaoCaptura t = transacaoRepository.findById(transacaoId)
            .orElseThrow(() -> new IllegalArgumentException("Transação não encontrada: " + transacaoId));

        Liquidacao liq = new Liquidacao();
        liq.setTransacaoId(transacaoId);
        liq.setTaxaAdmin(taxaAdmin);
        liq.setValorLiquido(t.getValor().subtract(taxaAdmin));
        liq.setValorRepasse(liq.getValorLiquido());
        liq.setDataLiquidacao(LocalDateTime.now());
        liq.setStatus("PROCESADA");
        liq = liquidacaoRepository.save(liq);

        t.setStatus("LIQUIDADA");
        transacaoRepository.save(t);

        TransacaoLiquidadaEvent event = TransacaoLiquidadaEvent.liquidada(
            liq.getId(), transacaoId, liq.getValorLiquido(), liq.getValorRepasse());
        kafkaTemplate.send(Topics.ACQUIRER_TRANSACAO_LIQUIDADA, String.valueOf(transacaoId), event);

        log.info("Liquidação processada: id={}, transacaoId={}", liq.getId(), transacaoId);
        return liq;
    }

    public List<Liquidacao> listarPendentes() {
        return liquidacaoRepository.findByStatus("PENDENTE");
    }
}
```

- [ ] **Step 4: Compile to verify**

```bash
mvn -pl backend/aurix-acquirer -am compile -DskipTests
```

Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add backend/aurix-acquirer/src/main/java/com/aurix/platform/acquirer/service/
git commit -m "feat(acquirer): add TransacaoService, EstabelecimentoService, LiquidacaoService"
```

---

### Task 9: Acquirer — Create controllers + tests

**Files:**
- Create: `TransacaoController.java`
- Create: `EstabelecimentoController.java`
- Create: `LiquidacaoController.java`
- Create: test files

- [ ] **Step 1: Create TransacaoController**

```java
package com.aurix.platform.acquirer.controller;

import com.aurix.platform.acquirer.entity.TransacaoCaptura;
import com.aurix.platform.acquirer.service.TransacaoService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/transacoes")
@Tag(name = "Transações", description = "Autorização, captura e estorno de transações")
public class TransacaoController {
    private final TransacaoService transacaoService;

    public TransacaoController(TransacaoService transacaoService) {
        this.transacaoService = transacaoService;
    }

    @PostMapping("/autorizar")
    @Operation(summary = "Autorizar transação")
    public ResponseEntity<TransacaoCaptura> autorizar(@RequestBody Map<String, Object> request) {
        Long terminalId = Long.valueOf(request.get("terminalId").toString());
        String bandeira = (String) request.get("bandeira");
        BigDecimal valor = new BigDecimal(request.get("valor").toString());
        Integer parcelas = request.containsKey("parcelas") ? Integer.valueOf(request.get("parcelas").toString()) : 1;
        String dadosCartao = (String) request.get("dadosCartao");
        return ResponseEntity.ok(transacaoService.autorizar(terminalId, bandeira, valor, parcelas, dadosCartao));
    }

    @PostMapping("/{id}/capturar")
    @Operation(summary = "Capturar transação autorizada")
    public ResponseEntity<TransacaoCaptura> capturar(@PathVariable Long id) {
        return ResponseEntity.ok(transacaoService.capturar(id));
    }

    @PostMapping("/{id}/estornar")
    @Operation(summary = "Estornar transação")
    public ResponseEntity<TransacaoCaptura> estornar(@PathVariable Long id, @RequestBody Map<String, String> body) {
        return ResponseEntity.ok(transacaoService.estornar(id, body.getOrDefault("motivo", "")));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Consultar transação")
    public ResponseEntity<TransacaoCaptura> buscar(@PathVariable Long id) {
        return ResponseEntity.ok(transacaoService.buscar(id));
    }

    @GetMapping
    @Operation(summary = "Listar transações")
    public ResponseEntity<List<TransacaoCaptura>> listar(
            @RequestParam(required = false) Long terminalId,
            @RequestParam(required = false) String status) {
        return ResponseEntity.ok(transacaoService.listar(terminalId, status));
    }
}
```

- [ ] **Step 2: Create EstabelecimentoController**

```java
package com.aurix.platform.acquirer.controller;

import com.aurix.platform.acquirer.entity.Estabelecimento;
import com.aurix.platform.acquirer.entity.Terminal;
import com.aurix.platform.acquirer.service.EstabelecimentoService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/estabelecimentos")
@Tag(name = "Estabelecimentos", description = "Cadastro de estabelecimentos e terminais")
public class EstabelecimentoController {
    private final EstabelecimentoService estabelecimentoService;

    public EstabelecimentoController(EstabelecimentoService estabelecimentoService) {
        this.estabelecimentoService = estabelecimentoService;
    }

    @PostMapping
    @Operation(summary = "Cadastrar estabelecimento")
    public ResponseEntity<Estabelecimento> cadastrar(@Valid @RequestBody Estabelecimento e) {
        return ResponseEntity.ok(estabelecimentoService.cadastrar(e));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Consultar estabelecimento")
    public ResponseEntity<Estabelecimento> buscar(@PathVariable Long id) {
        return ResponseEntity.ok(estabelecimentoService.buscar(id));
    }

    @GetMapping
    @Operation(summary = "Listar estabelecimentos")
    public ResponseEntity<List<Estabelecimento>> listar() {
        return ResponseEntity.ok(estabelecimentoService.listar());
    }

    @PostMapping("/{id}/terminais")
    @Operation(summary = "Cadastrar terminal")
    public ResponseEntity<Terminal> cadastrarTerminal(@PathVariable Long id, @Valid @RequestBody Terminal t) {
        t.setEstabelecimentoId(id);
        return ResponseEntity.ok(estabelecimentoService.cadastrarTerminal(t));
    }

    @GetMapping("/{id}/terminais")
    @Operation(summary = "Listar terminais do estabelecimento")
    public ResponseEntity<List<Terminal>> listarTerminais(@PathVariable Long id) {
        return ResponseEntity.ok(estabelecimentoService.listarTerminais(id));
    }
}
```

- [ ] **Step 3: Create LiquidacaoController**

```java
package com.aurix.platform.acquirer.controller;

import com.aurix.platform.acquirer.entity.Liquidacao;
import com.aurix.platform.acquirer.service.LiquidacaoService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/liquidacoes")
@Tag(name = "Liquidações", description = "Processamento de liquidações")
public class LiquidacaoController {
    private final LiquidacaoService liquidacaoService;

    public LiquidacaoController(LiquidacaoService liquidacaoService) {
        this.liquidacaoService = liquidacaoService;
    }

    @PostMapping("/processar")
    @Operation(summary = "Processar liquidação")
    public ResponseEntity<Liquidacao> processar(@RequestBody Map<String, Object> request) {
        Long transacaoId = Long.valueOf(request.get("transacaoId").toString());
        BigDecimal taxaAdmin = new BigDecimal(request.get("taxaAdmin").toString());
        return ResponseEntity.ok(liquidacaoService.processarLiquidacao(transacaoId, taxaAdmin));
    }

    @GetMapping("/pendentes")
    @Operation(summary = "Listar liquidações pendentes")
    public ResponseEntity<List<Liquidacao>> pendentes() {
        return ResponseEntity.ok(liquidacaoService.listarPendentes());
    }
}
```

- [ ] **Step 4: Create TransacaoServiceTest**

```java
package com.aurix.platform.acquirer.service;

import com.aurix.platform.acquirer.entity.TransacaoCaptura;
import com.aurix.platform.acquirer.repository.TransacaoCapturaRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.kafka.core.KafkaTemplate;

import java.math.BigDecimal;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class TransacaoServiceTest {

    @Mock private TransacaoCapturaRepository transacaoRepository;
    @Mock private KafkaTemplate<String, Object> kafkaTemplate;
    private TransacaoService transacaoService;

    @BeforeEach
    void setUp() {
        transacaoService = new TransacaoService(transacaoRepository, kafkaTemplate);
    }

    @Test
    void shouldAuthorizeTransaction() {
        var saved = new TransacaoCaptura();
        saved.setId(1L);
        saved.setStatus("AUTORIZADA");
        saved.setCodigoAutorizacao("ABC123");

        when(transacaoRepository.save(any())).thenReturn(saved);

        var result = transacaoService.autorizar(1L, "VISA", BigDecimal.valueOf(100), 1, "token123");

        assertNotNull(result);
        assertEquals("AUTORIZADA", result.getStatus());
        verify(kafkaTemplate).send(anyString(), anyString(), any());
    }

    @Test
    void shouldCaptureAuthorizedTransaction() {
        var t = new TransacaoCaptura();
        t.setId(1L);
        t.setStatus("AUTORIZADA");
        t.setValor(BigDecimal.valueOf(100));
        t.setTerminalId(1L);

        when(transacaoRepository.findById(1L)).thenReturn(Optional.of(t));
        when(transacaoRepository.save(any())).thenReturn(t);

        var result = transacaoService.capturar(1L);

        assertEquals("CAPTURADA", result.getStatus());
        verify(kafkaTemplate).send(eq("acquirer.transacao.capturada.v1"), anyString(), any());
    }

    @Test
    void shouldThrowWhenCapturingNonAuthorized() {
        var t = new TransacaoCaptura();
        t.setId(1L);
        t.setStatus("CAPTURADA");

        when(transacaoRepository.findById(1L)).thenReturn(Optional.of(t));

        assertThrows(IllegalStateException.class, () -> transacaoService.capturar(1L));
    }
}
```

- [ ] **Step 5: Compile and run tests**

```bash
mvn test -pl backend/aurix-acquirer -am -Dtest='!*IntegrationTest' -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add backend/aurix-acquirer/src/main/java/com/aurix/platform/acquirer/controller/ backend/aurix-acquirer/src/test/
git commit -m "feat(acquirer): add controllers and unit tests"
```

---

### Task 10: Collections — Create module skeleton + entities + repositories

**Files:**
- Create: `backend/aurix-collections/pom.xml`
- Create: `backend/aurix-collections/src/main/java/com/aurix/platform/collections/AurixCollectionsApplication.java`
- Create: `backend/aurix-collections/src/main/resources/application.yml`
- Create: `backend/aurix-collections/Dockerfile`
- Create: `backend/aurix-collections/src/main/java/.../config/SecurityConfig.java`
- Create: `backend/aurix-collections/src/main/java/.../config/CollectionsKafkaConfig.java`
- Create: entity classes (Cobranca, CarnetParcela, Acordo, Negativacao, RegistroCobranca)
- Create: repository interfaces
- Modify: `backend/pom.xml`

- [ ] **Step 1: Create pom.xml** (same pattern as acquirer, with artifactId aureuis-collections, name "AURIX Collections", description "Módulo de Cobrança")

- [ ] **Step 2: Create application main class, application.yml, SecurityConfig** (port 8128, context-path `/api/collections`, group-id `aurix-collections-group`)

- [ ] **Step 3: Create CollectionsKafkaConfig**

```java
package com.aurix.platform.collections.config;

import com.aurix.platform.shared.event.Topics;
import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration("collectionsKafkaConfig")
public class CollectionsKafkaConfig {
    @Bean public NewTopic boletoEmitidoTopic() {
        return TopicBuilder.name(Topics.COLLECTIONS_BOLETO_EMITIDO).partitions(3).replicas(1).build();
    }
    @Bean public NewTopic cobrancaPagaTopic() {
        return TopicBuilder.name(Topics.COLLECTIONS_COBRANCA_PAGA).partitions(3).replicas(1).build();
    }
    @Bean public NewTopic cobrancaNegativadaTopic() {
        return TopicBuilder.name(Topics.COLLECTIONS_COBRANCA_NEGATIVADA).partitions(3).replicas(1).build();
    }
    @Bean public NewTopic cobrancaCanceladaTopic() {
        return TopicBuilder.name(Topics.COLLECTIONS_COBRANCA_CANCELADA).partitions(3).replicas(1).build();
    }
}
```

- [ ] **Step 4: Create entity classes**

Create in `com.aurix.platform.collections.entity`:

Cobranca: id, clienteId, tipo (BOLETO/CARNE/ACORDO), valor (BigDecimal), valorPago (BigDecimal), dataVencimento (LocalDate), dataPagamento (LocalDate), status (String), nossoNumero (String), linhaDigitavel (String), codigoBarras (String), tenantId

CarnetParcela: id, cobrancaId, numeroParcela, valor (BigDecimal), dataVencimento (LocalDate), status, nossoNumero

Acordo: id, cobrancaOrigemId, clienteId, valorTotal (BigDecimal), numeroParcelas, valorParcela (BigDecimal), dataPrimeiroVencimento (LocalDate), status

Negativacao: id, cobrancaId, orgao (String), dataEnvio (LocalDate), dataBaixa (LocalDate), status

RegistroCobranca: id, cobrancaId, camara (String), dataRegistro (LocalDateTime), protocolo, status

Each with `@Entity @Table(name = "...", schema = "aurix")`, empty constructor, getters + setters.

- [ ] **Step 5: Create repository interfaces**

CobrancaRepository:
- findByClienteId(Long)
- findByStatus(String)
- findByNossoNumero(String)
- findByClienteIdAndStatus(Long, String)

CarnetParcelaRepository:
- findByCobrancaId(Long)
- findByStatus(String)

AcordoRepository:
- findByClienteId(Long)
- findByStatus(String)

NegativacaoRepository:
- findByCobrancaId(Long)
- findByStatus(String)

RegistroCobrancaRepository:
- findByCobrancaId(Long)
- findByStatus(String)

- [ ] **Step 6: Add module to backend/pom.xml**

- [ ] **Step 7: Compile**

```bash
mvn -pl backend/aurix-collections -am compile -DskipTests
```

Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add backend/aurix-collections/ backend/pom.xml
git commit -m "feat(collections): create module skeleton with entities, repositories, kafka config"
```

---

### Task 11: Collections — Create services

**Files:**
- Create: `BoletoService.java`
- Create: `CarnetService.java`
- Create: `AcordoService.java`
- Create: `NegativacaoService.java`

- [ ] **Step 1: Create BoletoService**

```java
package com.aurix.platform.collections.service;

import com.aurix.platform.collections.entity.Cobranca;
import com.aurix.platform.collections.repository.CobrancaRepository;
import com.aurix.platform.shared.event.BoletoEmitidoEvent;
import com.aurix.platform.shared.event.CobrancaPagaEvent;
import com.aurix.platform.shared.event.CobrancaCanceladaEvent;
import com.aurix.platform.shared.event.Topics;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.List;
import java.util.UUID;

@Service
public class BoletoService {
    private static final Logger log = LoggerFactory.getLogger(BoletoService.class);
    private final CobrancaRepository cobrancaRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public BoletoService(CobrancaRepository cobrancaRepository, KafkaTemplate<String, Object> kafkaTemplate) {
        this.cobrancaRepository = cobrancaRepository;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Transactional
    public Cobranca emitir(Long clienteId, BigDecimal valor, LocalDate dataVencimento) {
        Cobranca c = new Cobranca();
        c.setClienteId(clienteId);
        c.setTipo("BOLETO");
        c.setValor(valor);
        c.setDataVencimento(dataVencimento);
        c.setStatus("EMITIDA");
        c.setNossoNumero(UUID.randomUUID().toString().replace("-", "").substring(0, 15));
        c = cobrancaRepository.save(c);

        BoletoEmitidoEvent event = BoletoEmitidoEvent.emitido(c.getId(), c.getNossoNumero(), valor, dataVencimento, clienteId);
        kafkaTemplate.send(Topics.COLLECTIONS_BOLETO_EMITIDO, String.valueOf(clienteId), event);

        log.info("Boleto emitido: id={}, nossoNumero={}", c.getId(), c.getNossoNumero());
        return c;
    }

    public Cobranca consultar(String nossoNumero) {
        return cobrancaRepository.findByNossoNumero(nossoNumero)
            .orElseThrow(() -> new IllegalArgumentException("Boleto não encontrado: " + nossoNumero));
    }

    @Transactional
    public Cobranca registrarPagamento(Long id, BigDecimal valorPago) {
        Cobranca c = cobrancaRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Cobrança não encontrada: " + id));
        c.setValorPago(valorPago);
        c.setDataPagamento(LocalDate.now());
        c.setStatus("PAGA");
        c = cobrancaRepository.save(c);

        CobrancaPagaEvent event = CobrancaPagaEvent.paga(id, valorPago, LocalDate.now());
        kafkaTemplate.send(Topics.COLLECTIONS_COBRANCA_PAGA, String.valueOf(c.getClienteId()), event);

        log.info("Cobrança paga: id={}, valor={}", id, valorPago);
        return c;
    }

    @Transactional
    public Cobranca cancelar(Long id, String motivo) {
        Cobranca c = cobrancaRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Cobrança não encontrada: " + id));
        c.setStatus("CANCELADA");
        c = cobrancaRepository.save(c);

        CobrancaCanceladaEvent event = CobrancaCanceladaEvent.cancelada(id, motivo);
        kafkaTemplate.send(Topics.COLLECTIONS_COBRANCA_CANCELADA, String.valueOf(c.getClienteId()), event);

        log.info("Cobrança cancelada: id={}, motivo={}", id, motivo);
        return c;
    }

    public List<Cobranca> listar(Long clienteId, String status) {
        if (clienteId != null && status != null) return cobrancaRepository.findByClienteIdAndStatus(clienteId, status);
        if (clienteId != null) return cobrancaRepository.findByClienteId(clienteId);
        if (status != null) return cobrancaRepository.findByStatus(status);
        return cobrancaRepository.findAll();
    }
}
```

- [ ] **Step 2: Create CarnetService, AcordoService, NegativacaoService** (follow same pattern — inject repository + KafkaTemplate, create/update methods with event publishing, basic queries)

CarnetService:
- criar(cobrancaId, numeroParcelas, valorTotal, ...) — cria N parcelas
- listarPorCobranca(cobrancaId)
- pagarParcela(parcelaId)

AcordoService:
- propor(cobrancaOrigemId, clienteId, valorTotal, numeroParcelas, ...)
- listarPorCliente(clienteId)
- quitar(acordoId)

NegativacaoService:
- negativar(cobrancaId, orgao)
- baixar(negativacaoId)
- listarAtivas()

- [ ] **Step 3: Compile to verify**

```bash
mvn -pl backend/aurix-collections -am compile -DskipTests
```

Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add backend/aurix-collections/src/main/java/com/aurix/platform/collections/service/
git commit -m "feat(collections): add BoletoService, CarnetService, AcordoService, NegativacaoService"
```

---

### Task 12: Collections — Create controllers + tests

**Files:**
- Create: `BoletoController.java`
- Create: `CarnetController.java`
- Create: `AcordoController.java`
- Create: `NegativacaoController.java`
- Create: test files

- [ ] **Step 1: Create BoletoController**

```java
package com.aurix.platform.collections.controller;

import com.aurix.platform.collections.entity.Cobranca;
import com.aurix.platform.collections.service.BoletoService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/boletos")
@Tag(name = "Boletos", description = "Emissão e gestão de boletos")
public class BoletoController {
    private final BoletoService boletoService;

    public BoletoController(BoletoService boletoService) {
        this.boletoService = boletoService;
    }

    @PostMapping("/emitir")
    @Operation(summary = "Emitir boleto")
    public ResponseEntity<Cobranca> emitir(@RequestBody Map<String, Object> request) {
        Long clienteId = Long.valueOf(request.get("clienteId").toString());
        BigDecimal valor = new BigDecimal(request.get("valor").toString());
        LocalDate dataVencimento = LocalDate.parse(request.get("dataVencimento").toString());
        return ResponseEntity.ok(boletoService.emitir(clienteId, valor, dataVencimento));
    }

    @GetMapping("/{nossoNumero}")
    @Operation(summary = "Consultar boleto")
    public ResponseEntity<Cobranca> consultar(@PathVariable String nossoNumero) {
        return ResponseEntity.ok(boletoService.consultar(nossoNumero));
    }

    @PostMapping("/{id}/pagar")
    @Operation(summary = "Registrar pagamento")
    public ResponseEntity<Cobranca> pagar(@PathVariable Long id, @RequestBody Map<String, Object> request) {
        BigDecimal valorPago = new BigDecimal(request.get("valorPago").toString());
        return ResponseEntity.ok(boletoService.registrarPagamento(id, valorPago));
    }

    @PostMapping("/{id}/cancelar")
    @Operation(summary = "Cancelar cobrança")
    public ResponseEntity<Cobranca> cancelar(@PathVariable Long id, @RequestBody Map<String, String> request) {
        return ResponseEntity.ok(boletoService.cancelar(id, request.getOrDefault("motivo", "")));
    }

    @GetMapping
    @Operation(summary = "Listar cobranças")
    public ResponseEntity<List<Cobranca>> listar(
            @RequestParam(required = false) Long clienteId,
            @RequestParam(required = false) String status) {
        return ResponseEntity.ok(boletoService.listar(clienteId, status));
    }
}
```

- [ ] **Step 2: Create CarnetController, AcordoController, NegativacaoController**

Each follows the same pattern — inject service, map request body to method calls, return ResponseEntity with appropriate status codes.

- [ ] **Step 3: Create BoletoServiceTest**

```java
package com.aurix.platform.collections.service;

import com.aurix.platform.collections.entity.Cobranca;
import com.aurix.platform.collections.repository.CobrancaRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.kafka.core.KafkaTemplate;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class BoletoServiceTest {

    @Mock private CobrancaRepository cobrancaRepository;
    @Mock private KafkaTemplate<String, Object> kafkaTemplate;
    private BoletoService boletoService;

    @BeforeEach
    void setUp() {
        boletoService = new BoletoService(cobrancaRepository, kafkaTemplate);
    }

    @Test
    void shouldEmitBoleto() {
        var saved = new Cobranca();
        saved.setId(1L);
        saved.setNossoNumero("ABC123");
        saved.setStatus("EMITIDA");

        when(cobrancaRepository.save(any())).thenReturn(saved);

        var result = boletoService.emitir(1L, BigDecimal.valueOf(1000), LocalDate.now().plusDays(30));

        assertNotNull(result);
        assertEquals("EMITIDA", result.getStatus());
        verify(kafkaTemplate).send(anyString(), anyString(), any());
    }

    @Test
    void shouldRegisterPayment() {
        var cobranca = new Cobranca();
        cobranca.setId(1L);
        cobranca.setClienteId(1L);
        cobranca.setStatus("EMITIDA");

        when(cobrancaRepository.findById(1L)).thenReturn(Optional.of(cobranca));
        when(cobrancaRepository.save(any())).thenReturn(cobranca);

        var result = boletoService.registrarPagamento(1L, BigDecimal.valueOf(1000));

        assertEquals("PAGA", result.getStatus());
        verify(kafkaTemplate).send(eq("collections.cobranca.paga.v1"), anyString(), any());
    }
}
```

- [ ] **Step 4: Compile and run tests**

```bash
mvn test -pl backend/aurix-collections -am -Dtest='!*IntegrationTest' -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add backend/aurix-collections/src/main/java/com/aurix/platform/collections/controller/ backend/aurix-collections/src/test/
git commit -m "feat(collections): add controllers and unit tests"
```

---

### Task 13: Infra — Docker, Docker Compose, Traefik, E2E

**Files:**
- Create: `backend/aurix-acquirer/Dockerfile` (if not yet created)
- Create: `backend/aurix-collections/Dockerfile` (if not yet created)
- Create: `backend/aurix-guarantee/Dockerfile` (if not yet created)
- Modify: `infrastructure/docker-compose.yml`
- Modify: `infrastructure/traefik/dynamic.yml`
- Modify: `aurix-tests/e2e/config.py`

- [ ] **Step 1: Read existing docker-compose.yml** to understand service patterns

- [ ] **Step 2: Add 3 services to docker-compose.yml**

```yaml
aurix-acquirer:
  build:
    context: ../backend
    dockerfile: aurix-acquirer/Dockerfile
  ports:
    - "8127:8127"
  environment:
    - SPRING_PROFILES_ACTIVE=dev
    - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/aurix
    - SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka-1:29092
  depends_on:
    postgres:
      condition: service_healthy
    kafka-1:
      condition: service_started
  networks:
    - aurix-platform-network
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:8127/api/acquirer/actuator/health"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 40s

aurix-collections:
  # same pattern, port 8128, context /api/collections

aurix-guarantee:
  # same pattern, port 8129, context /api/guarantee
```

- [ ] **Step 3: Add routes to traefik/dynamic.yml**

```yaml
- "traefik.http.routers.aurix-acquirer.rule=PathPrefix(`/api/acquirer`)"
- "traefik.http.routers.aurix-acquirer.service=aurix-acquirer"
- "traefik.http.services.aurix-acquirer.loadbalancer.server.port=8127"
# same for collections (8128) and guarantee (8129)
```

- [ ] **Step 4: Add health endpoints to e2e/config.py**

```python
"aurix-acquirer": "http://localhost:8127/api/acquirer/actuator/health",
"aurix-collections": "http://localhost:8128/api/collections/actuator/health",
"aurix-guarantee": "http://localhost:8129/api/guarantee/actuator/health",
```

- [ ] **Step 5: Commit**

```bash
git add infrastructure/docker-compose.yml infrastructure/traefik/dynamic.yml aurix-tests/e2e/config.py
git commit -m "infra: add acquirer, collections, guarantee services to docker-compose, traefik, and e2e config"
```

---

### Task 14: Final build verification

**Files:** (none — just run commands)

- [ ] **Step 1: Full compile check**

```bash
mvn clean compile -DskipTests -q
```

Expected: BUILD SUCCESS

- [ ] **Step 2: Run unit tests for all 3 new modules**

```bash
mvn test -pl backend/aurix-guarantee,backend/aurix-acquirer,backend/aurix-collections -am -Dtest='!*IntegrationTest,!*ContractTest' -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Verify git status**

```bash
git status
git log --oneline -15
```
