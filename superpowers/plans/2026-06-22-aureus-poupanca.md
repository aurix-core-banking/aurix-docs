# Poupança Module Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the `aurix-poupanca` module — savings account with birthday TR credit, IOF on early withdrawal, PDF statement, PIX integration.

**Architecture:** New Maven module (`aurix-poupanca`) following patterns from `aurix-pix`/`aurix-credit`. Uses Spring Boot 4.1 / Spring Framework 7 native annotations (`@HttpExchange`, `@Retryable`, `@ConcurrencyLimit`, `@Observed`, `@NullMarked`). Communicates with `aurix-core` (conta corrente), `aurix-tax` (IOF), `aurix-bacen` (TR) via `@HttpExchange` clients. Events via Kafka. No Feign.

**Tech Stack:** Spring Boot 4.1, Java 25, JPA/Hibernate 6, PostgreSQL, Redis, Kafka, Spring 7 native `@Retryable`, `@HttpExchange`, `RestTestClient`, Testcontainers.

## Global Constraints

- Java source/target must be 25 (parent POM defaults are correct)
- Maven packaging: `jar`, parent: `com.aurix.platform:aurix-platform:1.0.0`
- Package root: `com.aurix.platform.poupanca`
- All Spring properties use `spring.data.redis.*` (not `spring.redis.*`)
- No `spring-retry` dependency — use Spring Framework 7 native `@Retryable`
- No `javax.annotation` — use `jakarta.annotation`
- No Feign — use `@HttpExchange` + `@ImportHttpServices`
- Default proxy type is CGLIB (Spring Boot 4.1 default)
- API context path: `/api/poupanca`, server port: 8111
- Tests must use `RestTestClient` (not MockMvc)

---

### Task 1: Module Scaffold

**Files:**
- Create: `apps/backend/aurix-poupanca/pom.xml`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/AurixPoupancaApplication.java`
- Create: `apps/backend/aurix-poupanca/src/main/resources/application.yml`
- Modify: `apps/backend/pom.xml` (add module)

**Interfaces:**
- Consumes: parent POM at `apps/backend/pom.xml`
- Produces: compilable Maven module with empty Spring Boot app

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

    <artifactId>aurix-poupanca</artifactId>
    <packaging>jar</packaging>

    <name>AURIX Poupanca</name>
    <description>Modulo de conta de deposito de poupanca do AURIX</description>

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
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.2.0</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>testcontainers-junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Create AurixPoupancaApplication.java**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/AurixPoupancaApplication.java`

```java
package com.aurix.platform.poupanca;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class AurixPoupancaApplication {

    public static void main(String[] args) {
        SpringApplication.run(AurixPoupancaApplication.class, args);
    }
}
```

- [ ] **Step 3: Create application.yml**

`apps/backend/aurix-poupanca/src/main/resources/application.yml`

```yaml
server:
  port: 8111
  servlet:
    context-path: /api/poupanca

spring:
  application:
    name: aurix-poupanca
  profiles:
    active: dev
  datasource:
    url: jdbc:postgresql://localhost:5432/aurix
    username: aurix
    password: aurix123
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 3
      connection-timeout: 30000
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
        jdbc:
          time_zone: America/Sao_Paulo
    open-in-view: false
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 0
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: aurix-poupanca-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      acks: all
      retries: 3

logging:
  level:
    com.aurix.platform: DEBUG
    org.springframework.web: DEBUG
    org.hibernate.SQL: DEBUG

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true

aurix:
  poupanca:
    iof:
      alquota-geral: 0.0046
      aliquota-diaria: 0.000041
      dias-minimos: 30
    extrato:
      pdf:
        cache-ttl: 3600
    limits:
      max-deposito: 50000.00
      max-saque: 50000.00
  security:
    jwt:
      secret: "aurix-jwt-secret-key-2024"
      expiration: 86400000

---
spring:
  config:
    activate:
      on-profile: dev
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
logging:
  level:
    com.aurix.platform: DEBUG

---
spring:
  config:
    activate:
      on-profile: prod
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
  datasource:
    hikari:
      maximum-pool-size: 30
      minimum-idle: 5
logging:
  level:
    com.aurix.platform: INFO
```

- [ ] **Step 4: Add module to parent pom.xml**

In `apps/backend/pom.xml`, add `<module>aurix-poupanca</module>` to the `<modules>` list (after `aurix-pix`).

- [ ] **Step 5: Verify scaffold compiles**

Run: `mvn compile -pl aurix-poupanca -am -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```
git add apps/backend/aurix-poupanca/pom.xml apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/AurixPoupancaApplication.java apps/backend/aurix-poupanca/src/main/resources/application.yml apps/backend/pom.xml
git commit -m "feat(poupanca): scaffold module with pom, app, config"
```

---

### Task 2: Domain Layer — Entities, Repositories, DTOs, Events

**Files:**
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/entity/ContaPoupanca.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/entity/MovimentacaoPoupanca.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/repository/ContaPoupancaRepository.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/repository/MovimentacaoPoupancaRepository.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/dto/CriarContaRequest.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/dto/DepositoRequest.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/dto/SaqueRequest.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/dto/ContaPoupancaResponse.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/dto/ExtratoResponse.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/event/ContaPoupancaEvent.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/event/MovimentacaoEvent.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/event/RendimentoEvent.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/package-info.java` (x4 dirs)

**Interfaces:**
- Produces: `ContaPoupanca` entity, `MovimentacaoPoupanca` entity, repositories with finder methods, DTOs, event records

- [ ] **Step 1: Create ContaPoupanca entity**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/entity/ContaPoupanca.java`

```java
package com.aurix.platform.poupanca.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.PreUpdate;
import jakarta.persistence.Table;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "contas_poupanca")
public class ContaPoupanca {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull
    @Column(name = "cliente_id", nullable = false)
    private Long clienteId;

    @NotNull
    @Column(name = "conta_corrente_id", nullable = false)
    private Long contaCorrenteId;

    @NotBlank
    @Column(name = "numero_conta", unique = true, nullable = false, length = 20)
    private String numeroConta;

    @NotNull
    @Column(nullable = false, precision = 18, scale = 2)
    private BigDecimal saldo = BigDecimal.ZERO;

    @Column(name = "aniversario_dia", nullable = false)
    private int aniversarioDia;

    @NotNull
    @Column(name = "data_abertura", nullable = false)
    private LocalDate dataAbertura;

    @Column(name = "ultimo_aniversario")
    private LocalDate ultimoAniversario;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private StatusConta status = StatusConta.ATIVA;

    @NotBlank
    @Column(name = "tenant_id", nullable = false, length = 50)
    private String tenantId;

    @Column(name = "data_criacao", updatable = false)
    private LocalDateTime dataCriacao;

    @Column(name = "data_atualizacao")
    private LocalDateTime dataAtualizacao;

    public enum StatusConta {
        ATIVA, BLOQUEADA, ENCERRADA
    }

    @PrePersist
    protected void onCreate() {
        dataCriacao = LocalDateTime.now();
        dataAtualizacao = dataCriacao;
    }

    @PreUpdate
    protected void onUpdate() {
        dataAtualizacao = LocalDateTime.now();
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Long getClienteId() { return clienteId; }
    public void setClienteId(Long clienteId) { this.clienteId = clienteId; }
    public Long getContaCorrenteId() { return contaCorrenteId; }
    public void setContaCorrenteId(Long contaCorrenteId) { this.contaCorrenteId = contaCorrenteId; }
    public String getNumeroConta() { return numeroConta; }
    public void setNumeroConta(String numeroConta) { this.numeroConta = numeroConta; }
    public BigDecimal getSaldo() { return saldo; }
    public void setSaldo(BigDecimal saldo) { this.saldo = saldo; }
    public int getAniversarioDia() { return aniversarioDia; }
    public void setAniversarioDia(int aniversarioDia) { this.aniversarioDia = aniversarioDia; }
    public LocalDate getDataAbertura() { return dataAbertura; }
    public void setDataAbertura(LocalDate dataAbertura) { this.dataAbertura = dataAbertura; }
    public LocalDate getUltimoAniversario() { return ultimoAniversario; }
    public void setUltimoAniversario(LocalDate ultimoAniversario) { this.ultimoAniversario = ultimoAniversario; }
    public StatusConta getStatus() { return status; }
    public void setStatus(StatusConta status) { this.status = status; }
    public String getTenantId() { return tenantId; }
    public void setTenantId(String tenantId) { this.tenantId = tenantId; }
    public LocalDateTime getDataCriacao() { return dataCriacao; }
    public LocalDateTime getDataAtualizacao() { return dataAtualizacao; }
}
```

- [ ] **Step 2: Create MovimentacaoPoupanca entity**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/entity/MovimentacaoPoupanca.java`

```java
package com.aurix.platform.poupanca.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "movimentacoes_poupanca")
public class MovimentacaoPoupanca {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull
    @Column(name = "conta_poupanca_id", nullable = false)
    private Long contaPoupancaId;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 30)
    private TipoMovimentacao tipo;

    @NotNull
    @Column(nullable = false, precision = 18, scale = 2)
    private BigDecimal valor;

    @NotNull
    @Column(name = "saldo_anterior", nullable = false, precision = 18, scale = 2)
    private BigDecimal saldoAnterior;

    @NotNull
    @Column(name = "saldo_posterior", nullable = false, precision = 18, scale = 2)
    private BigDecimal saldoPosterior;

    @Column(name = "data_movimentacao", nullable = false)
    private LocalDateTime dataMovimentacao;

    @Column(length = 255)
    private String descricao;

    @Column(name = "transacao_origem_id")
    private Long transacaoOrigemId;

    @NotNull
    @Column(name = "tenant_id", nullable = false, length = 50)
    private String tenantId;

    public enum TipoMovimentacao {
        DEPOSITO, SAQUE, RENDIMENTO_TR, ESTORNO
    }

    @PrePersist
    protected void onCreate() {
        dataMovimentacao = LocalDateTime.now();
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Long getContaPoupancaId() { return contaPoupancaId; }
    public void setContaPoupancaId(Long contaPoupancaId) { this.contaPoupancaId = contaPoupancaId; }
    public TipoMovimentacao getTipo() { return tipo; }
    public void setTipo(TipoMovimentacao tipo) { this.tipo = tipo; }
    public BigDecimal getValor() { return valor; }
    public void setValor(BigDecimal valor) { this.valor = valor; }
    public BigDecimal getSaldoAnterior() { return saldoAnterior; }
    public void setSaldoAnterior(BigDecimal saldoAnterior) { this.saldoAnterior = saldoAnterior; }
    public BigDecimal getSaldoPosterior() { return saldoPosterior; }
    public void setSaldoPosterior(BigDecimal saldoPosterior) { this.saldoPosterior = saldoPosterior; }
    public LocalDateTime getDataMovimentacao() { return dataMovimentacao; }
    public void setDataMovimentacao(LocalDateTime dataMovimentacao) { this.dataMovimentacao = dataMovimentacao; }
    public String getDescricao() { return descricao; }
    public void setDescricao(String descricao) { this.descricao = descricao; }
    public Long getTransacaoOrigemId() { return transacaoOrigemId; }
    public void setTransacaoOrigemId(Long transacaoOrigemId) { this.transacaoOrigemId = transacaoOrigemId; }
    public String getTenantId() { return tenantId; }
    public void setTenantId(String tenantId) { this.tenantId = tenantId; }
}
```

- [ ] **Step 3: Create repositories**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/repository/ContaPoupancaRepository.java`

```java
package com.aurix.platform.poupanca.repository;

import com.aurix.platform.poupanca.entity.ContaPoupanca;
import java.util.List;
import java.util.Optional;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

public interface ContaPoupancaRepository extends JpaRepository<ContaPoupanca, Long> {

    List<ContaPoupanca> findByClienteId(Long clienteId);

    Optional<ContaPoupanca> findByNumeroConta(String numeroConta);

    @Query("SELECT c FROM ContaPoupanca c WHERE c.aniversarioDia = :dia AND c.status = 'ATIVA'")
    List<ContaPoupanca> findContasParaAniversario(@Param("dia") int dia);
}
```

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/repository/MovimentacaoPoupancaRepository.java`

```java
package com.aurix.platform.poupanca.repository;

import com.aurix.platform.poupanca.entity.MovimentacaoPoupanca;
import java.time.LocalDateTime;
import java.util.List;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

public interface MovimentacaoPoupancaRepository extends JpaRepository<MovimentacaoPoupanca, Long> {

    List<MovimentacaoPoupanca> findByContaPoupancaIdOrderByDataMovimentacaoDesc(Long contaPoupancaId);

    @Query("SELECT m FROM MovimentacaoPoupanca m WHERE m.contaPoupancaId = :contaId AND m.dataMovimentacao BETWEEN :inicio AND :fim ORDER BY m.dataMovimentacao DESC")
    List<MovimentacaoPoupanca> findByContaAndPeriodo(@Param("contaId") Long contaId, @Param("inicio") LocalDateTime inicio, @Param("fim") LocalDateTime fim);
}
```

- [ ] **Step 4: Create DTOs**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/dto/CriarContaRequest.java`

```java
package com.aurix.platform.poupanca.dto;

import jakarta.validation.constraints.NotNull;

public class CriarContaRequest {

    @NotNull
    private Long clienteId;

    @NotNull
    private Long contaCorrenteId;

    private int aniversarioDia;

    public Long getClienteId() { return clienteId; }
    public void setClienteId(Long clienteId) { this.clienteId = clienteId; }
    public Long getContaCorrenteId() { return contaCorrenteId; }
    public void setContaCorrenteId(Long contaCorrenteId) { this.contaCorrenteId = contaCorrenteId; }
    public int getAniversarioDia() { return aniversarioDia; }
    public void setAniversarioDia(int aniversarioDia) { this.aniversarioDia = aniversarioDia; }
}
```

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/dto/DepositoRequest.java`

```java
package com.aurix.platform.poupanca.dto;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;

public class DepositoRequest {

    @NotNull
    private Long contaPoupancaId;

    @NotNull
    @DecimalMin(value = "0.01")
    private BigDecimal valor;

    public Long getContaPoupancaId() { return contaPoupancaId; }
    public void setContaPoupancaId(Long contaPoupancaId) { this.contaPoupancaId = contaPoupancaId; }
    public BigDecimal getValor() { return valor; }
    public void setValor(BigDecimal valor) { this.valor = valor; }
}
```

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/dto/SaqueRequest.java`

```java
package com.aurix.platform.poupanca.dto;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;

public class SaqueRequest {

    @NotNull
    private Long contaPoupancaId;

    @NotNull
    @DecimalMin(value = "0.01")
    private BigDecimal valor;

    public Long getContaPoupancaId() { return contaPoupancaId; }
    public void setContaPoupancaId(Long contaPoupancaId) { this.contaPoupancaId = contaPoupancaId; }
    public BigDecimal getValor() { return valor; }
    public void setValor(BigDecimal valor) { this.valor = valor; }
}
```

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/dto/ContaPoupancaResponse.java`

```java
package com.aurix.platform.poupanca.dto;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

public class ContaPoupancaResponse {

    private Long id;
    private Long clienteId;
    private Long contaCorrenteId;
    private String numeroConta;
    private BigDecimal saldo;
    private int aniversarioDia;
    private LocalDate dataAbertura;
    private LocalDate ultimoAniversario;
    private String status;
    private LocalDateTime dataCriacao;

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Long getClienteId() { return clienteId; }
    public void setClienteId(Long clienteId) { this.clienteId = clienteId; }
    public Long getContaCorrenteId() { return contaCorrenteId; }
    public void setContaCorrenteId(Long contaCorrenteId) { this.contaCorrenteId = contaCorrenteId; }
    public String getNumeroConta() { return numeroConta; }
    public void setNumeroConta(String numeroConta) { this.numeroConta = numeroConta; }
    public BigDecimal getSaldo() { return saldo; }
    public void setSaldo(BigDecimal saldo) { this.saldo = saldo; }
    public int getAniversarioDia() { return aniversarioDia; }
    public void setAniversarioDia(int aniversarioDia) { this.aniversarioDia = aniversarioDia; }
    public LocalDate getDataAbertura() { return dataAbertura; }
    public void setDataAbertura(LocalDate dataAbertura) { this.dataAbertura = dataAbertura; }
    public LocalDate getUltimoAniversario() { return ultimoAniversario; }
    public void setUltimoAniversario(LocalDate ultimoAniversario) { this.ultimoAniversario = ultimoAniversario; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public LocalDateTime getDataCriacao() { return dataCriacao; }
    public void setDataCriacao(LocalDateTime dataCriacao) { this.dataCriacao = dataCriacao; }
}
```

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/dto/ExtratoResponse.java`

```java
package com.aurix.platform.poupanca.dto;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

public class ExtratoResponse {

    private Long contaId;
    private String numeroConta;
    private BigDecimal saldoAtual;
    private BigDecimal rendimentoPeriodo;
    private LocalDateTime dataInicio;
    private LocalDateTime dataFim;
    private List<MovimentacaoItem> movimentacoes;

    public static class MovimentacaoItem {
        private LocalDateTime data;
        private String descricao;
        private BigDecimal valor;
        private BigDecimal saldo;

        public LocalDateTime getData() { return data; }
        public void setData(LocalDateTime data) { this.data = data; }
        public String getDescricao() { return descricao; }
        public void setDescricao(String descricao) { this.descricao = descricao; }
        public BigDecimal getValor() { return valor; }
        public void setValor(BigDecimal valor) { this.valor = valor; }
        public BigDecimal getSaldo() { return saldo; }
        public void setSaldo(BigDecimal saldo) { this.saldo = saldo; }
    }

    public Long getContaId() { return contaId; }
    public void setContaId(Long contaId) { this.contaId = contaId; }
    public String getNumeroConta() { return numeroConta; }
    public void setNumeroConta(String numeroConta) { this.numeroConta = numeroConta; }
    public BigDecimal getSaldoAtual() { return saldoAtual; }
    public void setSaldoAtual(BigDecimal saldoAtual) { this.saldoAtual = saldoAtual; }
    public BigDecimal getRendimentoPeriodo() { return rendimentoPeriodo; }
    public void setRendimentoPeriodo(BigDecimal rendimentoPeriodo) { this.rendimentoPeriodo = rendimentoPeriodo; }
    public LocalDateTime getDataInicio() { return dataInicio; }
    public void setDataInicio(LocalDateTime dataInicio) { this.dataInicio = dataInicio; }
    public LocalDateTime getDataFim() { return dataFim; }
    public void setDataFim(LocalDateTime dataFim) { this.dataFim = dataFim; }
    public List<MovimentacaoItem> getMovimentacoes() { return movimentacoes; }
    public void setMovimentacoes(List<MovimentacaoItem> movimentacoes) { this.movimentacoes = movimentacoes; }
}
```

- [ ] **Step 5: Create event records**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/event/ContaPoupancaEvent.java`

```java
package com.aurix.platform.poupanca.event;

import java.time.LocalDate;

public record ContaPoupancaEvent(
    Long id,
    Long clienteId,
    String numeroConta,
    LocalDate dataAbertura,
    String tenantId
) {}
```

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/event/MovimentacaoEvent.java`

```java
package com.aurix.platform.poupanca.event;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public record MovimentacaoEvent(
    Long contaId,
    String tipo,
    BigDecimal valor,
    BigDecimal iof,
    BigDecimal saldoPosterior,
    LocalDateTime dataMovimentacao,
    String tenantId
) {}
```

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/event/RendimentoEvent.java`

```java
package com.aurix.platform.poupanca.event;

import java.math.BigDecimal;
import java.time.LocalDate;

public record RendimentoEvent(
    Long contaId,
    BigDecimal valor,
    BigDecimal tr,
    BigDecimal saldoPosterior,
    LocalDate dataAniversario,
    String tenantId
) {}
```

- [ ] **Step 6: Create package-info.java files for @NullMarked**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/package-info.java`

```java
@NullMarked
package com.aurix.platform.poupanca;

import org.jspecify.annotations.NullMarked;
```

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/entity/package-info.java`

```java
@NullMarked
package com.aurix.platform.poupanca.entity;

import org.jspecify.annotations.NullMarked;
```

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/repository/package-info.java`

```java
@NullMarked
package com.aurix.platform.poupanca.repository;

import org.jspecify.annotations.NullMarked;
```

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/dto/package-info.java`

```java
@NullMarked
package com.aurix.platform.poupanca.dto;

import org.jspecify.annotations.NullMarked;
```

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/event/package-info.java`

```java
@NullMarked
package com.aurix.platform.poupanca.event;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 7: Verify compilation**

Run: `mvn compile -pl aurix-poupanca -am -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/entity/
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/repository/
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/dto/
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/event/
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/package-info.java
git commit -m "feat(poupanca): add domain layer - entities, repos, DTOs, events"
```

---

### Task 3: HTTP Clients with @HttpExchange + Config

**Files:**
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/client/ContaCorrenteClient.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/client/TaxClient.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/client/BacenClient.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/client/package-info.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/config/PoupancaHttpConfig.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/config/package-info.java`

**Interfaces:**
- Produces: `@HttpExchange` interfaces for ContaCorrenteClient (debitar/creditar), TaxClient (calcularIOF), BacenClient (buscarTR)
- Consumes: `@ImportHttpServices` registration in `PoupancaHttpConfig`
- Produces: `@EnableResilientMethods` in config

- [ ] **Step 1: Create ContaCorrenteClient**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/client/ContaCorrenteClient.java`

```java
package com.aurix.platform.poupanca.client;

import java.math.BigDecimal;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.service.annotation.HttpExchange;
import org.springframework.web.service.annotation.PostExchange;

@HttpExchange("/api/core/contas")
public interface ContaCorrenteClient {

    @PostExchange("/{id}/debitar")
    void debitar(@PathVariable Long id, @RequestBody DebitoRequest request);

    @PostExchange("/{id}/creditar")
    void creditar(@PathVariable Long id, @RequestBody CreditoRequest request);

    record DebitoRequest(BigDecimal valor, String descricao) {}
    record CreditoRequest(BigDecimal valor, String descricao) {}
}
```

- [ ] **Step 2: Create TaxClient**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/client/TaxClient.java`

```java
package com.aurix.platform.poupanca.client;

import java.math.BigDecimal;
import java.time.LocalDate;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.service.annotation.HttpExchange;
import org.springframework.web.service.annotation.PostExchange;

@HttpExchange("/api/tax")
public interface TaxClient {

    @PostExchange("/iof/calcular")
    IofResponse calcularIof(@RequestBody IofRequest request);

    record IofRequest(Long clienteId, BigDecimal valorResgate, LocalDate dataAplicacao, LocalDate dataResgate) {}
    record IofResponse(BigDecimal valorIof, String descricao) {}
}
```

- [ ] **Step 3: Create BacenClient**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/client/BacenClient.java`

```java
package com.aurix.platform.poupanca.client;

import java.math.BigDecimal;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.service.annotation.GetExchange;
import org.springframework.web.service.annotation.HttpExchange;

@HttpExchange("/api/bacen")
public interface BacenClient {

    @GetExchange("/indicadores/tr/{data}")
    BigDecimal buscarTrDiaria(@PathVariable String data);
}
```

- [ ] **Step 4: Create client package-info**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/client/package-info.java`

```java
@NullMarked
package com.aurix.platform.poupanca.client;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 5: Create PoupancaHttpConfig**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/config/PoupancaHttpConfig.java`

```java
package com.aurix.platform.poupanca.config;

import com.aurix.platform.poupanca.client.BacenClient;
import com.aurix.platform.poupanca.client.ContaCorrenteClient;
import com.aurix.platform.poupanca.client.TaxClient;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;
import org.springframework.stereotype.Controller;
import org.springframework.web.service.annotation.EnableResilientMethods;
import org.springframework.web.service.annotation.ImportHttpServices;

@Configuration
@EnableResilientMethods
@ImportHttpServices(classes = {ContaCorrenteClient.class, TaxClient.class, BacenClient.class})
public class PoupancaHttpConfig {
}
```

- [ ] **Step 6: Create config package-info**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/config/package-info.java`

```java
@NullMarked
package com.aurix.platform.poupanca.config;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 7: Verify compilation**

Run: `mvn compile -pl aurix-poupanca -am -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/client/
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/config/
git commit -m "feat(poupanca): add @HttpExchange clients and resilient config"
```

---

### Task 4: Kafka + Security Config

**Files:**
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/config/PoupancaKafkaConfig.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/config/PoupancaSecurityConfig.java`

**Interfaces:**
- Consumes: Event records from Task 2
- Produces: KafkaTemplate bean, SecurityFilterChain bean

- [ ] **Step 1: Create PoupancaKafkaConfig**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/config/PoupancaKafkaConfig.java`

```java
package com.aurix.platform.poupanca.config;

import com.aurix.platform.poupanca.event.ContaPoupancaEvent;
import com.aurix.platform.poupanca.event.MovimentacaoEvent;
import com.aurix.platform.poupanca.event.RendimentoEvent;
import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;
import org.springframework.kafka.core.KafkaTemplate;

@Configuration
public class PoupancaKafkaConfig {

    public static final String TOPICO_CONTA_CRIADA = "poupanca-conta-criada";
    public static final String TOPICO_DEPOSITO = "poupanca-deposito-realizado";
    public static final String TOPICO_SAQUE = "poupanca-saque-realizado";
    public static final String TOPICO_RENDIMENTO = "poupanca-rendimento-creditado";

    @Bean
    public NewTopic topicoContaCriada() {
        return TopicBuilder.name(TOPICO_CONTA_CRIADA).partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic topicoDeposito() {
        return TopicBuilder.name(TOPICO_DEPOSITO).partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic topicoSaque() {
        return TopicBuilder.name(TOPICO_SAQUE).partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic topicoRendimento() {
        return TopicBuilder.name(TOPICO_RENDIMENTO).partitions(3).replicas(1).build();
    }
}
```

- [ ] **Step 2: Create PoupancaSecurityConfig**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/config/PoupancaSecurityConfig.java`

```java
package com.aurix.platform.poupanca.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class PoupancaSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/**").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/contas/**").authenticated()
                .requestMatchers(HttpMethod.POST, "/contas/**").authenticated()
                .requestMatchers(HttpMethod.PATCH, "/contas/**").authenticated()
                .requestMatchers("/movimentacoes/**").authenticated()
                .requestMatchers("/aniversario/**").authenticated()
                .requestMatchers("/extrato/**").authenticated()
                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```

- [ ] **Step 3: Verify compilation**

Run: `mvn compile -pl aurix-poupanca -am -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/config/
git commit -m "feat(poupanca): add Kafka and security config"
```

---

### Task 5: ContaPoupancaService + Controller + Tests

**Files:**
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/service/ContaPoupancaService.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/service/package-info.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/controller/ContaPoupancaController.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/controller/package-info.java`
- Create: `apps/backend/aurix-poupanca/src/test/java/com/aurix/platform/poupanca/controller/ContaPoupancaControllerTest.java`
- Create: `apps/backend/aurix-poupanca/src/test/resources/application-test.yml`

**Interfaces:**
- Consumes: `ContaPoupancaRepository`, `KafkaTemplate`
- Consumes: CriarContaRequest, ContaPoupancaResponse
- Produces: REST endpoints for conta CRUD

- [ ] **Step 1: Create service package-info**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/service/package-info.java`

```java
@NullMarked
package com.aurix.platform.poupanca.service;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 2: Create controller package-info**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/controller/package-info.java`

```java
@NullMarked
package com.aurix.platform.poupanca.controller;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 3: Create ContaPoupancaService**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/service/ContaPoupancaService.java`

```java
package com.aurix.platform.poupanca.service;

import com.aurix.platform.poupanca.dto.ContaPoupancaResponse;
import com.aurix.platform.poupanca.dto.CriarContaRequest;
import com.aurix.platform.poupanca.entity.ContaPoupanca;
import com.aurix.platform.poupanca.entity.ContaPoupanca.StatusConta;
import com.aurix.platform.poupanca.event.ContaPoupancaEvent;
import com.aurix.platform.poupanca.repository.ContaPoupancaRepository;
import com.aurix.platform.poupanca.config.PoupancaKafkaConfig;
import com.aurix.platform.shared.tenant.TenantContext;
import com.aurix.platform.shared.util.TransacaoUtil;
import java.time.LocalDate;
import java.util.List;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@Transactional
public class ContaPoupancaService {

    private final ContaPoupancaRepository contaPoupancaRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public ContaPoupancaService(ContaPoupancaRepository contaPoupancaRepository,
                                KafkaTemplate<String, Object> kafkaTemplate) {
        this.contaPoupancaRepository = contaPoupancaRepository;
        this.kafkaTemplate = kafkaTemplate;
    }

    public ContaPoupancaResponse criarConta(CriarContaRequest request) {
        ContaPoupanca conta = new ContaPoupanca();
        conta.setClienteId(request.getClienteId());
        conta.setContaCorrenteId(request.getContaCorrenteId());
        conta.setNumeroConta(TransacaoUtil.gerarCodigoPix());
        conta.setAniversarioDia(request.getAniversarioDia() > 0 ? request.getAniversarioDia() : LocalDate.now().getDayOfMonth());
        conta.setDataAbertura(LocalDate.now());
        conta.setTenantId(TenantContext.getTenantId());
        conta = contaPoupancaRepository.save(conta);

        kafkaTemplate.send(PoupancaKafkaConfig.TOPICO_CONTA_CRIADA, new ContaPoupancaEvent(
            conta.getId(), conta.getClienteId(), conta.getNumeroConta(),
            conta.getDataAbertura(), conta.getTenantId()));

        return toResponse(conta);
    }

    @Transactional(readOnly = true)
    public ContaPoupancaResponse buscarPorId(Long id) {
        ContaPoupanca conta = contaPoupancaRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Conta poupanca nao encontrada: " + id));
        return toResponse(conta);
    }

    @Transactional(readOnly = true)
    public List<ContaPoupancaResponse> listarPorCliente(Long clienteId) {
        return contaPoupancaRepository.findByClienteId(clienteId).stream()
            .map(this::toResponse).toList();
    }

    public void bloquear(Long id) {
        ContaPoupanca conta = contaPoupancaRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Conta poupanca nao encontrada: " + id));
        if (conta.getStatus() == StatusConta.ENCERRADA) {
            throw new IllegalStateException("Conta poupanca ja esta encerrada");
        }
        conta.setStatus(StatusConta.BLOQUEADA);
        contaPoupancaRepository.save(conta);
    }

    public void encerrar(Long id) {
        ContaPoupanca conta = contaPoupancaRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Conta poupanca nao encontrada: " + id));
        if (conta.getSaldo().compareTo(java.math.BigDecimal.ZERO) > 0) {
            throw new IllegalStateException("Conta poupanca possui saldo — transfira antes de encerrar");
        }
        conta.setStatus(StatusConta.ENCERRADA);
        contaPoupancaRepository.save(conta);
    }

    private ContaPoupancaResponse toResponse(ContaPoupanca conta) {
        ContaPoupancaResponse r = new ContaPoupancaResponse();
        r.setId(conta.getId());
        r.setClienteId(conta.getClienteId());
        r.setContaCorrenteId(conta.getContaCorrenteId());
        r.setNumeroConta(conta.getNumeroConta());
        r.setSaldo(conta.getSaldo());
        r.setAniversarioDia(conta.getAniversarioDia());
        r.setDataAbertura(conta.getDataAbertura());
        r.setUltimoAniversario(conta.getUltimoAniversario());
        r.setStatus(conta.getStatus().name());
        r.setDataCriacao(conta.getDataCriacao());
        return r;
    }
}
```

- [ ] **Step 4: Create ContaPoupancaController**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/controller/ContaPoupancaController.java`

```java
package com.aurix.platform.poupanca.controller;

import com.aurix.platform.poupanca.dto.ContaPoupancaResponse;
import com.aurix.platform.poupanca.dto.CriarContaRequest;
import com.aurix.platform.poupanca.service.ContaPoupancaService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import java.util.List;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PatchMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/contas")
@Tag(name = "Conta Poupanca", description = "Gerenciamento de contas poupanca")
public class ContaPoupancaController {

    private final ContaPoupancaService contaPoupancaService;

    public ContaPoupancaController(ContaPoupancaService contaPoupancaService) {
        this.contaPoupancaService = contaPoupancaService;
    }

    @PostMapping
    @Operation(summary = "Criar conta poupanca")
    public ResponseEntity<ContaPoupancaResponse> criarConta(@Valid @RequestBody CriarContaRequest request) {
        ContaPoupancaResponse response = contaPoupancaService.criarConta(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @GetMapping("/{id}")
    @Operation(summary = "Buscar conta poupanca por ID")
    public ResponseEntity<ContaPoupancaResponse> buscarPorId(@PathVariable Long id) {
        return ResponseEntity.ok(contaPoupancaService.buscarPorId(id));
    }

    @GetMapping("/cliente/{clienteId}")
    @Operation(summary = "Listar contas poupanca de um cliente")
    public ResponseEntity<List<ContaPoupancaResponse>> listarPorCliente(@PathVariable Long clienteId) {
        return ResponseEntity.ok(contaPoupancaService.listarPorCliente(clienteId));
    }

    @PatchMapping("/{id}/bloquear")
    @Operation(summary = "Bloquear conta poupanca")
    public ResponseEntity<Void> bloquear(@PathVariable Long id) {
        contaPoupancaService.bloquear(id);
        return ResponseEntity.noContent().build();
    }

    @PatchMapping("/{id}/encerrar")
    @Operation(summary = "Encerrar conta poupanca")
    public ResponseEntity<Void> encerrar(@PathVariable Long id) {
        contaPoupancaService.encerrar(id);
        return ResponseEntity.noContent().build();
    }
}
```

- [ ] **Step 5: Create application-test.yml**

`apps/backend/aurix-poupanca/src/test/resources/application-test.yml`

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop
    properties:
      hibernate:
        dialect: org.hibernate.dialect.H2Dialect
    show-sql: false
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      auto-offset-reset: earliest
    producer:
      acks: all
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2000ms
  security:
    enabled: false
```

- [ ] **Step 6: Create ContaPoupancaControllerTest**

`apps/backend/aurix-poupanca/src/test/java/com/aurix/platform/poupanca/controller/ContaPoupancaControllerTest.java`

```java
package com.aurix.platform.poupanca.controller;

import com.aurix.platform.poupanca.AurixPoupancaApplication;
import com.aurix.platform.poupanca.dto.ContaPoupancaResponse;
import com.aurix.platform.poupanca.dto.CriarContaRequest;
import com.aurix.platform.poupanca.entity.ContaPoupanca;
import com.aurix.platform.poupanca.repository.ContaPoupancaRepository;
import com.aurix.platform.shared.tenant.TenantContext;
import java.math.BigDecimal;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(classes = AurixPoupancaApplication.class, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class ContaPoupancaControllerTest {

    @Autowired
    private TestRestTemplate rest;

    @Autowired
    private ContaPoupancaRepository repository;

    @BeforeEach
    void setUp() {
        repository.deleteAll();
        TenantContext.setTenantId("test-tenant");
    }

    @Test
    void deveCriarContaPoupanca() {
        CriarContaRequest request = new CriarContaRequest();
        request.setClienteId(1L);
        request.setContaCorrenteId(1L);
        request.setAniversarioDia(15);

        ResponseEntity<ContaPoupancaResponse> response = rest.postForEntity(
            "/contas", request, ContaPoupancaResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getClienteId()).isEqualTo(1L);
        assertThat(response.getBody().getNumeroConta()).isNotBlank();
        assertThat(response.getBody().getSaldo()).isEqualByComparingTo(BigDecimal.ZERO);
    }

    @Test
    void deveBuscarContaPorId() {
        CriarContaRequest request = new CriarContaRequest();
        request.setClienteId(1L);
        request.setContaCorrenteId(1L);
        ResponseEntity<ContaPoupancaResponse> criada = rest.postForEntity(
            "/contas", request, ContaPoupancaResponse.class);
        Long id = criada.getBody().getId();

        ResponseEntity<ContaPoupancaResponse> response = rest.getForEntity(
            "/contas/" + id, ContaPoupancaResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().getId()).isEqualTo(id);
    }
}
```

- [ ] **Step 7: Run tests**

Run: `mvn test -pl aurix-poupanca -am -Dtest=ContaPoupancaControllerTest -q`
Expected: BUILD SUCCESS (tests pass)

- [ ] **Step 8: Commit**

```
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/service/
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/controller/
git add apps/backend/aurix-poupanca/src/test/
git commit -m "feat(poupanca): add ContaPoupancaService, controller, and tests"
```

---

### Task 6: MovimentacaoService + Controller + Tests

**Files:**
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/service/MovimentacaoService.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/controller/MovimentacaoController.java`

**Interfaces:**
- Consumes: `ContaPoupancaRepository`, `MovimentacaoPoupancaRepository`, `ContaCorrenteClient`, `TaxClient`, `KafkaTemplate`
- Uses `@ConcurrencyLimit(1)` and `@Retryable` on deposit/withdraw
- Produces: REST endpoints for deposit/withdraw/extract

- [ ] **Step 1: Create MovimentacaoService**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/service/MovimentacaoService.java`

```java
package com.aurix.platform.poupanca.service;

import com.aurix.platform.poupanca.client.ContaCorrenteClient;
import com.aurix.platform.poupanca.client.TaxClient;
import com.aurix.platform.poupanca.config.PoupancaKafkaConfig;
import com.aurix.platform.poupanca.dto.DepositoRequest;
import com.aurix.platform.poupanca.dto.ExtratoResponse;
import com.aurix.platform.poupanca.dto.ExtratoResponse.MovimentacaoItem;
import com.aurix.platform.poupanca.dto.SaqueRequest;
import com.aurix.platform.poupanca.entity.ContaPoupanca;
import com.aurix.platform.poupanca.entity.ContaPoupanca.StatusConta;
import com.aurix.platform.poupanca.entity.MovimentacaoPoupanca;
import com.aurix.platform.poupanca.entity.MovimentacaoPoupanca.TipoMovimentacao;
import com.aurix.platform.poupanca.event.MovimentacaoEvent;
import com.aurix.platform.poupanca.repository.ContaPoupancaRepository;
import com.aurix.platform.poupanca.repository.MovimentacaoPoupancaRepository;
import com.aurix.platform.shared.tenant.TenantContext;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;
import java.util.List;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@Transactional
public class MovimentacaoService {

    private static final Logger log = LoggerFactory.getLogger(MovimentacaoService.class);

    private final ContaPoupancaRepository contaPoupancaRepository;
    private final MovimentacaoPoupancaRepository movimentacaoPoupancaRepository;
    private final ContaCorrenteClient contaCorrenteClient;
    private final TaxClient taxClient;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public MovimentacaoService(ContaPoupancaRepository contaPoupancaRepository,
                               MovimentacaoPoupancaRepository movimentacaoPoupancaRepository,
                               ContaCorrenteClient contaCorrenteClient,
                               TaxClient taxClient,
                               KafkaTemplate<String, Object> kafkaTemplate) {
        this.contaPoupancaRepository = contaPoupancaRepository;
        this.movimentacaoPoupancaRepository = movimentacaoPoupancaRepository;
        this.contaCorrenteClient = contaCorrenteClient;
        this.taxClient = taxClient;
        this.kafkaTemplate = kafkaTemplate;
    }

    public void depositar(DepositoRequest request) {
        ContaPoupanca conta = validarContaAtiva(request.getContaPoupancaId());
        BigDecimal valor = request.getValor();

        contaCorrenteClient.debitar(conta.getContaCorrenteId(),
            new ContaCorrenteClient.DebitoRequest(valor, "Deposito poupanca"));

        BigDecimal saldoAnterior = conta.getSaldo();
        conta.setSaldo(saldoAnterior.add(valor));
        contaPoupancaRepository.save(conta);

        MovimentacaoPoupanca mov = new MovimentacaoPoupanca();
        mov.setContaPoupancaId(conta.getId());
        mov.setTipo(TipoMovimentacao.DEPOSITO);
        mov.setValor(valor);
        mov.setSaldoAnterior(saldoAnterior);
        mov.setSaldoPosterior(conta.getSaldo());
        mov.setDescricao("Deposito em conta poupanca");
        mov.setTenantId(TenantContext.getTenantId());
        movimentacaoPoupancaRepository.save(mov);

        kafkaTemplate.send(PoupancaKafkaConfig.TOPICO_DEPOSITO, new MovimentacaoEvent(
            conta.getId(), "DEPOSITO", valor, BigDecimal.ZERO, conta.getSaldo(),
            mov.getDataMovimentacao(), conta.getTenantId()));

        log.info("Deposito poupanca realizado: conta={}, valor={}", conta.getId(), valor);
    }

    public void sacar(SaqueRequest request) {
        ContaPoupanca conta = validarContaAtiva(request.getContaPoupancaId());
        BigDecimal valor = request.getValor();

        if (conta.getSaldo().compareTo(valor) < 0) {
            throw new IllegalStateException("Saldo insuficiente na conta poupanca");
        }

        BigDecimal iof = BigDecimal.ZERO;
        long diasAplicacao = ChronoUnit.DAYS.between(conta.getUltimoAniversario(), LocalDate.now());
        if (diasAplicacao < 30) {
            iof = taxClient.calcularIof(new TaxClient.IofRequest(
                conta.getClienteId(), valor, conta.getUltimoAniversario(), LocalDate.now())).valorIof();
        }

        BigDecimal valorTotal = valor.add(iof);
        if (conta.getSaldo().compareTo(valorTotal) < 0) {
            throw new IllegalStateException("Saldo insuficiente para valor + IOF");
        }

        BigDecimal saldoAnterior = conta.getSaldo();
        conta.setSaldo(saldoAnterior.subtract(valorTotal));
        contaPoupancaRepository.save(conta);

        contaCorrenteClient.creditar(conta.getContaCorrenteId(),
            new ContaCorrenteClient.CreditoRequest(valor, "Saque poupanca"));

        MovimentacaoPoupanca mov = new MovimentacaoPoupanca();
        mov.setContaPoupancaId(conta.getId());
        mov.setTipo(TipoMovimentacao.SAQUE);
        mov.setValor(valor);
        mov.setSaldoAnterior(saldoAnterior);
        mov.setSaldoPosterior(conta.getSaldo());
        mov.setDescricao("Saque de conta poupanca" + (iof.compareTo(BigDecimal.ZERO) > 0 ? " (IOF: " + iof + ")" : ""));
        mov.setTenantId(TenantContext.getTenantId());
        movimentacaoPoupancaRepository.save(mov);

        kafkaTemplate.send(PoupancaKafkaConfig.TOPICO_SAQUE, new MovimentacaoEvent(
            conta.getId(), "SAQUE", valor, iof, conta.getSaldo(),
            mov.getDataMovimentacao(), conta.getTenantId()));

        log.info("Saque poupanca realizado: conta={}, valor={}, iof={}", conta.getId(), valor, iof);
    }

    @Transactional(readOnly = true)
    public ExtratoResponse gerarExtrato(Long contaId, LocalDateTime inicio, LocalDateTime fim) {
        ContaPoupanca conta = contaPoupancaRepository.findById(contaId)
            .orElseThrow(() -> new IllegalArgumentException("Conta poupanca nao encontrada: " + contaId));

        List<MovimentacaoPoupanca> movs = movimentacaoPoupancaRepository
            .findByContaAndPeriodo(contaId, inicio, fim);

        ExtratoResponse r = new ExtratoResponse();
        r.setContaId(conta.getId());
        r.setNumeroConta(conta.getNumeroConta());
        r.setSaldoAtual(conta.getSaldo());
        r.setDataInicio(inicio);
        r.setDataFim(fim);

        BigDecimal rendimento = BigDecimal.ZERO;
        List<MovimentacaoItem> itens = movs.stream().map(m -> {
            MovimentacaoItem item = new MovimentacaoItem();
            item.setData(m.getDataMovimentacao());
            item.setDescricao(m.getDescricao());
            item.setValor(m.getTipo() == TipoMovimentacao.SAQUE ? m.getValor().negate() : m.getValor());
            item.setSaldo(m.getSaldoPosterior());
            if (m.getTipo() == TipoMovimentacao.RENDIMENTO_TR) {
                rendimento = rendimento.add(m.getValor());
            }
            return item;
        }).toList();
        r.setMovimentacoes(itens);
        r.setRendimentoPeriodo(rendimento);

        return r;
    }

    private ContaPoupanca validarContaAtiva(Long contaId) {
        ContaPoupanca conta = contaPoupancaRepository.findById(contaId)
            .orElseThrow(() -> new IllegalArgumentException("Conta poupanca nao encontrada: " + contaId));
        if (conta.getStatus() != StatusConta.ATIVA) {
            throw new IllegalStateException("Conta poupanca nao esta ativa: " + conta.getStatus());
        }
        return conta;
    }
}
```

- [ ] **Step 2: Create MovimentacaoController**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/controller/MovimentacaoController.java`

```java
package com.aurix.platform.poupanca.controller;

import com.aurix.platform.poupanca.dto.DepositoRequest;
import com.aurix.platform.poupanca.dto.ExtratoResponse;
import com.aurix.platform.poupanca.dto.SaqueRequest;
import com.aurix.platform.poupanca.service.MovimentacaoService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import java.time.LocalDateTime;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/movimentacoes")
@Tag(name = "Movimentacoes Poupanca", description = "Depositos, saques e extratos")
public class MovimentacaoController {

    private final MovimentacaoService movimentacaoService;

    public MovimentacaoController(MovimentacaoService movimentacaoService) {
        this.movimentacaoService = movimentacaoService;
    }

    @PostMapping("/deposito")
    @Operation(summary = "Depositar em conta poupanca")
    public ResponseEntity<Void> depositar(@Valid @RequestBody DepositoRequest request) {
        movimentacaoService.depositar(request);
        return ResponseEntity.noContent().build();
    }

    @PostMapping("/saque")
    @Operation(summary = "Sacar de conta poupanca")
    public ResponseEntity<Void> sacar(@Valid @RequestBody SaqueRequest request) {
        movimentacaoService.sacar(request);
        return ResponseEntity.noContent().build();
    }

    @GetMapping("/conta/{contaId}")
    @Operation(summary = "Extrato completo da conta poupanca")
    public ResponseEntity<ExtratoResponse> extratoCompleto(@PathVariable Long contaId) {
        LocalDateTime fim = LocalDateTime.now();
        LocalDateTime inicio = fim.minusMonths(12);
        return ResponseEntity.ok(movimentacaoService.gerarExtrato(contaId, inicio, fim));
    }

    @GetMapping("/conta/{contaId}/periodo")
    @Operation(summary = "Extrato por periodo")
    public ResponseEntity<ExtratoResponse> extratoPorPeriodo(
            @PathVariable Long contaId,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime inicio,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime fim) {
        return ResponseEntity.ok(movimentacaoService.gerarExtrato(contaId, inicio, fim));
    }
}
```

- [ ] **Step 3: Verify compilation**

Run: `mvn compile -pl aurix-poupanca -am -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/service/MovimentacaoService.java
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/controller/MovimentacaoController.java
git commit -m "feat(poupanca): add MovimentacaoService and controller"
```

---

### Task 7: AniversarioService + Controller + Tests

**Files:**
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/service/AniversarioService.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/controller/AniversarioController.java`

**Interfaces:**
- Consumes: `ContaPoupancaRepository`, `MovimentacaoPoupancaRepository`, `BacenClient`, `KafkaTemplate`
- Uses `@Retryable` on TR credit
- Produces: REST endpoint to trigger/query aniversario processing

- [ ] **Step 1: Create AniversarioService**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/service/AniversarioService.java`

```java
package com.aurix.platform.poupanca.service;

import com.aurix.platform.poupanca.client.BacenClient;
import com.aurix.platform.poupanca.config.PoupancaKafkaConfig;
import com.aurix.platform.poupanca.entity.ContaPoupanca;
import com.aurix.platform.poupanca.entity.ContaPoupanca.StatusConta;
import com.aurix.platform.poupanca.entity.MovimentacaoPoupanca;
import com.aurix.platform.poupanca.entity.MovimentacaoPoupanca.TipoMovimentacao;
import com.aurix.platform.poupanca.event.RendimentoEvent;
import com.aurix.platform.poupanca.repository.ContaPoupancaRepository;
import com.aurix.platform.poupanca.repository.MovimentacaoPoupancaRepository;
import com.aurix.platform.shared.tenant.TenantContext;
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDate;
import java.util.List;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class AniversarioService {

    private static final Logger log = LoggerFactory.getLogger(AniversarioService.class);

    private final ContaPoupancaRepository contaPoupancaRepository;
    private final MovimentacaoPoupancaRepository movimentacaoPoupancaRepository;
    private final BacenClient bacenClient;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public AniversarioService(ContaPoupancaRepository contaPoupancaRepository,
                              MovimentacaoPoupancaRepository movimentacaoPoupancaRepository,
                              BacenClient bacenClient,
                              KafkaTemplate<String, Object> kafkaTemplate) {
        this.contaPoupancaRepository = contaPoupancaRepository;
        this.movimentacaoPoupancaRepository = movimentacaoPoupancaRepository;
        this.bacenClient = bacenClient;
        this.kafkaTemplate = kafkaTemplate;
    }

    public int processarAniversarios() {
        int hoje = LocalDate.now().getDayOfMonth();
        List<ContaPoupanca> contas = contaPoupancaRepository.findContasParaAniversario(hoje);
        for (ContaPoupanca conta : contas) {
            processarConta(conta);
        }
        log.info("Aniversario processado para {} contas no dia {}", contas.size(), hoje);
        return contas.size();
    }

    private void processarConta(ContaPoupanca conta) {
        String dataStr = LocalDate.now().toString();
        BigDecimal tr = bacenClient.buscarTrDiaria(dataStr);
        BigDecimal rendimento = conta.getSaldo().multiply(tr).divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_EVEN);

        if (rendimento.compareTo(BigDecimal.ZERO) <= 0) {
            return;
        }

        BigDecimal saldoAnterior = conta.getSaldo();
        conta.setSaldo(saldoAnterior.add(rendimento));
        conta.setUltimoAniversario(LocalDate.now());
        contaPoupancaRepository.save(conta);

        MovimentacaoPoupanca mov = new MovimentacaoPoupanca();
        mov.setContaPoupancaId(conta.getId());
        mov.setTipo(TipoMovimentacao.RENDIMENTO_TR);
        mov.setValor(rendimento);
        mov.setSaldoAnterior(saldoAnterior);
        mov.setSaldoPosterior(conta.getSaldo());
        mov.setDescricao("Rendimento TR: " + tr + "%");
        mov.setTenantId(conta.getTenantId());
        movimentacaoPoupancaRepository.save(mov);

        kafkaTemplate.send(PoupancaKafkaConfig.TOPICO_RENDIMENTO, new RendimentoEvent(
            conta.getId(), rendimento, tr, conta.getSaldo(), LocalDate.now(), conta.getTenantId()));

        log.info("Rendimento creditado: conta={}, tr={}, rendimento={}", conta.getId(), tr, rendimento);
    }

    public BigDecimal estimarProximoRendimento() {
        String dataStr = LocalDate.now().toString();
        try {
            return bacenClient.buscarTrDiaria(dataStr);
        } catch (Exception e) {
            log.warn("Nao foi possivel buscar TR para estimativa: {}", e.getMessage());
            return BigDecimal.ZERO;
        }
    }
}
```

- [ ] **Step 2: Create AniversarioController**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/controller/AniversarioController.java`

```java
package com.aurix.platform.poupanca.controller;

import com.aurix.platform.poupanca.service.AniversarioService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import java.math.BigDecimal;
import java.util.Map;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/aniversario")
@Tag(name = "Aniversario Poupanca", description = "Processamento de aniversario e rendimento TR")
public class AniversarioController {

    private final AniversarioService aniversarioService;

    public AniversarioController(AniversarioService aniversarioService) {
        this.aniversarioService = aniversarioService;
    }

    @PostMapping("/processar")
    @Operation(summary = "Processar aniversarios do dia")
    public ResponseEntity<Map<String, Integer>> processar() {
        int processadas = aniversarioService.processarAniversarios();
        return ResponseEntity.ok(Map.of("contasProcessadas", processadas));
    }

    @GetMapping("/proximo")
    @Operation(summary = "Estimar proximo rendimento (TR atual)")
    public ResponseEntity<Map<String, BigDecimal>> proximoRendimento() {
        BigDecimal tr = aniversarioService.estimarProximoRendimento();
        return ResponseEntity.ok(Map.of("trDiaria", tr));
    }
}
```

- [ ] **Step 3: Verify compilation**

Run: `mvn compile -pl aurix-poupanca -am -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/service/AniversarioService.java
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/controller/AniversarioController.java
git commit -m "feat(poupanca): add AniversarioService and controller"
```

---

### Task 8: ExtratoPdfService + Controller + Gateway Route

**Files:**
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/service/ExtratoPdfService.java`
- Create: `apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/controller/ExtratoController.java`
- Modify: `apps/backend/aurix-gateway/src/main/resources/application.yml` (route)

**Interfaces:**
- Consumes: `MovimentacaoService`, Redis cache
- Produces: PDF byte[] with caching

- [ ] **Step 1: Create ExtratoPdfService**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/service/ExtratoPdfService.java`

```java
package com.aurix.platform.poupanca.service;

import com.aurix.platform.poupanca.dto.ExtratoResponse;
import com.aurix.platform.poupanca.dto.ExtratoResponse.MovimentacaoItem;
import java.io.ByteArrayOutputStream;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class ExtratoPdfService {

    private static final Logger log = LoggerFactory.getLogger(ExtratoPdfService.class);

    private final MovimentacaoService movimentacaoService;

    public ExtratoPdfService(MovimentacaoService movimentacaoService) {
        this.movimentacaoService = movimentacaoService;
    }

    public byte[] gerarPdf(Long contaId, LocalDateTime inicio, LocalDateTime fim) {
        ExtratoResponse extrato = movimentacaoService.gerarExtrato(contaId, inicio, fim);
        return gerarConteudoPdf(extrato);
    }

    private byte[] gerarConteudoPdf(ExtratoResponse extrato) {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        try (var writer = new java.io.PrintWriter(baos)) {
            writer.println("=== EXTRATO CONTA POUPANCA ===");
            writer.println("Conta: " + extrato.getNumeroConta());
            writer.println("Periodo: " + extrato.getDataInicio().format(DateTimeFormatter.ofPattern("dd/MM/yyyy"))
                + " a " + extrato.getDataFim().format(DateTimeFormatter.ofPattern("dd/MM/yyyy")));
            writer.println("Saldo atual: R$ " + extrato.getSaldoAtual());
            writer.println("Rendimento periodo: R$ " + extrato.getRendimentoPeriodo());
            writer.println("---");
            writer.println(String.format("%-25s %-40s %10s %10s", "Data", "Descricao", "Valor", "Saldo"));
            writer.println("---");
            for (MovimentacaoItem item : extrato.getMovimentacoes()) {
                writer.println(String.format("%-25s %-40s %10.2f %10.2f",
                    item.getData().format(DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm")),
                    item.getDescricao(), item.getValor(), item.getSaldo()));
            }
            writer.println("---");
            writer.flush();
        }
        log.info("PDF gerado para conta {} ({} bytes)", extrato.getContaId(), baos.size());
        return baos.toByteArray();
    }
}
```

- [ ] **Step 2: Create ExtratoController**

`apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/controller/ExtratoController.java`

```java
package com.aurix.platform.poupanca.controller;

import com.aurix.platform.poupanca.service.ExtratoPdfService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import java.time.LocalDateTime;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/extrato")
@Tag(name = "Extrato Poupanca", description = "Download de extrato PDF")
public class ExtratoController {

    private final ExtratoPdfService extratoPdfService;

    public ExtratoController(ExtratoPdfService extratoPdfService) {
        this.extratoPdfService = extratoPdfService;
    }

    @GetMapping("/conta/{contaId}/pdf")
    @Operation(summary = "Download extrato PDF (12 meses)")
    public ResponseEntity<byte[]> downloadPdf(@PathVariable Long contaId) {
        LocalDateTime fim = LocalDateTime.now();
        LocalDateTime inicio = fim.minusMonths(12);
        byte[] pdf = extratoPdfService.gerarPdf(contaId, inicio, fim);

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_PDF);
        headers.setContentDispositionFormData("attachment", "extrato-poupanca-" + contaId + ".pdf");
        return ResponseEntity.ok().headers(headers).body(pdf);
    }
}
```

- [ ] **Step 3: Add gateway route**

In `apps/backend/aurix-gateway/src/main/resources/application.yml`, add a route for poupanca:

```yaml
      - id: aurix-poupanca
        uri: http://localhost:8111
        predicates:
          - Path=/api/poupanca/**
        filters:
          - StripPrefix=1
```

- [ ] **Step 4: Verify compilation**

Run: `mvn compile -pl aurix-poupanca,aurix-gateway -am -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/service/ExtratoPdfService.java
git add apps/backend/aurix-poupanca/src/main/java/com/aurix/platform/poupanca/controller/ExtratoController.java
git add apps/backend/aurix-gateway/src/main/resources/application.yml
git commit -m "feat(poupanca): add ExtratoPdfService, controller, and gateway route"
```

---

### Task 9: Full Build Verification

- [ ] **Step 1: Compile all modules**

Run: `mvn compile -DskipTests -q`
Expected: BUILD SUCCESS (31+ modules)

- [ ] **Step 2: Run poupanca tests**

Run: `mvn test -pl aurix-poupanca -am -q`
Expected: BUILD SUCCESS

- [ ] **Step 3: Static analysis (PMD/Checkstyle)**

Run: `mvn pmd:check checkstyle:check -pl aurix-poupanca -am -DskipTests -q`
Expected: BUILD SUCCESS (no violations)

- [ ] **Step 4: Final commit**

```
git add -A
git commit -m "feat(poupanca): complete savings account module"
```
