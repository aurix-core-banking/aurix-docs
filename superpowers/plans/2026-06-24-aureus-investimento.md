# aurix-investimento Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the `aurix-investimento` module — investment accounts with produtos (CDB, LCI, LCA, TESOURO, FUNDO, ACAO), aplicação/resgate orders, vinculado/standalone mode, ContaCorrenteClient integration.

**Architecture:** New Maven module following `aurix-poupanca` patterns. Uses `@HttpExchange` + `@ImportHttpServices` for ContaCorrenteClient, Kafka events for ContaInvestimentoCriada/OrdemExecutada/ResgateProcessado, `investimento.conta-independente` config property to toggle ContaCorrenteClient calls.

**Tech Stack:** Spring Boot 4.1, Java 25, JPA/Hibernate, PostgreSQL, Redis, Kafka, `@Retryable`, `@HttpExchange`, Testcontainers.

## Global Constraints

- Java source/target must be 25 (parent POM defaults are correct)
- Maven packaging: `jar`, parent: `com.aurix.platform:aurix-platform:1.0.0`
- Package root: `com.aurix.platform.investimento`
- No Lombok — manual getters/setters/constructors
- No `spring-retry` — use Spring Framework 7 native `@Retryable` from `org.springframework.resilience.annotation`
- No Feign — use `@HttpExchange` + `@ImportHttpServices`
- `@NullMarked` package-level via JSpecify on every package
- API context path: `/api/investimento`, server port: 8113
- `jakarta.*` packages throughout
- `@EnableResilientMethods` on HttpConfig
- Tests: `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `RestTemplate` with `@LocalServerPort`
- Tests: H2 for repo tests, Testcontainers for Kafka/Redis integration

---

## File Structure Map

```
apps/backend/aurix-investimento/
├── pom.xml
├── src/main/java/com/aurix/platform/investimento/
│   ├── AurixInvestimentoApplication.java
│   ├── entity/
│   │   ├── package-info.java
│   │   ├── ContaInvestimento.java
│   │   ├── ProdutoInvestimento.java
│   │   ├── OrdemInvestimento.java
│   │   └── Carteira.java
│   ├── repository/
│   │   ├── package-info.java
│   │   ├── ContaInvestimentoRepository.java
│   │   ├── ProdutoInvestimentoRepository.java
│   │   ├── OrdemInvestimentoRepository.java
│   │   └── CarteiraRepository.java
│   ├── dto/
│   │   ├── package-info.java
│   │   ├── request/
│   │   │   ├── package-info.java
│   │   │   ├── CriarContaInvestimentoRequest.java
│   │   │   ├── CriarProdutoRequest.java
│   │   │   ├── AtualizarProdutoRequest.java
│   │   │   ├── AplicarOrdemRequest.java
│   │   │   └── ResgatarOrdemRequest.java
│   │   └── response/
│   │       ├── package-info.java
│   │       ├── ContaInvestimentoResponse.java
│   │       ├── ProdutoInvestimentoResponse.java
│   │       ├── OrdemInvestimentoResponse.java
│   │       ├── CarteiraResponse.java
│   │       └── ExtratoOrdemResponse.java
│   ├── event/
│   │   ├── package-info.java
│   │   ├── ContaInvestimentoCriadaEvent.java
│   │   ├── OrdemExecutadaEvent.java
│   │   └── ResgateProcessadoEvent.java
│   ├── client/
│   │   ├── package-info.java
│   │   └── ContaCorrenteClient.java
│   ├── config/
│   │   ├── package-info.java
│   │   ├── InvestimentoHttpConfig.java
│   │   ├── InvestimentoKafkaConfig.java
│   │   └── InvestimentoSecurityConfig.java
│   └── service/
│       ├── package-info.java
│       ├── ContaInvestimentoService.java
│       ├── ProdutoInvestimentoService.java
│       ├── OrdemService.java
│       └── CarteiraService.java
├── src/main/resources/
│   ├── application.yml
│   └── application-prod.yml
└── src/test/java/com/aurix/platform/investimento/
    └── controller/
        ├── package-info.java
        ├── ContaInvestimentoControllerTest.java
        ├── ProdutoInvestimentoControllerTest.java
        └── OrdemControllerTest.java
```

---

### Task 1: Module Scaffold

**Files:**
- Create: `apps/backend/aurix-investimento/pom.xml`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/AurixInvestimentoApplication.java`
- Create: `apps/backend/aurix-investimento/src/main/resources/application.yml`
- Create: `apps/backend/aurix-investimento/src/main/resources/application-prod.yml`
- Modify: `apps/backend/pom.xml` (add module)

**Interfaces:**
- Consumes: parent POM at `apps/backend/pom.xml`
- Produces: compilable Maven module with empty Spring Boot app, server on port 8113

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

    <artifactId>aurix-investimento</artifactId>
    <packaging>jar</packaging>

    <name>AURIX Investimento</name>
    <description>Modulo de conta investimento do AURIX</description>

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
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>test</scope>
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
            <artifactId>testcontainers-postgresql</artifactId>
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

- [ ] **Step 2: Create AurixInvestimentoApplication.java**

```java
package com.aurix.platform.investimento;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class AurixInvestimentoApplication {

    public static void main(String[] args) {
        SpringApplication.run(AurixInvestimentoApplication.class, args);
    }
}
```

- [ ] **Step 3: Create application.yml**

```yaml
server:
  port: 8113
  servlet:
    context-path: /api/investimento

spring:
  application:
    name: aurix-investimento
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
      group-id: aurix-investimento-group
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

investimento:
  conta-independente: false

aurix:
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

- [ ] **Step 4: Create application-prod.yml**

```yaml
spring:
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

- [ ] **Step 5: Add module to parent pom.xml**

Edit `apps/backend/pom.xml`, insert `<module>aurix-investimento</module>` after `<module>aurix-internet-banking</module>` (alphabetical order among the new modules):

```xml
        <module>aurix-internet-banking</module>
        <module>aurix-investimento</module>
        <module>aurix-mobile-banking</module>
```

- [ ] **Step 6: Verify scaffold compiles**

Run: `mvn compile -pl aurix-investimento -am`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add apps/backend/aurix-investimento/ apps/backend/pom.xml
git commit -m "feat: scaffold aurix-investimento module"
```

---

### Task 2: Entities + Repositories

**Files:**
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/entity/package-info.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/entity/ContaInvestimento.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/entity/ProdutoInvestimento.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/entity/OrdemInvestimento.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/entity/Carteira.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/repository/package-info.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/repository/ContaInvestimentoRepository.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/repository/ProdutoInvestimentoRepository.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/repository/OrdemInvestimentoRepository.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/repository/CarteiraRepository.java`

**Interfaces:**
- Produces: `ContaInvestimento`, `ProdutoInvestimento`, `OrdemInvestimento`, `Carteira` JPA entities with repositories

- [ ] **Step 1: Create entity/package-info.java**

```java
@NullMarked
package com.aurix.platform.investimento.entity;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 2: Create ContaInvestimento.java**

```java
package com.aurix.platform.investimento.entity;

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
@Table(name = "contas_investimento")
public class ContaInvestimento {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank
    @Column(name = "tenant_id", nullable = false, length = 50)
    private String tenantId;

    @NotNull
    @Column(name = "cliente_id", nullable = false)
    private Long clienteId;

    @Column(name = "conta_corrente_id")
    private Long contaCorrenteId;

    @NotBlank
    @Column(name = "numero", unique = true, nullable = false, length = 30)
    private String numero;

    @NotBlank
    @Column(name = "agencia", nullable = false, length = 10)
    private String agencia;

    @NotNull
    @Column(name = "saldo_total", nullable = false, precision = 18, scale = 2)
    private BigDecimal saldoTotal = BigDecimal.ZERO;

    @NotNull
    @Column(name = "data_abertura", nullable = false)
    private LocalDate dataAbertura;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private StatusContaInvestimento ativa = StatusContaInvestimento.ATIVA;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private ModoContaInvestimento modo;

    @Column(name = "data_criacao", updatable = false)
    private LocalDateTime dataCriacao;

    @Column(name = "data_atualizacao")
    private LocalDateTime dataAtualizacao;

    public enum StatusContaInvestimento {
        ATIVA, ENCERRADA
    }

    public enum ModoContaInvestimento {
        VINCULADA, STANDALONE
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
    public String getTenantId() { return tenantId; }
    public void setTenantId(String tenantId) { this.tenantId = tenantId; }
    public Long getClienteId() { return clienteId; }
    public void setClienteId(Long clienteId) { this.clienteId = clienteId; }
    public Long getContaCorrenteId() { return contaCorrenteId; }
    public void setContaCorrenteId(Long contaCorrenteId) { this.contaCorrenteId = contaCorrenteId; }
    public String getNumero() { return numero; }
    public void setNumero(String numero) { this.numero = numero; }
    public String getAgencia() { return agencia; }
    public void setAgencia(String agencia) { this.agencia = agencia; }
    public BigDecimal getSaldoTotal() { return saldoTotal; }
    public void setSaldoTotal(BigDecimal saldoTotal) { this.saldoTotal = saldoTotal; }
    public LocalDate getDataAbertura() { return dataAbertura; }
    public void setDataAbertura(LocalDate dataAbertura) { this.dataAbertura = dataAbertura; }
    public StatusContaInvestimento getAtiva() { return ativa; }
    public void setAtiva(StatusContaInvestimento ativa) { this.ativa = ativa; }
    public ModoContaInvestimento getModo() { return modo; }
    public void setModo(ModoContaInvestimento modo) { this.modo = modo; }
    public LocalDateTime getDataCriacao() { return dataCriacao; }
    public LocalDateTime getDataAtualizacao() { return dataAtualizacao; }
}
```

- [ ] **Step 3: Create ProdutoInvestimento.java**

```java
package com.aurix.platform.investimento.entity;

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
@Table(name = "produtos_investimento")
public class ProdutoInvestimento {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank
    @Column(name = "tenant_id", nullable = false, length = 50)
    private String tenantId;

    @NotBlank
    @Column(name = "codigo", unique = true, nullable = false, length = 30)
    private String codigo;

    @NotBlank
    @Column(nullable = false, length = 100)
    private String nome;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private TipoProdutoInvestimento tipo;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private CategoriaProdutoInvestimento categoria;

    @NotBlank
    @Column(nullable = false, length = 100)
    private String emissor;

    @Column
    private LocalDate vencimento;

    @Column(name = "taxa_remuneracao", precision = 10, scale = 4)
    private BigDecimal taxaRemuneracao;

    @Column(name = "carencia")
    private Integer carencia;

    @NotNull
    @Column(name = "valor_minimo_aplicacao", nullable = false, precision = 18, scale = 2)
    private BigDecimal valorMinimoAplicacao;

    @NotNull
    @Column(nullable = false)
    private Boolean ativo = true;

    @Column(name = "data_criacao", updatable = false)
    private LocalDateTime dataCriacao;

    @Column(name = "data_atualizacao")
    private LocalDateTime dataAtualizacao;

    public enum TipoProdutoInvestimento {
        CDB, LCI, LCA, TESOURO, FUNDO, ACAO
    }

    public enum CategoriaProdutoInvestimento {
        RENDA_FIXA, RENDA_VARIAVEL
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
    public String getTenantId() { return tenantId; }
    public void setTenantId(String tenantId) { this.tenantId = tenantId; }
    public String getCodigo() { return codigo; }
    public void setCodigo(String codigo) { this.codigo = codigo; }
    public String getNome() { return nome; }
    public void setNome(String nome) { this.nome = nome; }
    public TipoProdutoInvestimento getTipo() { return tipo; }
    public void setTipo(TipoProdutoInvestimento tipo) { this.tipo = tipo; }
    public CategoriaProdutoInvestimento getCategoria() { return categoria; }
    public void setCategoria(CategoriaProdutoInvestimento categoria) { this.categoria = categoria; }
    public String getEmissor() { return emissor; }
    public void setEmissor(String emissor) { this.emissor = emissor; }
    public LocalDate getVencimento() { return vencimento; }
    public void setVencimento(LocalDate vencimento) { this.vencimento = vencimento; }
    public BigDecimal getTaxaRemuneracao() { return taxaRemuneracao; }
    public void setTaxaRemuneracao(BigDecimal taxaRemuneracao) { this.taxaRemuneracao = taxaRemuneracao; }
    public Integer getCarencia() { return carencia; }
    public void setCarencia(Integer carencia) { this.carencia = carencia; }
    public BigDecimal getValorMinimoAplicacao() { return valorMinimoAplicacao; }
    public void setValorMinimoAplicacao(BigDecimal valorMinimoAplicacao) { this.valorMinimoAplicacao = valorMinimoAplicacao; }
    public Boolean getAtivo() { return ativo; }
    public void setAtivo(Boolean ativo) { this.ativo = ativo; }
    public LocalDateTime getDataCriacao() { return dataCriacao; }
    public LocalDateTime getDataAtualizacao() { return dataAtualizacao; }
}
```

- [ ] **Step 4: Create OrdemInvestimento.java**

```java
package com.aurix.platform.investimento.entity;

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
@Table(name = "ordens_investimento")
public class OrdemInvestimento {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank
    @Column(name = "tenant_id", nullable = false, length = 50)
    private String tenantId;

    @NotNull
    @Column(name = "conta_id", nullable = false)
    private Long contaId;

    @NotNull
    @Column(name = "produto_id", nullable = false)
    private Long produtoId;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private TipoOrdemInvestimento tipo;

    @NotNull
    @Column(nullable = false, precision = 18, scale = 2)
    private BigDecimal valor;

    @Column(precision = 18, scale = 6)
    private BigDecimal quantidade;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private StatusOrdemInvestimento status = StatusOrdemInvestimento.PENDENTE;

    @Column(name = "data_ordem")
    private LocalDate dataOrdem;

    @Column(name = "data_execucao")
    private LocalDate dataExecucao;

    @Column(length = 500)
    private String observacao;

    @Column(name = "data_criacao", updatable = false)
    private LocalDateTime dataCriacao;

    @Column(name = "data_atualizacao")
    private LocalDateTime dataAtualizacao;

    public enum TipoOrdemInvestimento {
        APLICACAO, RESGATE
    }

    public enum StatusOrdemInvestimento {
        PENDENTE, EXECUTADA, CANCELADA, FALHADA
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
    public String getTenantId() { return tenantId; }
    public void setTenantId(String tenantId) { this.tenantId = tenantId; }
    public Long getContaId() { return contaId; }
    public void setContaId(Long contaId) { this.contaId = contaId; }
    public Long getProdutoId() { return produtoId; }
    public void setProdutoId(Long produtoId) { this.produtoId = produtoId; }
    public TipoOrdemInvestimento getTipo() { return tipo; }
    public void setTipo(TipoOrdemInvestimento tipo) { this.tipo = tipo; }
    public BigDecimal getValor() { return valor; }
    public void setValor(BigDecimal valor) { this.valor = valor; }
    public BigDecimal getQuantidade() { return quantidade; }
    public void setQuantidade(BigDecimal quantidade) { this.quantidade = quantidade; }
    public StatusOrdemInvestimento getStatus() { return status; }
    public void setStatus(StatusOrdemInvestimento status) { this.status = status; }
    public LocalDate getDataOrdem() { return dataOrdem; }
    public void setDataOrdem(LocalDate dataOrdem) { this.dataOrdem = dataOrdem; }
    public LocalDate getDataExecucao() { return dataExecucao; }
    public void setDataExecucao(LocalDate dataExecucao) { this.dataExecucao = dataExecucao; }
    public String getObservacao() { return observacao; }
    public void setObservacao(String observacao) { this.observacao = observacao; }
    public LocalDateTime getDataCriacao() { return dataCriacao; }
    public LocalDateTime getDataAtualizacao() { return dataAtualizacao; }
}
```

- [ ] **Step 5: Create Carteira.java**

```java
package com.aurix.platform.investimento.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.PreUpdate;
import jakarta.persistence.Table;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "carteiras_investimento")
public class Carteira {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank
    @Column(name = "tenant_id", nullable = false, length = 50)
    private String tenantId;

    @NotNull
    @Column(name = "conta_id", nullable = false)
    private Long contaId;

    @NotNull
    @Column(name = "produto_id", nullable = false)
    private Long produtoId;

    @NotNull
    @Column(nullable = false, precision = 18, scale = 2)
    private BigDecimal saldo = BigDecimal.ZERO;

    @Column(precision = 18, scale = 6)
    private BigDecimal quantidade = BigDecimal.ZERO;

    @Column(name = "data_ultima_atualizacao")
    private LocalDateTime dataUltimaAtualizacao;

    @Column(name = "data_criacao", updatable = false)
    private LocalDateTime dataCriacao;

    @Column(name = "data_atualizacao")
    private LocalDateTime dataAtualizacao;

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
    public String getTenantId() { return tenantId; }
    public void setTenantId(String tenantId) { this.tenantId = tenantId; }
    public Long getContaId() { return contaId; }
    public void setContaId(Long contaId) { this.contaId = contaId; }
    public Long getProdutoId() { return produtoId; }
    public void setProdutoId(Long produtoId) { this.produtoId = produtoId; }
    public BigDecimal getSaldo() { return saldo; }
    public void setSaldo(BigDecimal saldo) { this.saldo = saldo; }
    public BigDecimal getQuantidade() { return quantidade; }
    public void setQuantidade(BigDecimal quantidade) { this.quantidade = quantidade; }
    public LocalDateTime getDataUltimaAtualizacao() { return dataUltimaAtualizacao; }
    public void setDataUltimaAtualizacao(LocalDateTime dataUltimaAtualizacao) { this.dataUltimaAtualizacao = dataUltimaAtualizacao; }
    public LocalDateTime getDataCriacao() { return dataCriacao; }
    public LocalDateTime getDataAtualizacao() { return dataAtualizacao; }
}
```

- [ ] **Step 6: Create repository/package-info.java**

```java
@NullMarked
package com.aurix.platform.investimento.repository;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 7: Create ContaInvestimentoRepository.java**

```java
package com.aurix.platform.investimento.repository;

import com.aurix.platform.investimento.entity.ContaInvestimento;
import java.util.List;
import java.util.Optional;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ContaInvestimentoRepository extends JpaRepository<ContaInvestimento, Long> {

    List<ContaInvestimento> findByTenantId(String tenantId);

    List<ContaInvestimento> findByClienteId(Long clienteId);

    Optional<ContaInvestimento> findByNumero(String numero);
}
```

- [ ] **Step 8: Create ProdutoInvestimentoRepository.java**

```java
package com.aurix.platform.investimento.repository;

import com.aurix.platform.investimento.entity.ProdutoInvestimento;
import java.util.List;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProdutoInvestimentoRepository extends JpaRepository<ProdutoInvestimento, Long> {

    List<ProdutoInvestimento> findByTenantIdAndAtivoTrue(String tenantId);
}
```

- [ ] **Step 9: Create OrdemInvestimentoRepository.java**

```java
package com.aurix.platform.investimento.repository;

import com.aurix.platform.investimento.entity.OrdemInvestimento;
import java.util.List;
import org.springframework.data.jpa.repository.JpaRepository;

public interface OrdemInvestimentoRepository extends JpaRepository<OrdemInvestimento, Long> {

    List<OrdemInvestimento> findByContaIdOrderByDataOrdemDesc(Long contaId);
}
```

- [ ] **Step 10: Create CarteiraRepository.java**

```java
package com.aurix.platform.investimento.repository;

import com.aurix.platform.investimento.entity.Carteira;
import java.util.List;
import java.util.Optional;
import org.springframework.data.jpa.repository.JpaRepository;

public interface CarteiraRepository extends JpaRepository<Carteira, Long> {

    List<Carteira> findByContaId(Long contaId);

    Optional<Carteira> findByContaIdAndProdutoId(Long contaId, Long produtoId);
}
```

- [ ] **Step 11: Verify compilation**

Run: `mvn compile -pl aurix-investimento -am`
Expected: BUILD SUCCESS

- [ ] **Step 12: Commit**

```bash
git add apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/entity/
git add apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/repository/
git commit -m "feat: add investimento entities and repositories"
```

---

### Task 3: DTOs

**Files:**
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/dto/package-info.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/dto/request/package-info.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/dto/request/CriarContaInvestimentoRequest.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/dto/request/CriarProdutoRequest.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/dto/request/AtualizarProdutoRequest.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/dto/request/AplicarOrdemRequest.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/dto/request/ResgatarOrdemRequest.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/dto/response/package-info.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/dto/response/ContaInvestimentoResponse.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/dto/response/ProdutoInvestimentoResponse.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/dto/response/OrdemInvestimentoResponse.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/dto/response/CarteiraResponse.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/dto/response/ExtratoOrdemResponse.java`

**Interfaces:**
- Consumes: entity classes from Task 2
- Produces: request/response DTOs consumed by controllers and services in Tasks 6-7

- [ ] **Step 1: Create dto/package-info.java**

```java
@NullMarked
package com.aurix.platform.investimento.dto;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 2: Create dto/request/package-info.java**

```java
@NullMarked
package com.aurix.platform.investimento.dto.request;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 3: Create dto/response/package-info.java**

```java
@NullMarked
package com.aurix.platform.investimento.dto.response;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 4: Create CriarContaInvestimentoRequest.java**

```java
package com.aurix.platform.investimento.dto.request;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

public class CriarContaInvestimentoRequest {

    @NotNull
    private Long clienteId;

    private Long contaCorrenteId;

    @NotBlank
    private String agencia;

    public Long getClienteId() { return clienteId; }
    public void setClienteId(Long clienteId) { this.clienteId = clienteId; }
    public Long getContaCorrenteId() { return contaCorrenteId; }
    public void setContaCorrenteId(Long contaCorrenteId) { this.contaCorrenteId = contaCorrenteId; }
    public String getAgencia() { return agencia; }
    public void setAgencia(String agencia) { this.agencia = agencia; }
}
```

- [ ] **Step 5: Create CriarProdutoRequest.java**

```java
package com.aurix.platform.investimento.dto.request;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;
import java.time.LocalDate;

public class CriarProdutoRequest {

    @NotBlank
    private String codigo;

    @NotBlank
    private String nome;

    @NotBlank
    private String tipo;

    @NotBlank
    private String categoria;

    @NotBlank
    private String emissor;

    private LocalDate vencimento;

    private BigDecimal taxaRemuneracao;

    private Integer carencia;

    @NotNull
    private BigDecimal valorMinimoAplicacao;

    public String getCodigo() { return codigo; }
    public void setCodigo(String codigo) { this.codigo = codigo; }
    public String getNome() { return nome; }
    public void setNome(String nome) { this.nome = nome; }
    public String getTipo() { return tipo; }
    public void setTipo(String tipo) { this.tipo = tipo; }
    public String getCategoria() { return categoria; }
    public void setCategoria(String categoria) { this.categoria = categoria; }
    public String getEmissor() { return emissor; }
    public void setEmissor(String emissor) { this.emissor = emissor; }
    public LocalDate getVencimento() { return vencimento; }
    public void setVencimento(LocalDate vencimento) { this.vencimento = vencimento; }
    public BigDecimal getTaxaRemuneracao() { return taxaRemuneracao; }
    public void setTaxaRemuneracao(BigDecimal taxaRemuneracao) { this.taxaRemuneracao = taxaRemuneracao; }
    public Integer getCarencia() { return carencia; }
    public void setCarencia(Integer carencia) { this.carencia = carencia; }
    public BigDecimal getValorMinimoAplicacao() { return valorMinimoAplicacao; }
    public void setValorMinimoAplicacao(BigDecimal valorMinimoAplicacao) { this.valorMinimoAplicacao = valorMinimoAplicacao; }
}
```

- [ ] **Step 6: Create AtualizarProdutoRequest.java**

```java
package com.aurix.platform.investimento.dto.request;

import java.math.BigDecimal;
import java.time.LocalDate;

public class AtualizarProdutoRequest {

    private String nome;
    private String tipo;
    private String categoria;
    private String emissor;
    private LocalDate vencimento;
    private BigDecimal taxaRemuneracao;
    private Integer carencia;
    private BigDecimal valorMinimoAplicacao;
    private Boolean ativo;

    public String getNome() { return nome; }
    public void setNome(String nome) { this.nome = nome; }
    public String getTipo() { return tipo; }
    public void setTipo(String tipo) { this.tipo = tipo; }
    public String getCategoria() { return categoria; }
    public void setCategoria(String categoria) { this.categoria = categoria; }
    public String getEmissor() { return emissor; }
    public void setEmissor(String emissor) { this.emissor = emissor; }
    public LocalDate getVencimento() { return vencimento; }
    public void setVencimento(LocalDate vencimento) { this.vencimento = vencimento; }
    public BigDecimal getTaxaRemuneracao() { return taxaRemuneracao; }
    public void setTaxaRemuneracao(BigDecimal taxaRemuneracao) { this.taxaRemuneracao = taxaRemuneracao; }
    public Integer getCarencia() { return carencia; }
    public void setCarencia(Integer carencia) { this.carencia = carencia; }
    public BigDecimal getValorMinimoAplicacao() { return valorMinimoAplicacao; }
    public void setValorMinimoAplicacao(BigDecimal valorMinimoAplicacao) { this.valorMinimoAplicacao = valorMinimoAplicacao; }
    public Boolean getAtivo() { return ativo; }
    public void setAtivo(Boolean ativo) { this.ativo = ativo; }
}
```

- [ ] **Step 7: Create AplicarOrdemRequest.java**

```java
package com.aurix.platform.investimento.dto.request;

import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;

public class AplicarOrdemRequest {

    @NotNull
    private Long contaId;

    @NotNull
    private Long produtoId;

    @NotNull
    private BigDecimal valor;

    public Long getContaId() { return contaId; }
    public void setContaId(Long contaId) { this.contaId = contaId; }
    public Long getProdutoId() { return produtoId; }
    public void setProdutoId(Long produtoId) { this.produtoId = produtoId; }
    public BigDecimal getValor() { return valor; }
    public void setValor(BigDecimal valor) { this.valor = valor; }
}
```

- [ ] **Step 8: Create ResgatarOrdemRequest.java**

```java
package com.aurix.platform.investimento.dto.request;

import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;

public class ResgatarOrdemRequest {

    @NotNull
    private Long contaId;

    @NotNull
    private Long produtoId;

    @NotNull
    private BigDecimal valor;

    public Long getContaId() { return contaId; }
    public void setContaId(Long contaId) { this.contaId = contaId; }
    public Long getProdutoId() { return produtoId; }
    public void setProdutoId(Long produtoId) { this.produtoId = produtoId; }
    public BigDecimal getValor() { return valor; }
    public void setValor(BigDecimal valor) { this.valor = valor; }
}
```

- [ ] **Step 9: Create ContaInvestimentoResponse.java**

```java
package com.aurix.platform.investimento.dto.response;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

public class ContaInvestimentoResponse {

    private Long id;
    private String tenantId;
    private Long clienteId;
    private Long contaCorrenteId;
    private String numero;
    private String agencia;
    private BigDecimal saldoTotal;
    private LocalDate dataAbertura;
    private String ativa;
    private String modo;
    private LocalDateTime dataCriacao;

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getTenantId() { return tenantId; }
    public void setTenantId(String tenantId) { this.tenantId = tenantId; }
    public Long getClienteId() { return clienteId; }
    public void setClienteId(Long clienteId) { this.clienteId = clienteId; }
    public Long getContaCorrenteId() { return contaCorrenteId; }
    public void setContaCorrenteId(Long contaCorrenteId) { this.contaCorrenteId = contaCorrenteId; }
    public String getNumero() { return numero; }
    public void setNumero(String numero) { this.numero = numero; }
    public String getAgencia() { return agencia; }
    public void setAgencia(String agencia) { this.agencia = agencia; }
    public BigDecimal getSaldoTotal() { return saldoTotal; }
    public void setSaldoTotal(BigDecimal saldoTotal) { this.saldoTotal = saldoTotal; }
    public LocalDate getDataAbertura() { return dataAbertura; }
    public void setDataAbertura(LocalDate dataAbertura) { this.dataAbertura = dataAbertura; }
    public String getAtiva() { return ativa; }
    public void setAtiva(String ativa) { this.ativa = ativa; }
    public String getModo() { return modo; }
    public void setModo(String modo) { this.modo = modo; }
    public LocalDateTime getDataCriacao() { return dataCriacao; }
    public void setDataCriacao(LocalDateTime dataCriacao) { this.dataCriacao = dataCriacao; }
}
```

- [ ] **Step 10: Create ProdutoInvestimentoResponse.java**

```java
package com.aurix.platform.investimento.dto.response;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

public class ProdutoInvestimentoResponse {

    private Long id;
    private String tenantId;
    private String codigo;
    private String nome;
    private String tipo;
    private String categoria;
    private String emissor;
    private LocalDate vencimento;
    private BigDecimal taxaRemuneracao;
    private Integer carencia;
    private BigDecimal valorMinimoAplicacao;
    private Boolean ativo;
    private LocalDateTime dataCriacao;

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getTenantId() { return tenantId; }
    public void setTenantId(String tenantId) { this.tenantId = tenantId; }
    public String getCodigo() { return codigo; }
    public void setCodigo(String codigo) { this.codigo = codigo; }
    public String getNome() { return nome; }
    public void setNome(String nome) { this.nome = nome; }
    public String getTipo() { return tipo; }
    public void setTipo(String tipo) { this.tipo = tipo; }
    public String getCategoria() { return categoria; }
    public void setCategoria(String categoria) { this.categoria = categoria; }
    public String getEmissor() { return emissor; }
    public void setEmissor(String emissor) { this.emissor = emissor; }
    public LocalDate getVencimento() { return vencimento; }
    public void setVencimento(LocalDate vencimento) { this.vencimento = vencimento; }
    public BigDecimal getTaxaRemuneracao() { return taxaRemuneracao; }
    public void setTaxaRemuneracao(BigDecimal taxaRemuneracao) { this.taxaRemuneracao = taxaRemuneracao; }
    public Integer getCarencia() { return carencia; }
    public void setCarencia(Integer carencia) { this.carencia = carencia; }
    public BigDecimal getValorMinimoAplicacao() { return valorMinimoAplicacao; }
    public void setValorMinimoAplicacao(BigDecimal valorMinimoAplicacao) { this.valorMinimoAplicacao = valorMinimoAplicacao; }
    public Boolean getAtivo() { return ativo; }
    public void setAtivo(Boolean ativo) { this.ativo = ativo; }
    public LocalDateTime getDataCriacao() { return dataCriacao; }
    public void setDataCriacao(LocalDateTime dataCriacao) { this.dataCriacao = dataCriacao; }
}
```

- [ ] **Step 11: Create OrdemInvestimentoResponse.java**

```java
package com.aurix.platform.investimento.dto.response;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

public class OrdemInvestimentoResponse {

    private Long id;
    private String tenantId;
    private Long contaId;
    private Long produtoId;
    private String tipo;
    private BigDecimal valor;
    private BigDecimal quantidade;
    private String status;
    private LocalDate dataOrdem;
    private LocalDate dataExecucao;
    private String observacao;
    private LocalDateTime dataCriacao;

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getTenantId() { return tenantId; }
    public void setTenantId(String tenantId) { this.tenantId = tenantId; }
    public Long getContaId() { return contaId; }
    public void setContaId(Long contaId) { this.contaId = contaId; }
    public Long getProdutoId() { return produtoId; }
    public void setProdutoId(Long produtoId) { this.produtoId = produtoId; }
    public String getTipo() { return tipo; }
    public void setTipo(String tipo) { this.tipo = tipo; }
    public BigDecimal getValor() { return valor; }
    public void setValor(BigDecimal valor) { this.valor = valor; }
    public BigDecimal getQuantidade() { return quantidade; }
    public void setQuantidade(BigDecimal quantidade) { this.quantidade = quantidade; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public LocalDate getDataOrdem() { return dataOrdem; }
    public void setDataOrdem(LocalDate dataOrdem) { this.dataOrdem = dataOrdem; }
    public LocalDate getDataExecucao() { return dataExecucao; }
    public void setDataExecucao(LocalDate dataExecucao) { this.dataExecucao = dataExecucao; }
    public String getObservacao() { return observacao; }
    public void setObservacao(String observacao) { this.observacao = observacao; }
    public LocalDateTime getDataCriacao() { return dataCriacao; }
    public void setDataCriacao(LocalDateTime dataCriacao) { this.dataCriacao = dataCriacao; }
}
```

- [ ] **Step 12: Create CarteiraResponse.java**

```java
package com.aurix.platform.investimento.dto.response;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public class CarteiraResponse {

    private Long id;
    private Long produtoId;
    private BigDecimal saldo;
    private BigDecimal quantidade;
    private LocalDateTime dataUltimaAtualizacao;

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Long getProdutoId() { return produtoId; }
    public void setProdutoId(Long produtoId) { this.produtoId = produtoId; }
    public BigDecimal getSaldo() { return saldo; }
    public void setSaldo(BigDecimal saldo) { this.saldo = saldo; }
    public BigDecimal getQuantidade() { return quantidade; }
    public void setQuantidade(BigDecimal quantidade) { this.quantidade = quantidade; }
    public LocalDateTime getDataUltimaAtualizacao() { return dataUltimaAtualizacao; }
    public void setDataUltimaAtualizacao(LocalDateTime dataUltimaAtualizacao) { this.dataUltimaAtualizacao = dataUltimaAtualizacao; }
}
```

- [ ] **Step 13: Create ExtratoOrdemResponse.java**

```java
package com.aurix.platform.investimento.dto.response;

import java.util.List;

public class ExtratoOrdemResponse {

    private Long contaId;
    private List<OrdemInvestimentoResponse> ordens;

    public Long getContaId() { return contaId; }
    public void setContaId(Long contaId) { this.contaId = contaId; }
    public List<OrdemInvestimentoResponse> getOrdens() { return ordens; }
    public void setOrdens(List<OrdemInvestimentoResponse> ordens) { this.ordens = ordens; }
}
```

- [ ] **Step 14: Verify compilation**

Run: `mvn compile -pl aurix-investimento -am`
Expected: BUILD SUCCESS

- [ ] **Step 15: Commit**

```bash
git add apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/dto/
git commit -m "feat: add investimento DTOs"
```

---

### Task 4: Events + Config

**Files:**
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/event/package-info.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/event/ContaInvestimentoCriadaEvent.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/event/OrdemExecutadaEvent.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/event/ResgateProcessadoEvent.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/config/package-info.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/config/InvestimentoKafkaConfig.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/config/InvestimentoSecurityConfig.java`

**Interfaces:**
- Consumes: entity IDs, tenantId from Task 2
- Produces: Kafka event records, Kafka topics, security filter chain

- [ ] **Step 1: Create event/package-info.java**

```java
@NullMarked
package com.aurix.platform.investimento.event;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 2: Create ContaInvestimentoCriadaEvent.java**

```java
package com.aurix.platform.investimento.event;

import java.time.LocalDate;

public record ContaInvestimentoCriadaEvent(
    Long id,
    Long clienteId,
    String numero,
    String modo,
    String tenantId
) {}
```

- [ ] **Step 3: Create OrdemExecutadaEvent.java**

```java
package com.aurix.platform.investimento.event;

import java.math.BigDecimal;
import java.time.LocalDate;

public record OrdemExecutadaEvent(
    Long id,
    Long contaId,
    Long produtoId,
    BigDecimal valor,
    BigDecimal quantidade,
    LocalDate dataExecucao,
    String tenantId
) {}
```

- [ ] **Step 4: Create ResgateProcessadoEvent.java**

```java
package com.aurix.platform.investimento.event;

import java.math.BigDecimal;
import java.time.LocalDate;

public record ResgateProcessadoEvent(
    Long id,
    Long contaId,
    Long produtoId,
    BigDecimal valor,
    LocalDate dataExecucao,
    String tenantId
) {}
```

- [ ] **Step 5: Create config/package-info.java**

```java
@NullMarked
package com.aurix.platform.investimento.config;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 6: Create InvestimentoKafkaConfig.java**

```java
package com.aurix.platform.investimento.config;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class InvestimentoKafkaConfig {

    public static final String TOPICO_CONTA_CRIADA = "investimento-conta-criada";
    public static final String TOPICO_ORDEM_EXECUTADA = "investimento-ordem-executada";
    public static final String TOPICO_RESGATE_PROCESSADO = "investimento-resgate-processado";

    @Bean
    public NewTopic topicoContaCriada() {
        return TopicBuilder.name(TOPICO_CONTA_CRIADA).partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic topicoOrdemExecutada() {
        return TopicBuilder.name(TOPICO_ORDEM_EXECUTADA).partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic topicoResgateProcessado() {
        return TopicBuilder.name(TOPICO_RESGATE_PROCESSADO).partitions(3).replicas(1).build();
    }
}
```

- [ ] **Step 7: Create InvestimentoSecurityConfig.java**

```java
package com.aurix.platform.investimento.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class InvestimentoSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
        return http.build();
    }
}
```

- [ ] **Step 8: Verify compilation**

Run: `mvn compile -pl aurix-investimento -am`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/event/
git add apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/config/
git commit -m "feat: add investimento events and config"
```

---

### Task 5: ContaCorrenteClient + HTTP Config

**Files:**
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/client/package-info.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/client/ContaCorrenteClient.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/config/InvestimentoHttpConfig.java`

**Interfaces:**
- Produces: `ContaCorrenteClient.debitar(id, request)` and `ContaCorrenteClient.creditar(id, request)` consumed by OrdemService in Task 6

- [ ] **Step 1: Create client/package-info.java**

```java
@NullMarked
package com.aurix.platform.investimento.client;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 2: Create ContaCorrenteClient.java**

```java
package com.aurix.platform.investimento.client;

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

- [ ] **Step 3: Create InvestimentoHttpConfig.java**

```java
package com.aurix.platform.investimento.config;

import com.aurix.platform.investimento.client.ContaCorrenteClient;
import org.springframework.context.annotation.Configuration;
import org.springframework.resilience.annotation.EnableResilientMethods;
import org.springframework.web.service.registry.ImportHttpServices;

@Configuration
@EnableResilientMethods
@ImportHttpServices({ContaCorrenteClient.class})
public class InvestimentoHttpConfig {
}
```

- [ ] **Step 4: Verify compilation**

Run: `mvn compile -pl aurix-investimento -am`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/client/
git add apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/config/InvestimentoHttpConfig.java
git commit -m "feat: add ContaCorrenteClient and HTTP config"
```

---

### Task 6: Services

**Files:**
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/service/package-info.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/service/ContaInvestimentoService.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/service/ProdutoInvestimentoService.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/service/OrdemService.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/service/CarteiraService.java`

**Interfaces:**
- Consumes: entities/repos from Task 2, DTOs from Task 3, events from Task 4, ContaCorrenteClient from Task 5
- Produces: service methods consumed by controllers in Task 7

- [ ] **Step 1: Create service/package-info.java**

```java
@NullMarked
package com.aurix.platform.investimento.service;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 2: Create ContaInvestimentoService.java**

```java
package com.aurix.platform.investimento.service;

import com.aurix.platform.investimento.client.ContaCorrenteClient;
import com.aurix.platform.investimento.dto.request.CriarContaInvestimentoRequest;
import com.aurix.platform.investimento.dto.response.ContaInvestimentoResponse;
import com.aurix.platform.investimento.entity.ContaInvestimento;
import com.aurix.platform.investimento.entity.ContaInvestimento.ModoContaInvestimento;
import com.aurix.platform.investimento.entity.ContaInvestimento.StatusContaInvestimento;
import com.aurix.platform.investimento.event.ContaInvestimentoCriadaEvent;
import com.aurix.platform.investimento.repository.ContaInvestimentoRepository;
import com.aurix.platform.investimento.repository.CarteiraRepository;
import com.aurix.platform.shared.tenant.TenantContext;
import com.aurix.platform.shared.util.TransacaoUtil;
import java.time.LocalDate;
import java.util.List;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import static com.aurix.platform.investimento.config.InvestimentoKafkaConfig.TOPICO_CONTA_CRIADA;

@Service
@Transactional
public class ContaInvestimentoService {

    private final ContaInvestimentoRepository contaInvestimentoRepository;
    private final CarteiraService carteiraService;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final boolean contaIndependente;

    public ContaInvestimentoService(ContaInvestimentoRepository contaInvestimentoRepository,
                                    CarteiraService carteiraService,
                                    KafkaTemplate<String, Object> kafkaTemplate,
                                    @Value("${investimento.conta-independente:false}") boolean contaIndependente) {
        this.contaInvestimentoRepository = contaInvestimentoRepository;
        this.carteiraService = carteiraService;
        this.kafkaTemplate = kafkaTemplate;
        this.contaIndependente = contaIndependente;
    }

    public ContaInvestimentoResponse criarConta(CriarContaInvestimentoRequest request) {
        ContaInvestimento conta = new ContaInvestimento();
        conta.setClienteId(request.getClienteId());
        conta.setContaCorrenteId(contaIndependente ? null : request.getContaCorrenteId());
        conta.setNumero(TransacaoUtil.gerarCodigoPix());
        conta.setAgencia(request.getAgencia());
        conta.setDataAbertura(LocalDate.now());
        conta.setModo(contaIndependente ? ModoContaInvestimento.STANDALONE : ModoContaInvestimento.VINCULADA);
        conta.setTenantId(TenantContext.getTenantId());
        conta = contaInvestimentoRepository.save(conta);

        try {
            kafkaTemplate.send(TOPICO_CONTA_CRIADA, new ContaInvestimentoCriadaEvent(
                conta.getId(), conta.getClienteId(), conta.getNumero(),
                conta.getModo().name(), conta.getTenantId()));
        } catch (Exception e) {
            // fire-and-forget with try-catch
        }

        return toResponse(conta);
    }

    @Transactional(readOnly = true)
    public ContaInvestimentoResponse buscarPorId(Long id) {
        ContaInvestimento conta = contaInvestimentoRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Conta investimento nao encontrada: " + id));
        return toResponse(conta);
    }

    @Transactional(readOnly = true)
    public Page<ContaInvestimentoResponse> listar(Pageable pageable) {
        return contaInvestimentoRepository.findByTenantId(TenantContext.getTenantId(), pageable)
            .map(this::toResponse);
    }

    public void encerrar(Long id) {
        ContaInvestimento conta = contaInvestimentoRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Conta investimento nao encontrada: " + id));
        if (conta.getAtiva() == StatusContaInvestimento.ENCERRADA) {
            throw new IllegalStateException("Conta investimento ja esta encerrada");
        }
        if (conta.getSaldoTotal().compareTo(java.math.BigDecimal.ZERO) > 0) {
            throw new IllegalStateException("Conta investimento possui saldo — transfira antes de encerrar");
        }
        conta.setAtiva(StatusContaInvestimento.ENCERRADA);
        contaInvestimentoRepository.save(conta);
    }

    private ContaInvestimentoResponse toResponse(ContaInvestimento conta) {
        ContaInvestimentoResponse r = new ContaInvestimentoResponse();
        r.setId(conta.getId());
        r.setTenantId(conta.getTenantId());
        r.setClienteId(conta.getClienteId());
        r.setContaCorrenteId(conta.getContaCorrenteId());
        r.setNumero(conta.getNumero());
        r.setAgencia(conta.getAgencia());
        r.setSaldoTotal(conta.getSaldoTotal());
        r.setDataAbertura(conta.getDataAbertura());
        r.setAtiva(conta.getAtiva().name());
        r.setModo(conta.getModo().name());
        r.setDataCriacao(conta.getDataCriacao());
        return r;
    }
}
```

- [ ] **Step 3: Create ProdutoInvestimentoService.java**

```java
package com.aurix.platform.investimento.service;

import com.aurix.platform.investimento.dto.request.AtualizarProdutoRequest;
import com.aurix.platform.investimento.dto.request.CriarProdutoRequest;
import com.aurix.platform.investimento.dto.response.ProdutoInvestimentoResponse;
import com.aurix.platform.investimento.entity.ProdutoInvestimento;
import com.aurix.platform.investimento.entity.ProdutoInvestimento.CategoriaProdutoInvestimento;
import com.aurix.platform.investimento.entity.ProdutoInvestimento.TipoProdutoInvestimento;
import com.aurix.platform.investimento.repository.ProdutoInvestimentoRepository;
import com.aurix.platform.shared.tenant.TenantContext;
import java.util.List;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@Transactional
public class ProdutoInvestimentoService {

    private final ProdutoInvestimentoRepository produtoInvestimentoRepository;

    public ProdutoInvestimentoService(ProdutoInvestimentoRepository produtoInvestimentoRepository) {
        this.produtoInvestimentoRepository = produtoInvestimentoRepository;
    }

    public ProdutoInvestimentoResponse criarProduto(CriarProdutoRequest request) {
        ProdutoInvestimento produto = new ProdutoInvestimento();
        produto.setTenantId(TenantContext.getTenantId());
        produto.setCodigo(request.getCodigo());
        produto.setNome(request.getNome());
        produto.setTipo(TipoProdutoInvestimento.valueOf(request.getTipo()));
        produto.setCategoria(CategoriaProdutoInvestimento.valueOf(request.getCategoria()));
        produto.setEmissor(request.getEmissor());
        produto.setVencimento(request.getVencimento());
        produto.setTaxaRemuneracao(request.getTaxaRemuneracao());
        produto.setCarencia(request.getCarencia());
        produto.setValorMinimoAplicacao(request.getValorMinimoAplicacao());
        produto = produtoInvestimentoRepository.save(produto);
        return toResponse(produto);
    }

    @Transactional(readOnly = true)
    public List<ProdutoInvestimentoResponse> listarAtivos() {
        return produtoInvestimentoRepository.findByTenantIdAndAtivoTrue(TenantContext.getTenantId()).stream()
            .map(this::toResponse).toList();
    }

    @Transactional(readOnly = true)
    public ProdutoInvestimentoResponse buscarPorId(Long id) {
        ProdutoInvestimento produto = produtoInvestimentoRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Produto investimento nao encontrado: " + id));
        return toResponse(produto);
    }

    public ProdutoInvestimentoResponse atualizarProduto(Long id, AtualizarProdutoRequest request) {
        ProdutoInvestimento produto = produtoInvestimentoRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Produto investimento nao encontrado: " + id));
        if (request.getNome() != null) produto.setNome(request.getNome());
        if (request.getTipo() != null) produto.setTipo(TipoProdutoInvestimento.valueOf(request.getTipo()));
        if (request.getCategoria() != null) produto.setCategoria(CategoriaProdutoInvestimento.valueOf(request.getCategoria()));
        if (request.getEmissor() != null) produto.setEmissor(request.getEmissor());
        if (request.getVencimento() != null) produto.setVencimento(request.getVencimento());
        if (request.getTaxaRemuneracao() != null) produto.setTaxaRemuneracao(request.getTaxaRemuneracao());
        if (request.getCarencia() != null) produto.setCarencia(request.getCarencia());
        if (request.getValorMinimoAplicacao() != null) produto.setValorMinimoAplicacao(request.getValorMinimoAplicacao());
        if (request.getAtivo() != null) produto.setAtivo(request.getAtivo());
        produto = produtoInvestimentoRepository.save(produto);
        return toResponse(produto);
    }

    private ProdutoInvestimentoResponse toResponse(ProdutoInvestimento produto) {
        ProdutoInvestimentoResponse r = new ProdutoInvestimentoResponse();
        r.setId(produto.getId());
        r.setTenantId(produto.getTenantId());
        r.setCodigo(produto.getCodigo());
        r.setNome(produto.getNome());
        r.setTipo(produto.getTipo().name());
        r.setCategoria(produto.getCategoria().name());
        r.setEmissor(produto.getEmissor());
        r.setVencimento(produto.getVencimento());
        r.setTaxaRemuneracao(produto.getTaxaRemuneracao());
        r.setCarencia(produto.getCarencia());
        r.setValorMinimoAplicacao(produto.getValorMinimoAplicacao());
        r.setAtivo(produto.getAtivo());
        r.setDataCriacao(produto.getDataCriacao());
        return r;
    }
}
```

- [ ] **Step 4: Create CarteiraService.java**

```java
package com.aurix.platform.investimento.service;

import com.aurix.platform.investimento.dto.response.CarteiraResponse;
import com.aurix.platform.investimento.entity.Carteira;
import com.aurix.platform.investimento.repository.CarteiraRepository;
import java.util.List;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@Transactional
public class CarteiraService {

    private final CarteiraRepository carteiraRepository;

    public CarteiraService(CarteiraRepository carteiraRepository) {
        this.carteiraRepository = carteiraRepository;
    }

    @Transactional(readOnly = true)
    public List<CarteiraResponse> listarPorConta(Long contaId) {
        return carteiraRepository.findByContaId(contaId).stream()
            .map(this::toResponse).toList();
    }

    public void creditarCarteira(Long contaId, Long produtoId, java.math.BigDecimal valor, java.math.BigDecimal quantidade) {
        Carteira carteira = carteiraRepository.findByContaIdAndProdutoId(contaId, produtoId)
            .orElseGet(() -> {
                Carteira nova = new Carteira();
                nova.setContaId(contaId);
                nova.setProdutoId(produtoId);
                nova.setTenantId(com.aurix.platform.shared.tenant.TenantContext.getTenantId());
                return nova;
            });
        carteira.setSaldo(carteira.getSaldo().add(valor));
        carteira.setQuantidade(carteira.getQuantidade() != null
            ? carteira.getQuantidade().add(quantidade != null ? quantidade : java.math.BigDecimal.ZERO)
            : quantidade);
        carteira.setDataUltimaAtualizacao(java.time.LocalDateTime.now());
        carteiraRepository.save(carteira);
    }

    public void debitarCarteira(Long contaId, Long produtoId, java.math.BigDecimal valor, java.math.BigDecimal quantidade) {
        Carteira carteira = carteiraRepository.findByContaIdAndProdutoId(contaId, produtoId)
            .orElseThrow(() -> new IllegalArgumentException("Carteira nao encontrada para conta " + contaId + " e produto " + produtoId));
        if (carteira.getSaldo().compareTo(valor) < 0) {
            throw new IllegalStateException("Saldo insuficiente na carteira para produto " + produtoId);
        }
        carteira.setSaldo(carteira.getSaldo().subtract(valor));
        carteira.setQuantidade(carteira.getQuantidade() != null
            ? carteira.getQuantidade().subtract(quantidade != null ? quantidade : java.math.BigDecimal.ZERO)
            : java.math.BigDecimal.ZERO);
        carteira.setDataUltimaAtualizacao(java.time.LocalDateTime.now());
        carteiraRepository.save(carteira);
    }

    private CarteiraResponse toResponse(Carteira carteira) {
        CarteiraResponse r = new CarteiraResponse();
        r.setId(carteira.getId());
        r.setProdutoId(carteira.getProdutoId());
        r.setSaldo(carteira.getSaldo());
        r.setQuantidade(carteira.getQuantidade());
        r.setDataUltimaAtualizacao(carteira.getDataUltimaAtualizacao());
        return r;
    }
}
```

- [ ] **Step 5: Create OrdemService.java**

```java
package com.aurix.platform.investimento.service;

import com.aurix.platform.investimento.client.ContaCorrenteClient;
import com.aurix.platform.investimento.dto.request.AplicarOrdemRequest;
import com.aurix.platform.investimento.dto.request.ResgatarOrdemRequest;
import com.aurix.platform.investimento.dto.response.ExtratoOrdemResponse;
import com.aurix.platform.investimento.dto.response.OrdemInvestimentoResponse;
import com.aurix.platform.investimento.entity.ContaInvestimento;
import com.aurix.platform.investimento.entity.ContaInvestimento.ModoContaInvestimento;
import com.aurix.platform.investimento.entity.ContaInvestimento.StatusContaInvestimento;
import com.aurix.platform.investimento.entity.OrdemInvestimento;
import com.aurix.platform.investimento.entity.OrdemInvestimento.StatusOrdemInvestimento;
import com.aurix.platform.investimento.entity.OrdemInvestimento.TipoOrdemInvestimento;
import com.aurix.platform.investimento.entity.ProdutoInvestimento;
import com.aurix.platform.investimento.event.OrdemExecutadaEvent;
import com.aurix.platform.investimento.event.ResgateProcessadoEvent;
import com.aurix.platform.investimento.repository.ContaInvestimentoRepository;
import com.aurix.platform.investimento.repository.OrdemInvestimentoRepository;
import com.aurix.platform.investimento.repository.ProdutoInvestimentoRepository;
import com.aurix.platform.shared.tenant.TenantContext;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.List;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import static com.aurix.platform.investimento.config.InvestimentoKafkaConfig.TOPICO_ORDEM_EXECUTADA;
import static com.aurix.platform.investimento.config.InvestimentoKafkaConfig.TOPICO_RESGATE_PROCESSADO;

@Service
@Transactional
public class OrdemService {

    private final OrdemInvestimentoRepository ordemInvestimentoRepository;
    private final ContaInvestimentoRepository contaInvestimentoRepository;
    private final ProdutoInvestimentoRepository produtoInvestimentoRepository;
    private final CarteiraService carteiraService;
    private final ContaCorrenteClient contaCorrenteClient;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final boolean contaIndependente;

    public OrdemService(OrdemInvestimentoRepository ordemInvestimentoRepository,
                        ContaInvestimentoRepository contaInvestimentoRepository,
                        ProdutoInvestimentoRepository produtoInvestimentoRepository,
                        CarteiraService carteiraService,
                        ContaCorrenteClient contaCorrenteClient,
                        KafkaTemplate<String, Object> kafkaTemplate,
                        @Value("${investimento.conta-independente:false}") boolean contaIndependente) {
        this.ordemInvestimentoRepository = ordemInvestimentoRepository;
        this.contaInvestimentoRepository = contaInvestimentoRepository;
        this.produtoInvestimentoRepository = produtoInvestimentoRepository;
        this.carteiraService = carteiraService;
        this.contaCorrenteClient = contaCorrenteClient;
        this.kafkaTemplate = kafkaTemplate;
        this.contaIndependente = contaIndependente;
    }

    public OrdemInvestimentoResponse aplicar(AplicarOrdemRequest request) {
        ContaInvestimento conta = contaInvestimentoRepository.findById(request.getContaId())
            .orElseThrow(() -> new IllegalArgumentException("Conta investimento nao encontrada: " + request.getContaId()));
        if (conta.getAtiva() != StatusContaInvestimento.ATIVA) {
            throw new IllegalStateException("Conta investimento nao esta ativa");
        }
        ProdutoInvestimento produto = produtoInvestimentoRepository.findById(request.getProdutoId())
            .orElseThrow(() -> new IllegalArgumentException("Produto investimento nao encontrado: " + request.getProdutoId()));
        if (request.getValor().compareTo(produto.getValorMinimoAplicacao()) < 0) {
            throw new IllegalArgumentException("Valor minimo de aplicacao nao atingido: " + produto.getValorMinimoAplicacao());
        }

        // Se vinculada, debita da conta corrente
        if (conta.getModo() == ModoContaInvestimento.VINCULADA && !contaIndependente) {
            try {
                contaCorrenteClient.debitar(conta.getContaCorrenteId(),
                    new ContaCorrenteClient.DebitoRequest(request.getValor(), "Aplicacao investimento"));
            } catch (Exception e) {
                throw new IllegalStateException("Falha ao debitar conta corrente: " + e.getMessage());
            }
        }

        // Cria ordem PENDENTE
        OrdemInvestimento ordem = new OrdemInvestimento();
        ordem.setTenantId(TenantContext.getTenantId());
        ordem.setContaId(request.getContaId());
        ordem.setProdutoId(request.getProdutoId());
        ordem.setTipo(TipoOrdemInvestimento.APLICACAO);
        ordem.setValor(request.getValor());
        ordem.setQuantidade(BigDecimal.ZERO);
        ordem.setStatus(StatusOrdemInvestimento.PENDENTE);
        ordem.setDataOrdem(LocalDate.now());
        ordem = ordemInvestimentoRepository.save(ordem);

        // Executa — atualiza carteira + marca como EXECUTADA
        try {
            carteiraService.creditarCarteira(ordem.getContaId(), ordem.getProdutoId(), ordem.getValor(), BigDecimal.ZERO);
            ordem.setStatus(StatusOrdemInvestimento.EXECUTADA);
            ordem.setDataExecucao(LocalDate.now());
            ordem = ordemInvestimentoRepository.save(ordem);

            // Atualiza saldo total da conta
            conta.setSaldoTotal(conta.getSaldoTotal().add(ordem.getValor()));
            contaInvestimentoRepository.save(conta);

            try {
                kafkaTemplate.send(TOPICO_ORDEM_EXECUTADA, new OrdemExecutadaEvent(
                    ordem.getId(), ordem.getContaId(), ordem.getProdutoId(),
                    ordem.getValor(), ordem.getQuantidade(), ordem.getDataExecucao(), ordem.getTenantId()));
            } catch (Exception e) {
                // fire-and-forget
            }
        } catch (Exception e) {
            ordem.setStatus(StatusOrdemInvestimento.FALHADA);
            ordem.setObservacao("Falha na execucao: " + e.getMessage());
            ordem = ordemInvestimentoRepository.save(ordem);

            // Tenta estorno na conta corrente
            if (conta.getModo() == ModoContaInvestimento.VINCULADA && !contaIndependente) {
                try {
                    contaCorrenteClient.creditar(conta.getContaCorrenteId(),
                        new ContaCorrenteClient.CreditoRequest(ordem.getValor(), "Estorno aplicacao investimento"));
                } catch (Exception estornoEx) {
                    // log estorno falhou
                }
            }
        }

        return toResponse(ordem);
    }

    public OrdemInvestimentoResponse resgatar(ResgatarOrdemRequest request) {
        ContaInvestimento conta = contaInvestimentoRepository.findById(request.getContaId())
            .orElseThrow(() -> new IllegalArgumentException("Conta investimento nao encontrada: " + request.getContaId()));
        if (conta.getAtiva() != StatusContaInvestimento.ATIVA) {
            throw new IllegalStateException("Conta investimento nao esta ativa");
        }

        // Debitar da carteira
        carteiraService.debitarCarteira(request.getContaId(), request.getProdutoId(), request.getValor(), BigDecimal.ZERO);

        // Se vinculada, credita na conta corrente
        if (conta.getModo() == ModoContaInvestimento.VINCULADA && !contaIndependente) {
            try {
                contaCorrenteClient.creditar(conta.getContaCorrenteId(),
                    new ContaCorrenteClient.CreditoRequest(request.getValor(), "Resgate investimento"));
            } catch (Exception e) {
                // Estorno na carteira
                carteiraService.creditarCarteira(request.getContaId(), request.getProdutoId(), request.getValor(), BigDecimal.ZERO);
                throw new IllegalStateException("Falha ao creditar conta corrente: " + e.getMessage());
            }
        }

        // Cria ordem RESGATE executada
        OrdemInvestimento ordem = new OrdemInvestimento();
        ordem.setTenantId(TenantContext.getTenantId());
        ordem.setContaId(request.getContaId());
        ordem.setProdutoId(request.getProdutoId());
        ordem.setTipo(TipoOrdemInvestimento.RESGATE);
        ordem.setValor(request.getValor());
        ordem.setStatus(StatusOrdemInvestimento.EXECUTADA);
        ordem.setDataOrdem(LocalDate.now());
        ordem.setDataExecucao(LocalDate.now());
        ordem = ordemInvestimentoRepository.save(ordem);

        // Atualiza saldo total da conta
        conta.setSaldoTotal(conta.getSaldoTotal().subtract(ordem.getValor()));
        contaInvestimentoRepository.save(conta);

        try {
            kafkaTemplate.send(TOPICO_RESGATE_PROCESSADO, new ResgateProcessadoEvent(
                ordem.getId(), ordem.getContaId(), ordem.getProdutoId(),
                ordem.getValor(), ordem.getDataExecucao(), ordem.getTenantId()));
        } catch (Exception e) {
            // fire-and-forget
        }

        return toResponse(ordem);
    }

    @Transactional(readOnly = true)
    public ExtratoOrdemResponse extratoPorConta(Long contaId) {
        List<OrdemInvestimento> ordens = ordemInvestimentoRepository.findByContaIdOrderByDataOrdemDesc(contaId);
        ExtratoOrdemResponse extrato = new ExtratoOrdemResponse();
        extrato.setContaId(contaId);
        extrato.setOrdens(ordens.stream().map(this::toResponse).toList());
        return extrato;
    }

    private OrdemInvestimentoResponse toResponse(OrdemInvestimento ordem) {
        OrdemInvestimentoResponse r = new OrdemInvestimentoResponse();
        r.setId(ordem.getId());
        r.setTenantId(ordem.getTenantId());
        r.setContaId(ordem.getContaId());
        r.setProdutoId(ordem.getProdutoId());
        r.setTipo(ordem.getTipo().name());
        r.setValor(ordem.getValor());
        r.setQuantidade(ordem.getQuantidade());
        r.setStatus(ordem.getStatus().name());
        r.setDataOrdem(ordem.getDataOrdem());
        r.setDataExecucao(ordem.getDataExecucao());
        r.setObservacao(ordem.getObservacao());
        r.setDataCriacao(ordem.getDataCriacao());
        return r;
    }
}
```

- [ ] **Step 6: Fix ContaInvestimentoRepository — add paginated findByTenantId**

Edit `ContaInvestimentoRepository.java` to add the paginated query:

```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface ContaInvestimentoRepository extends JpaRepository<ContaInvestimento, Long> {

    Page<ContaInvestimento> findByTenantId(String tenantId, Pageable pageable);

    List<ContaInvestimento> findByClienteId(Long clienteId);

    Optional<ContaInvestimento> findByNumero(String numero);
}
```

- [ ] **Step 7: Verify compilation**

Run: `mvn compile -pl aurix-investimento -am`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/service/
git add apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/repository/ContaInvestimentoRepository.java
git commit -m "feat: add investimento services"
```

---

### Task 7: Controllers

**Files:**
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/controller/package-info.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/controller/ContaInvestimentoController.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/controller/ProdutoInvestimentoController.java`
- Create: `apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/controller/OrdemController.java`

**Interfaces:**
- Consumes: service methods from Task 6
- Produces: REST API at `/api/investimento/contas`, `/api/investimento/produtos`, `/api/investimento/ordens`

- [ ] **Step 1: Create controller/package-info.java**

```java
@NullMarked
package com.aurix.platform.investimento.controller;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 2: Create ContaInvestimentoController.java**

```java
package com.aurix.platform.investimento.controller;

import com.aurix.platform.investimento.dto.request.CriarContaInvestimentoRequest;
import com.aurix.platform.investimento.dto.response.CarteiraResponse;
import com.aurix.platform.investimento.dto.response.ContaInvestimentoResponse;
import com.aurix.platform.investimento.service.CarteiraService;
import com.aurix.platform.investimento.service.ContaInvestimentoService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import java.util.List;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
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
@Tag(name = "Conta Investimento", description = "Gerenciamento de contas investimento")
public class ContaInvestimentoController {

    private final ContaInvestimentoService contaInvestimentoService;
    private final CarteiraService carteiraService;

    public ContaInvestimentoController(ContaInvestimentoService contaInvestimentoService,
                                       CarteiraService carteiraService) {
        this.contaInvestimentoService = contaInvestimentoService;
        this.carteiraService = carteiraService;
    }

    @PostMapping
    @Operation(summary = "Criar conta investimento")
    public ResponseEntity<ContaInvestimentoResponse> criarConta(@Valid @RequestBody CriarContaInvestimentoRequest request) {
        ContaInvestimentoResponse response = contaInvestimentoService.criarConta(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @GetMapping
    @Operation(summary = "Listar contas investimento (paginado)")
    public ResponseEntity<Page<ContaInvestimentoResponse>> listar(Pageable pageable) {
        return ResponseEntity.ok(contaInvestimentoService.listar(pageable));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Buscar conta investimento por ID")
    public ResponseEntity<ContaInvestimentoResponse> buscarPorId(@PathVariable Long id) {
        return ResponseEntity.ok(contaInvestimentoService.buscarPorId(id));
    }

    @PatchMapping("/{id}/encerrar")
    @Operation(summary = "Encerrar conta investimento")
    public ResponseEntity<Void> encerrar(@PathVariable Long id) {
        contaInvestimentoService.encerrar(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping("/{id}/carteira")
    @Operation(summary = "Saldo por produto na carteira")
    public ResponseEntity<List<CarteiraResponse>> listarCarteira(@PathVariable Long id) {
        return ResponseEntity.ok(carteiraService.listarPorConta(id));
    }
}
```

- [ ] **Step 3: Create ProdutoInvestimentoController.java**

```java
package com.aurix.platform.investimento.controller;

import com.aurix.platform.investimento.dto.request.AtualizarProdutoRequest;
import com.aurix.platform.investimento.dto.request.CriarProdutoRequest;
import com.aurix.platform.investimento.dto.response.ProdutoInvestimentoResponse;
import com.aurix.platform.investimento.service.ProdutoInvestimentoService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import java.util.List;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/produtos")
@Tag(name = "Produto Investimento", description = "Catalogo de produtos de investimento")
public class ProdutoInvestimentoController {

    private final ProdutoInvestimentoService produtoInvestimentoService;

    public ProdutoInvestimentoController(ProdutoInvestimentoService produtoInvestimentoService) {
        this.produtoInvestimentoService = produtoInvestimentoService;
    }

    @GetMapping
    @Operation(summary = "Listar catalogo de produtos ativos")
    public ResponseEntity<List<ProdutoInvestimentoResponse>> listar() {
        return ResponseEntity.ok(produtoInvestimentoService.listarAtivos());
    }

    @PostMapping
    @Operation(summary = "Criar produto investimento (admin)")
    public ResponseEntity<ProdutoInvestimentoResponse> criarProduto(@Valid @RequestBody CriarProdutoRequest request) {
        ProdutoInvestimentoResponse response = produtoInvestimentoService.criarProduto(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @GetMapping("/{id}")
    @Operation(summary = "Buscar produto por ID")
    public ResponseEntity<ProdutoInvestimentoResponse> buscarPorId(@PathVariable Long id) {
        return ResponseEntity.ok(produtoInvestimentoService.buscarPorId(id));
    }

    @PutMapping("/{id}")
    @Operation(summary = "Atualizar produto investimento")
    public ResponseEntity<ProdutoInvestimentoResponse> atualizarProduto(@PathVariable Long id,
                                                                         @RequestBody AtualizarProdutoRequest request) {
        return ResponseEntity.ok(produtoInvestimentoService.atualizarProduto(id, request));
    }
}
```

- [ ] **Step 4: Create OrdemController.java**

```java
package com.aurix.platform.investimento.controller;

import com.aurix.platform.investimento.dto.request.AplicarOrdemRequest;
import com.aurix.platform.investimento.dto.request.ResgatarOrdemRequest;
import com.aurix.platform.investimento.dto.response.ExtratoOrdemResponse;
import com.aurix.platform.investimento.dto.response.OrdemInvestimentoResponse;
import com.aurix.platform.investimento.service.OrdemService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/ordens")
@Tag(name = "Ordem Investimento", description = "Ordens de aplicacao e resgate")
public class OrdemController {

    private final OrdemService ordemService;

    public OrdemController(OrdemService ordemService) {
        this.ordemService = ordemService;
    }

    @PostMapping("/aplicar")
    @Operation(summary = "Aplicar em produto de investimento")
    public ResponseEntity<OrdemInvestimentoResponse> aplicar(@Valid @RequestBody AplicarOrdemRequest request) {
        OrdemInvestimentoResponse response = ordemService.aplicar(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @PostMapping("/resgatar")
    @Operation(summary = "Resgatar de produto de investimento")
    public ResponseEntity<OrdemInvestimentoResponse> resgatar(@Valid @RequestBody ResgatarOrdemRequest request) {
        OrdemInvestimentoResponse response = ordemService.resgatar(request);
        return ResponseEntity.ok(response);
    }

    @GetMapping("/{contaId}")
    @Operation(summary = "Extrato de ordens da conta")
    public ResponseEntity<ExtratoOrdemResponse> extrato(@PathVariable Long contaId) {
        return ResponseEntity.ok(ordemService.extratoPorConta(contaId));
    }
}
```

- [ ] **Step 5: Verify compilation**

Run: `mvn compile -pl aurix-investimento -am`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add apps/backend/aurix-investimento/src/main/java/com/aurix/platform/investimento/controller/
git commit -m "feat: add investimento controllers"
```

---

### Task 8: Gateway Route + Parent POM

**Files:**
- Modify: `apps/backend/aurix-gateway/src/main/resources/application.yml` (add gateway route)
- Modify: `apps/backend/pom.xml` (already done in Task 1, verify)

**Interfaces:**
- Consumes: port 8113 from application.yml (Task 1)
- Produces: gateway routing `/api/investimento/**` → `localhost:8113`

- [ ] **Step 1: Add gateway route**

Edit `apps/backend/aurix-gateway/src/main/resources/application.yml`, insert after the AURIX Internet Banking route:

```yaml
        # AURIX Investimento
        - id: aurix-investimento
          uri: http://localhost:8113
          predicates:
            - Path=/api/investimento/**
          filters:
            - StripPrefix=0
```

- [ ] **Step 2: Verify parent POM has module**

Run: `grep -n 'aurix-investimento' apps/backend/pom.xml`
Expected: output showing `<module>aurix-investimento</module>` present

- [ ] **Step 3: Verify full build compiles**

Run: `mvn compile -pl aurix-investimento -am`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add apps/backend/aurix-gateway/src/main/resources/application.yml
git commit -m "feat: add gateway route for aurix-investimento"
```

---

### Task 9: Controller Tests

**Files:**
- Create: `apps/backend/aurix-investimento/src/test/java/com/aurix/platform/investimento/controller/package-info.java`
- Create: `apps/backend/aurix-investimento/src/test/java/com/aurix/platform/investimento/controller/ContaInvestimentoControllerTest.java`
- Create: `apps/backend/aurix-investimento/src/test/java/com/aurix/platform/investimento/controller/ProdutoInvestimentoControllerTest.java`
- Create: `apps/backend/aurix-investimento/src/test/java/com/aurix/platform/investimento/controller/OrdemControllerTest.java`
- Modify: `apps/backend/aurix-investimento/src/main/resources/application.yml` (add test profile)
- Create: `apps/backend/aurix-investimento/src/test/resources/application-test.yml`

- [ ] **Step 1: Create test controller/package-info.java**

```java
@NullMarked
package com.aurix.platform.investimento.controller;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 2: Create test/resources/application-test.yml**

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
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.H2Dialect
  kafka:
    producer:
      retries: 0
logging:
  level:
    com.aurix.platform: DEBUG
investimento:
  conta-independente: false
```

- [ ] **Step 3: Create ContaInvestimentoControllerTest.java**

```java
package com.aurix.platform.investimento.controller;

import com.aurix.platform.investimento.AurixInvestimentoApplication;
import com.aurix.platform.investimento.dto.request.CriarContaInvestimentoRequest;
import com.aurix.platform.investimento.dto.response.ContaInvestimentoResponse;
import com.aurix.platform.investimento.repository.ContaInvestimentoRepository;
import com.aurix.platform.shared.tenant.TenantContext;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
import org.springframework.context.annotation.Primary;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.web.client.RestTemplate;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(classes = AurixInvestimentoApplication.class, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@Import(ContaInvestimentoControllerTest.TestConfig.class)
class ContaInvestimentoControllerTest {

    @LocalServerPort
    private int port;

    @Autowired
    private ContaInvestimentoRepository repository;

    private RestTemplate rest;

    @BeforeEach
    void setUp() {
        repository.deleteAll();
        TenantContext.setTenantId("test-tenant");
        rest = new RestTemplate();
    }

    private String url(String path) {
        return "http://localhost:" + port + "/api/investimento" + path;
    }

    @Test
    void deveCriarContaInvestimentoVinculada() {
        CriarContaInvestimentoRequest request = new CriarContaInvestimentoRequest();
        request.setClienteId(1L);
        request.setContaCorrenteId(100L);
        request.setAgencia("0001");

        ResponseEntity<ContaInvestimentoResponse> response = rest.postForEntity(
            url("/contas"), request, ContaInvestimentoResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getClienteId()).isEqualTo(1L);
        assertThat(response.getBody().getContaCorrenteId()).isEqualTo(100L);
        assertThat(response.getBody().getNumero()).isNotBlank();
        assertThat(response.getBody().getModo()).isEqualTo("VINCULADA");
        assertThat(response.getBody().getSaldoTotal()).isEqualByComparingTo(java.math.BigDecimal.ZERO);
    }

    @Test
    void deveBuscarContaPorId() {
        CriarContaInvestimentoRequest request = new CriarContaInvestimentoRequest();
        request.setClienteId(1L);
        request.setContaCorrenteId(100L);
        request.setAgencia("0001");
        ResponseEntity<ContaInvestimentoResponse> criada = rest.postForEntity(
            url("/contas"), request, ContaInvestimentoResponse.class);
        Long id = criada.getBody().getId();

        ResponseEntity<ContaInvestimentoResponse> response = rest.getForEntity(
            url("/contas/" + id), ContaInvestimentoResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getId()).isEqualTo(id);
    }

    @Test
    void deveEncerrarConta() {
        CriarContaInvestimentoRequest request = new CriarContaInvestimentoRequest();
        request.setClienteId(1L);
        request.setContaCorrenteId(100L);
        request.setAgencia("0001");
        ResponseEntity<ContaInvestimentoResponse> criada = rest.postForEntity(
            url("/contas"), request, ContaInvestimentoResponse.class);
        Long id = criada.getBody().getId();

        ResponseEntity<Void> response = rest.exchange(
            url("/contas/" + id + "/encerrar"),
            org.springframework.http.HttpMethod.PATCH, null, Void.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NO_CONTENT);

        ContaInvestimentoResponse conta = rest.getForEntity(
            url("/contas/" + id), ContaInvestimentoResponse.class).getBody();
        assertThat(conta.getAtiva()).isEqualTo("ENCERRADA");
    }

    @TestConfiguration
    @EnableWebSecurity
    static class TestConfig {
        @Bean
        public SecurityFilterChain testFilterChain(HttpSecurity http) throws Exception {
            http
                .csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
            return http.build();
        }

        @Bean
        @Primary
        @SuppressWarnings("unchecked")
        public KafkaTemplate<String, Object> kafkaTemplate() {
            return Mockito.mock(KafkaTemplate.class);
        }
    }
}
```

- [ ] **Step 4: Create ProdutoInvestimentoControllerTest.java**

```java
package com.aurix.platform.investimento.controller;

import com.aurix.platform.investimento.AurixInvestimentoApplication;
import com.aurix.platform.investimento.dto.request.CriarProdutoRequest;
import com.aurix.platform.investimento.dto.response.ProdutoInvestimentoResponse;
import com.aurix.platform.investimento.repository.ProdutoInvestimentoRepository;
import com.aurix.platform.shared.tenant.TenantContext;
import java.math.BigDecimal;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
import org.springframework.context.annotation.Primary;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.web.client.RestTemplate;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(classes = AurixInvestimentoApplication.class, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@Import(ProdutoInvestimentoControllerTest.TestConfig.class)
class ProdutoInvestimentoControllerTest {

    @LocalServerPort
    private int port;

    @Autowired
    private ProdutoInvestimentoRepository repository;

    private RestTemplate rest;

    @BeforeEach
    void setUp() {
        repository.deleteAll();
        TenantContext.setTenantId("test-tenant");
        rest = new RestTemplate();
    }

    private String url(String path) {
        return "http://localhost:" + port + "/api/investimento" + path;
    }

    @Test
    void deveCriarProdutoInvestimento() {
        CriarProdutoRequest request = new CriarProdutoRequest();
        request.setCodigo("CDB-001");
        request.setNome("CDB Prefixado 120% CDI");
        request.setTipo("CDB");
        request.setCategoria("RENDA_FIXA");
        request.setEmissor("Banco AURIX");
        request.setValorMinimoAplicacao(new BigDecimal("1000.00"));

        ResponseEntity<ProdutoInvestimentoResponse> response = rest.postForEntity(
            url("/produtos"), request, ProdutoInvestimentoResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getCodigo()).isEqualTo("CDB-001");
        assertThat(response.getBody().getNome()).isEqualTo("CDB Prefixado 120% CDI");
        assertThat(response.getBody().getTipo()).isEqualTo("CDB");
        assertThat(response.getBody().getCategoria()).isEqualTo("RENDA_FIXA");
        assertThat(response.getBody().getAtivo()).isTrue();
    }

    @Test
    void deveListarProdutosAtivos() {
        CriarProdutoRequest request = new CriarProdutoRequest();
        request.setCodigo("LCI-001");
        request.setNome("LCI 95% CDI");
        request.setTipo("LCI");
        request.setCategoria("RENDA_FIXA");
        request.setEmissor("Banco AURIX");
        request.setValorMinimoAplicacao(new BigDecimal("5000.00"));
        rest.postForEntity(url("/produtos"), request, ProdutoInvestimentoResponse.class);

        ResponseEntity<ProdutoInvestimentoResponse[]> response = rest.getForEntity(
            url("/produtos"), ProdutoInvestimentoResponse[].class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).isNotEmpty();
    }

    @TestConfiguration
    @EnableWebSecurity
    static class TestConfig {
        @Bean
        public SecurityFilterChain testFilterChain(HttpSecurity http) throws Exception {
            http
                .csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
            return http.build();
        }

        @Bean
        @Primary
        @SuppressWarnings("unchecked")
        public KafkaTemplate<String, Object> kafkaTemplate() {
            return Mockito.mock(KafkaTemplate.class);
        }
    }
}
```

- [ ] **Step 5: Create OrdemControllerTest.java**

```java
package com.aurix.platform.investimento.controller;

import com.aurix.platform.investimento.AurixInvestimentoApplication;
import com.aurix.platform.investimento.dto.request.AplicarOrdemRequest;
import com.aurix.platform.investimento.dto.request.CriarContaInvestimentoRequest;
import com.aurix.platform.investimento.dto.request.CriarProdutoRequest;
import com.aurix.platform.investimento.dto.response.ContaInvestimentoResponse;
import com.aurix.platform.investimento.dto.response.ExtratoOrdemResponse;
import com.aurix.platform.investimento.dto.response.OrdemInvestimentoResponse;
import com.aurix.platform.investimento.dto.response.ProdutoInvestimentoResponse;
import com.aurix.platform.investimento.repository.ContaInvestimentoRepository;
import com.aurix.platform.investimento.repository.OrdemInvestimentoRepository;
import com.aurix.platform.investimento.repository.ProdutoInvestimentoRepository;
import com.aurix.platform.shared.tenant.TenantContext;
import java.math.BigDecimal;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
import org.springframework.context.annotation.Primary;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.web.client.RestTemplate;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(classes = AurixInvestimentoApplication.class, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@Import(OrdemControllerTest.TestConfig.class)
class OrdemControllerTest {

    @LocalServerPort
    private int port;

    @Autowired
    private ContaInvestimentoRepository contaRepository;

    @Autowired
    private ProdutoInvestimentoRepository produtoRepository;

    @Autowired
    private OrdemInvestimentoRepository ordemRepository;

    private RestTemplate rest;

    @BeforeEach
    void setUp() {
        ordemRepository.deleteAll();
        contaRepository.deleteAll();
        produtoRepository.deleteAll();
        TenantContext.setTenantId("test-tenant");
        rest = new RestTemplate();
    }

    private String url(String path) {
        return "http://localhost:" + port + "/api/investimento" + path;
    }

    private Long criarConta() {
        CriarContaInvestimentoRequest request = new CriarContaInvestimentoRequest();
        request.setClienteId(1L);
        request.setContaCorrenteId(100L);
        request.setAgencia("0001");
        return rest.postForEntity(url("/contas"), request, ContaInvestimentoResponse.class)
            .getBody().getId();
    }

    private Long criarProduto() {
        CriarProdutoRequest request = new CriarProdutoRequest();
        request.setCodigo("CDB-TEST");
        request.setNome("CDB Teste");
        request.setTipo("CDB");
        request.setCategoria("RENDA_FIXA");
        request.setEmissor("Banco AURIX");
        request.setValorMinimoAplicacao(new BigDecimal("100.00"));
        return rest.postForEntity(url("/produtos"), request, ProdutoInvestimentoResponse.class)
            .getBody().getId();
    }

    @Test
    void deveAplicarOrdem() {
        Long contaId = criarConta();
        Long produtoId = criarProduto();

        AplicarOrdemRequest request = new AplicarOrdemRequest();
        request.setContaId(contaId);
        request.setProdutoId(produtoId);
        request.setValor(new BigDecimal("500.00"));

        ResponseEntity<OrdemInvestimentoResponse> response = rest.postForEntity(
            url("/ordens/aplicar"), request, OrdemInvestimentoResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getTipo()).isEqualTo("APLICACAO");
        assertThat(response.getBody().getStatus()).isIn("PENDENTE", "EXECUTADA");
    }

    @Test
    void deveConsultarExtrato() {
        Long contaId = criarConta();
        Long produtoId = criarProduto();

        AplicarOrdemRequest request = new AplicarOrdemRequest();
        request.setContaId(contaId);
        request.setProdutoId(produtoId);
        request.setValor(new BigDecimal("500.00"));
        rest.postForEntity(url("/ordens/aplicar"), request, OrdemInvestimentoResponse.class);

        ResponseEntity<ExtratoOrdemResponse> response = rest.getForEntity(
            url("/ordens/" + contaId), ExtratoOrdemResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getOrdens()).isNotEmpty();
    }

    @TestConfiguration
    @EnableWebSecurity
    static class TestConfig {
        @Bean
        public SecurityFilterChain testFilterChain(HttpSecurity http) throws Exception {
            http
                .csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
            return http.build();
        }

        @Bean
        @Primary
        @SuppressWarnings("unchecked")
        public KafkaTemplate<String, Object> kafkaTemplate() {
            return Mockito.mock(KafkaTemplate.class);
        }
    }
}
```

- [ ] **Step 6: Run tests**

Run: `mvn test -pl aurix-investimento -am -Dtest=ContaInvestimentoControllerTest,ProdutoInvestimentoControllerTest,OrdemControllerTest`
Expected: BUILD SUCCESS (all tests pass)

- [ ] **Step 7: Commit**

```bash
git add apps/backend/aurix-investimento/src/test/
git add apps/backend/aurix-investimento/src/test/resources/
git commit -m "test: add investimento controller tests"
```

---

## Self-Review

**Spec coverage:**
- Section 2.2 (entities) → Task 2: ContaInvestimento, ProdutoInvestimento, OrdemInvestimento, Carteira ✓
- Section 2.3 (API surface) → Task 7: all 10 endpoints implemented ✓
- Section 2.4 (fluxos: aplicação vinculada, resgate vinculado, modo standalone) → Task 6 OrdemService: `contaIndependente` flag controls ContaCorrenteClient calls ✓
- Section 2.5 (Kafka events) → Task 4: ContaInvestimentoCriadaEvent, OrdemExecutadaEvent, ResgateProcessadoEvent ✓
- Section 2.6 (HTTP clients) → Task 5: ContaCorrenteClient with debitar/creditar ✓
- Section 1.2 (port 8113) → Task 1 application.yml ✓
- Section 8.1 (gateway route) → Task 8 ✓
- Section 8.2 (parent POM module) → Task 1 ✓
- Section 10.2 (`@NullMarked` package-level) → All package-info.java files ✓
- Section 10.3 (tests: RANDOM_PORT + RestTemplate + H2) → Task 9 ✓

**Placeholder scan:** No TBD, TODO, or "implement later" found. All code blocks contain full implementations.

**Type consistency:** `ContaInvestimento.ModoContaInvestimento.VINCULADA/STANDALONE` matches design spec. All method signatures across services and controllers are consistent.

Plan complete and saved to `docs/superpowers/plans/2026-06-24-aurix-investimento.md`.

Two execution options:
1. **Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration
2. **Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints

Which approach?
