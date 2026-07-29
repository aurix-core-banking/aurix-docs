# AURIX Conta Salário Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `aurix-salario` module — dedicated salary account management with CNAB 240 payroll processing, employer agreements, portability, and batch credit jobs.

**Architecture:** Dedicated Spring Boot module (port 8112, route `/api/salario/**`, StripPrefix=0). 5 entities, 5 services, 4 controllers, CNAB parser, @Scheduled job. Same patterns as `aurix-poupanca`: no Lombok, no Feign, `@HttpExchange`, `@NullMarked`, Kafka fire-and-forget.

**Tech Stack:** Java 25, Spring Boot 4.1.0, Spring Cloud 2025.1.2, JPA, PostgreSQL, Kafka, Redis, Testcontainers, H2 (tests).

## Global Constraints

- No Lombok — manual constructors/getters/setters
- No Feign — use `@HttpExchange` + `@ImportHttpServices`
- No spring-retry — use native `@Retryable` from `org.springframework.resilience.annotation`
- `@NullMarked` on all packages (`org.jspecify.annotations`)
- `jakarta.*` packages, not `javax.*`
- Kafka fire-and-forget with try-catch (never roll back operations)
- Tests: `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `RestTemplate`
- Gateway: StripPrefix=0
- Port: 8112, context-path: `/api/salario`

---

### Task 1: Module Scaffold

**Files:**
- Modify: `backend/pom.xml` (add `<module>aurix-salario</module>` in `<modules>`)
- Create: `backend/aurix-salario/pom.xml`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/AurixSalarioApplication.java`
- Create: `backend/aurix-salario/src/main/resources/application.yml`
- Create: `backend/aurix-salario/src/main/resources/application-prod.yml`
- Modify: `backend/aurix-gateway/src/main/resources/application.yml` (add route for salario)

- [ ] **Step 1: Create `backend/aurix-salario/pom.xml`**

Copied from `aurix-poupanca/pom.xml`, replace `aurix-poupanca` with `aurix-salario`, name "AURIX Salario", description "Modulo de conta salario do AURIX". Add `spring-boot-starter-batch` dependency for CNAB batch processing.

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

    <artifactId>aurix-salario</artifactId>
    <packaging>jar</packaging>

    <name>AURIX Salario</name>
    <description>Modulo de conta salario do AURIX</description>

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

- [ ] **Step 2: Create `AurixSalarioApplication.java`**

Standard Spring Boot app with `@EnableScheduling`:

```java
package com.aurix.platform.salario;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class AurixSalarioApplication {

    public static void main(String[] args) {
        SpringApplication.run(AurixSalarioApplication.class, args);
    }
}
```

- [ ] **Step 3: Create `application.yml`**

Same pattern as poupanca, port 8112, context-path `/api/salario`:

```yaml
server:
  port: 8112
  servlet:
    context-path: /api/salario

spring:
  application:
    name: aurix-salario
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
      group-id: aurix-salario-group
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
  salario:
    cnab:
      dir-upload: ./data/cnab
      max-file-size: 10MB
    limits:
      max-salario: 50000.00
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

- [ ] **Step 4: Create `application-prod.yml`**

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

- [ ] **Step 5: Add module to parent `backend/pom.xml`**

Add after `<module>aurix-poupanca</module>`:
```xml
<module>aurix-salario</module>
```

- [ ] **Step 6: Add gateway route in `backend/aurix-gateway/src/main/resources/application.yml`**

Add after the poupanca route:
```yaml
# AURIX Salario
- id: aurix-salario
  uri: http://localhost:8112
  predicates:
    - Path=/api/salario/**
  filters:
    - StripPrefix=0
```

- [ ] **Step 7: Verify compilation**

Run: `mvn compile -pl aurix-salario -am -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add backend/pom.xml backend/aurix-salario/ backend/aurix-gateway/src/main/resources/application.yml
git commit -m "feat(salario): scaffold module with pom, app, config, gateway route"
```

---

### Task 2: Domain Layer

**Files:**
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/entity/ContaSalario.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/entity/ConvenioEmpresa.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/entity/FolhaPagamento.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/entity/ItemFolhaPagamento.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/entity/SolicitacaoPortabilidade.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/repository/ContaSalarioRepository.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/repository/ConvenioEmpresaRepository.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/repository/FolhaPagamentoRepository.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/repository/ItemFolhaPagamentoRepository.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/repository/SolicitacaoPortabilidadeRepository.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/dto/ContaSalarioRequest.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/dto/ContaSalarioResponse.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/dto/ConvenioRequest.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/dto/ConvenioResponse.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/dto/PortabilidadeRequest.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/dto/PortabilidadeResponse.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/dto/FolhaResponse.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/dto/ItemFolhaResponse.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/dto/CreditoDiretoRequest.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/event/ContaSalarioCriadaEvent.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/event/SalarioCreditadoEvent.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/event/PortabilidadeSolicitadaEvent.java`
- Create: `package-info.java` in each package (entity, repository, dto, event, client, service, controller, config, job)

- [ ] **Step 1: Create `entity/ContaSalario.java`**

```java
package com.aurix.platform.salario.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Table;
import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;
import java.time.LocalDate;

@Entity
@Table(name = "contas_salario", schema = "aurix")
public class ContaSalario extends BaseEntity {

    @NotNull
    @Column(name = "conta_corrente_id", nullable = false)
    private Long contaCorrenteId;

    @NotNull
    @Column(name = "empresa_id", nullable = false)
    private Long empresaId;

    @NotBlank
    @Column(name = "matricula_funcionario", nullable = false, length = 50)
    private String matriculaFuncionario;

    @NotNull
    @Column(name = "data_admissao", nullable = false)
    private LocalDate dataAdmissao;

    @Column(name = "data_rescisao")
    private LocalDate dataRescisao;

    @NotNull
    @DecimalMin("0.01")
    @Column(name = "valor_salario_bruto", nullable = false, precision = 15, scale = 2)
    private BigDecimal valorSalarioBruto;

    @NotNull
    @DecimalMin("0.01")
    @Column(name = "valor_salario_liquido", nullable = false, precision = 15, scale = 2)
    private BigDecimal valorSalarioLiquido;

    @NotNull
    @Min(1)
    @Max(31)
    @Column(name = "dia_pagamento", nullable = false)
    private Integer diaPagamento;

    @Column(name = "portabilidade_ativa", nullable = false)
    private Boolean portabilidadeAtiva = false;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private StatusContaSalario status = StatusContaSalario.ATIVA;

    public enum StatusContaSalario {
        ATIVA, RESCINDIDA, BLOQUEADA
    }

    public ContaSalario() {}

    public ContaSalario(Long contaCorrenteId, Long empresaId, String matriculaFuncionario,
                        LocalDate dataAdmissao, BigDecimal valorSalarioBruto,
                        BigDecimal valorSalarioLiquido, Integer diaPagamento) {
        this.contaCorrenteId = contaCorrenteId;
        this.empresaId = empresaId;
        this.matriculaFuncionario = matriculaFuncionario;
        this.dataAdmissao = dataAdmissao;
        this.valorSalarioBruto = valorSalarioBruto;
        this.valorSalarioLiquido = valorSalarioLiquido;
        this.diaPagamento = diaPagamento;
    }

    public Long getContaCorrenteId() { return contaCorrenteId; }
    public void setContaCorrenteId(Long contaCorrenteId) { this.contaCorrenteId = contaCorrenteId; }
    public Long getEmpresaId() { return empresaId; }
    public void setEmpresaId(Long empresaId) { this.empresaId = empresaId; }
    public String getMatriculaFuncionario() { return matriculaFuncionario; }
    public void setMatriculaFuncionario(String matriculaFuncionario) { this.matriculaFuncionario = matriculaFuncionario; }
    public LocalDate getDataAdmissao() { return dataAdmissao; }
    public void setDataAdmissao(LocalDate dataAdmissao) { this.dataAdmissao = dataAdmissao; }
    public LocalDate getDataRescisao() { return dataRescisao; }
    public void setDataRescisao(LocalDate dataRescisao) { this.dataRescisao = dataRescisao; }
    public BigDecimal getValorSalarioBruto() { return valorSalarioBruto; }
    public void setValorSalarioBruto(BigDecimal valorSalarioBruto) { this.valorSalarioBruto = valorSalarioBruto; }
    public BigDecimal getValorSalarioLiquido() { return valorSalarioLiquido; }
    public void setValorSalarioLiquido(BigDecimal valorSalarioLiquido) { this.valorSalarioLiquido = valorSalarioLiquido; }
    public Integer getDiaPagamento() { return diaPagamento; }
    public void setDiaPagamento(Integer diaPagamento) { this.diaPagamento = diaPagamento; }
    public Boolean getPortabilidadeAtiva() { return portabilidadeAtiva; }
    public void setPortabilidadeAtiva(Boolean portabilidadeAtiva) { this.portabilidadeAtiva = portabilidadeAtiva; }
    public StatusContaSalario getStatus() { return status; }
    public void setStatus(StatusContaSalario status) { this.status = status; }
}
```

- [ ] **Step 2: Create `entity/ConvenioEmpresa.java`**

```java
package com.aurix.platform.salario.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

@Entity
@Table(name = "convenios_empresa", schema = "aurix")
public class ConvenioEmpresa extends BaseEntity {

    @NotBlank
    @Column(name = "cnpj", nullable = false, length = 14)
    private String cnpj;

    @NotBlank
    @Column(name = "razao_social", nullable = false, length = 200)
    private String razaoSocial;

    @NotNull
    @Column(name = "conta_corrente_id", nullable = false)
    private Long contaCorrenteId;

    @Column(name = "ativo", nullable = false)
    private Boolean ativo = true;

    public ConvenioEmpresa() {}

    public ConvenioEmpresa(String cnpj, String razaoSocial, Long contaCorrenteId) {
        this.cnpj = cnpj;
        this.razaoSocial = razaoSocial;
        this.contaCorrenteId = contaCorrenteId;
    }

    public String getCnpj() { return cnpj; }
    public void setCnpj(String cnpj) { this.cnpj = cnpj; }
    public String getRazaoSocial() { return razaoSocial; }
    public void setRazaoSocial(String razaoSocial) { this.razaoSocial = razaoSocial; }
    public Long getContaCorrenteId() { return contaCorrenteId; }
    public void setContaCorrenteId(Long contaCorrenteId) { this.contaCorrenteId = contaCorrenteId; }
    public Boolean getAtivo() { return ativo; }
    public void setAtivo(Boolean ativo) { this.ativo = ativo; }
}
```

- [ ] **Step 3: Create `entity/FolhaPagamento.java`**

```java
package com.aurix.platform.salario.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Table;
import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "folhas_pagamento", schema = "aurix")
public class FolhaPagamento extends BaseEntity {

    @NotNull
    @Column(name = "empresa_id", nullable = false)
    private Long empresaId;

    @NotNull
    @Column(name = "arquivo_nome", nullable = false, length = 255)
    private String arquivoNome;

    @NotNull
    @Column(name = "total_funcionarios", nullable = false)
    private Integer totalFuncionarios;

    @NotNull
    @DecimalMin("0.01")
    @Column(name = "valor_total", nullable = false, precision = 15, scale = 2)
    private BigDecimal valorTotal;

    @NotNull
    @Column(name = "data_referencia", nullable = false)
    private LocalDate dataReferencia;

    @NotNull
    @Column(name = "data_processamento", nullable = false)
    private LocalDateTime dataProcessamento = LocalDateTime.now();

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private StatusFolha status = StatusFolha.RECEBIDO;

    public enum StatusFolha {
        RECEBIDO, VALIDADO, PROCESSADO, ERRO_ESTRUTURA
    }

    public FolhaPagamento() {}

    public FolhaPagamento(Long empresaId, String arquivoNome, Integer totalFuncionarios,
                          BigDecimal valorTotal, LocalDate dataReferencia) {
        this.empresaId = empresaId;
        this.arquivoNome = arquivoNome;
        this.totalFuncionarios = totalFuncionarios;
        this.valorTotal = valorTotal;
        this.dataReferencia = dataReferencia;
    }

    public Long getEmpresaId() { return empresaId; }
    public void setEmpresaId(Long empresaId) { this.empresaId = empresaId; }
    public String getArquivoNome() { return arquivoNome; }
    public void setArquivoNome(String arquivoNome) { this.arquivoNome = arquivoNome; }
    public Integer getTotalFuncionarios() { return totalFuncionarios; }
    public void setTotalFuncionarios(Integer totalFuncionarios) { this.totalFuncionarios = totalFuncionarios; }
    public BigDecimal getValorTotal() { return valorTotal; }
    public void setValorTotal(BigDecimal valorTotal) { this.valorTotal = valorTotal; }
    public LocalDate getDataReferencia() { return dataReferencia; }
    public void setDataReferencia(LocalDate dataReferencia) { this.dataReferencia = dataReferencia; }
    public LocalDateTime getDataProcessamento() { return dataProcessamento; }
    public void setDataProcessamento(LocalDateTime dataProcessamento) { this.dataProcessamento = dataProcessamento; }
    public StatusFolha getStatus() { return status; }
    public void setStatus(StatusFolha status) { this.status = status; }
}
```

- [ ] **Step 4: Create `entity/ItemFolhaPagamento.java`**

```java
package com.aurix.platform.salario.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Table;
import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;

@Entity
@Table(name = "itens_folha_pagamento", schema = "aurix")
public class ItemFolhaPagamento extends BaseEntity {

    @NotNull
    @Column(name = "folha_id", nullable = false)
    private Long folhaId;

    @NotNull
    @Column(name = "conta_salario_id", nullable = false)
    private Long contaSalarioId;

    @NotBlank
    @Column(name = "cpf_funcionario", nullable = false, length = 11)
    private String cpfFuncionario;

    @NotNull
    @DecimalMin("0.01")
    @Column(name = "valor_liquido", nullable = false, precision = 15, scale = 2)
    private BigDecimal valorLiquido;

    @Column(name = "descontos", columnDefinition = "jsonb")
    private String descontos;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private StatusItem status = StatusItem.PENDENTE;

    public enum StatusItem {
        PENDENTE, CREDITADO, PORTADO, ERRO
    }

    public ItemFolhaPagamento() {}

    public ItemFolhaPagamento(Long folhaId, Long contaSalarioId, String cpfFuncionario,
                              BigDecimal valorLiquido) {
        this.folhaId = folhaId;
        this.contaSalarioId = contaSalarioId;
        this.cpfFuncionario = cpfFuncionario;
        this.valorLiquido = valorLiquido;
    }

    public Long getFolhaId() { return folhaId; }
    public void setFolhaId(Long folhaId) { this.folhaId = folhaId; }
    public Long getContaSalarioId() { return contaSalarioId; }
    public void setContaSalarioId(Long contaSalarioId) { this.contaSalarioId = contaSalarioId; }
    public String getCpfFuncionario() { return cpfFuncionario; }
    public void setCpfFuncionario(String cpfFuncionario) { this.cpfFuncionario = cpfFuncionario; }
    public BigDecimal getValorLiquido() { return valorLiquido; }
    public void setValorLiquido(BigDecimal valorLiquido) { this.valorLiquido = valorLiquido; }
    public String getDescontos() { return descontos; }
    public void setDescontos(String descontos) { this.descontos = descontos; }
    public StatusItem getStatus() { return status; }
    public void setStatus(StatusItem status) { this.status = status; }
}
```

- [ ] **Step 5: Create `entity/SolicitacaoPortabilidade.java`**

```java
package com.aurix.platform.salario.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Table;
import jakarta.validation.constraints.DecimalMax;
import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "solicitacoes_portabilidade", schema = "aurix")
public class SolicitacaoPortabilidade extends BaseEntity {

    @NotNull
    @Column(name = "conta_salario_id", nullable = false)
    private Long contaSalarioId;

    @NotBlank
    @Column(name = "codigo_banco_destino", nullable = false, length = 3)
    private String codigoBancoDestino;

    @NotBlank
    @Column(name = "agencia_destino", nullable = false, length = 10)
    private String agenciaDestino;

    @NotBlank
    @Column(name = "conta_destino", nullable = false, length = 20)
    private String contaDestino;

    @DecimalMin("0.01")
    @DecimalMax("100.00")
    @Column(name = "valor_percentual", nullable = false, precision = 5, scale = 2)
    private BigDecimal valorPercentual = new BigDecimal("100.00");

    @NotNull
    @Column(name = "data_solicitacao", nullable = false)
    private LocalDateTime dataSolicitacao = LocalDateTime.now();

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private StatusPortabilidade status = StatusPortabilidade.PENDENTE;

    public enum StatusPortabilidade {
        PENDENTE, ATIVA, CANCELADA
    }

    public SolicitacaoPortabilidade() {}

    public SolicitacaoPortabilidade(Long contaSalarioId, String codigoBancoDestino,
                                    String agenciaDestino, String contaDestino) {
        this.contaSalarioId = contaSalarioId;
        this.codigoBancoDestino = codigoBancoDestino;
        this.agenciaDestino = agenciaDestino;
        this.contaDestino = contaDestino;
    }

    public Long getContaSalarioId() { return contaSalarioId; }
    public void setContaSalarioId(Long contaSalarioId) { this.contaSalarioId = contaSalarioId; }
    public String getCodigoBancoDestino() { return codigoBancoDestino; }
    public void setCodigoBancoDestino(String codigoBancoDestino) { this.codigoBancoDestino = codigoBancoDestino; }
    public String getAgenciaDestino() { return agenciaDestino; }
    public void setAgenciaDestino(String agenciaDestino) { this.agenciaDestino = agenciaDestino; }
    public String getContaDestino() { return contaDestino; }
    public void setContaDestino(String contaDestino) { this.contaDestino = contaDestino; }
    public BigDecimal getValorPercentual() { return valorPercentual; }
    public void setValorPercentual(BigDecimal valorPercentual) { this.valorPercentual = valorPercentual; }
    public LocalDateTime getDataSolicitacao() { return dataSolicitacao; }
    public void setDataSolicitacao(LocalDateTime dataSolicitacao) { this.dataSolicitacao = dataSolicitacao; }
    public StatusPortabilidade getStatus() { return status; }
    public void setStatus(StatusPortabilidade status) { this.status = status; }
}
```

- [ ] **Step 6: Create 5 repositories**

`ContaSalarioRepository.java`:
```java
package com.aurix.platform.salario.repository;

import com.aurix.platform.salario.entity.ContaSalario;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;
import java.util.Optional;

@Repository
public interface ContaSalarioRepository extends JpaRepository<ContaSalario, Long> {
    List<ContaSalario> findByTenantIdAndClienteId(String tenantId, Long clienteId);
    List<ContaSalario> findByTenantIdAndEmpresaId(String tenantId, Long empresaId);
    Optional<ContaSalario> findByTenantIdAndId(String tenantId, Long id);
    Optional<ContaSalario> findByTenantIdAndEmpresaIdAndMatriculaFuncionario(
        String tenantId, Long empresaId, String matriculaFuncionario);
}
```

`ConvenioEmpresaRepository.java`:
```java
package com.aurix.platform.salario.repository;

import com.aurix.platform.salario.entity.ConvenioEmpresa;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;
import java.util.Optional;

@Repository
public interface ConvenioEmpresaRepository extends JpaRepository<ConvenioEmpresa, Long> {
    List<ConvenioEmpresa> findByTenantId(String tenantId);
    Optional<ConvenioEmpresa> findByTenantIdAndId(String tenantId, Long id);
    Optional<ConvenioEmpresa> findByTenantIdAndCnpj(String tenantId, String cnpj);
}
```

`FolhaPagamentoRepository.java`:
```java
package com.aurix.platform.salario.repository;

import com.aurix.platform.salario.entity.FolhaPagamento;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface FolhaPagamentoRepository extends JpaRepository<FolhaPagamento, Long> {
    List<FolhaPagamento> findByTenantIdAndStatus(
        String tenantId, FolhaPagamento.StatusFolha status);
    List<FolhaPagamento> findByTenantIdAndEmpresaId(
        String tenantId, Long empresaId);
}
```

`ItemFolhaPagamentoRepository.java`:
```java
package com.aurix.platform.salario.repository;

import com.aurix.platform.salario.entity.ItemFolhaPagamento;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface ItemFolhaPagamentoRepository extends JpaRepository<ItemFolhaPagamento, Long> {
    List<ItemFolhaPagamento> findByFolhaId(Long folhaId);
    List<ItemFolhaPagamento> findByStatus(ItemFolhaPagamento.StatusItem status);
}
```

`SolicitacaoPortabilidadeRepository.java`:
```java
package com.aurix.platform.salario.repository;

import com.aurix.platform.salario.entity.SolicitacaoPortabilidade;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface SolicitacaoPortabilidadeRepository extends JpaRepository<SolicitacaoPortabilidade, Long> {
    List<SolicitacaoPortabilidade> findByContaSalarioId(Long contaSalarioId);
    List<SolicitacaoPortabilidade> findByContaSalarioIdAndStatus(
        Long contaSalarioId, SolicitacaoPortabilidade.StatusPortabilidade status);
}
```

Note: The repositories use `clienteId` as a query parameter for ContaSalarioRepository, but the entity doesn't have a `clienteId` field directly — it's derived through the `Conta` entity in the core module. For now, `findByTenantIdAndClienteId` is not practical since `ContaSalario` doesn't have a `clienteId` column directly. Instead, searching by tenant is used. The client lookup can be done by the service layer calling `ContaCorrenteClient`. **Correction**: remove `findByTenantIdAndClienteId` from ContaSalarioRepository since ContaSalario doesn't have a clienteId field — client association is always through contaCorrenteId via the core module.

- [ ] **Step 7: Create DTOs (9 files)**

`ContaSalarioRequest.java`:
```java
package com.aurix.platform.salario.dto;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;
import java.time.LocalDate;

public class ContaSalarioRequest {
    @NotNull
    private Long contaCorrenteId;
    @NotNull
    private Long empresaId;
    @NotBlank
    private String matriculaFuncionario;
    @NotNull
    private LocalDate dataAdmissao;
    @NotNull
    @DecimalMin("0.01")
    private BigDecimal valorSalarioBruto;
    @NotNull
    @DecimalMin("0.01")
    private BigDecimal valorSalarioLiquido;
    @NotNull
    @Min(1)
    @Max(31)
    private Integer diaPagamento;

    public ContaSalarioRequest() {}
    // getters/setters
    public Long getContaCorrenteId() { return contaCorrenteId; }
    public void setContaCorrenteId(Long v) { this.contaCorrenteId = v; }
    public Long getEmpresaId() { return empresaId; }
    public void setEmpresaId(Long v) { this.empresaId = v; }
    public String getMatriculaFuncionario() { return matriculaFuncionario; }
    public void setMatriculaFuncionario(String v) { this.matriculaFuncionario = v; }
    public LocalDate getDataAdmissao() { return dataAdmissao; }
    public void setDataAdmissao(LocalDate v) { this.dataAdmissao = v; }
    public BigDecimal getValorSalarioBruto() { return valorSalarioBruto; }
    public void setValorSalarioBruto(BigDecimal v) { this.valorSalarioBruto = v; }
    public BigDecimal getValorSalarioLiquido() { return valorSalarioLiquido; }
    public void setValorSalarioLiquido(BigDecimal v) { this.valorSalarioLiquido = v; }
    public Integer getDiaPagamento() { return diaPagamento; }
    public void setDiaPagamento(Integer v) { this.diaPagamento = v; }
}
```

`ContaSalarioResponse.java`:
```java
package com.aurix.platform.salario.dto;

import com.aurix.platform.salario.entity.ContaSalario.StatusContaSalario;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

public class ContaSalarioResponse {
    private Long id;
    private Long contaCorrenteId;
    private Long empresaId;
    private String matriculaFuncionario;
    private LocalDate dataAdmissao;
    private LocalDate dataRescisao;
    private BigDecimal valorSalarioBruto;
    private BigDecimal valorSalarioLiquido;
    private Integer diaPagamento;
    private Boolean portabilidadeAtiva;
    private StatusContaSalario status;
    private LocalDateTime dataCriacao;
    private LocalDateTime dataAtualizacao;

    public ContaSalarioResponse() {}
    // getters/setters
    public Long getId() { return id; }
    public void setId(Long v) { this.id = v; }
    public Long getContaCorrenteId() { return contaCorrenteId; }
    public void setContaCorrenteId(Long v) { this.contaCorrenteId = v; }
    public Long getEmpresaId() { return empresaId; }
    public void setEmpresaId(Long v) { this.empresaId = v; }
    public String getMatriculaFuncionario() { return matriculaFuncionario; }
    public void setMatriculaFuncionario(String v) { this.matriculaFuncionario = v; }
    public LocalDate getDataAdmissao() { return dataAdmissao; }
    public void setDataAdmissao(LocalDate v) { this.dataAdmissao = v; }
    public LocalDate getDataRescisao() { return dataRescisao; }
    public void setDataRescisao(LocalDate v) { this.dataRescisao = v; }
    public BigDecimal getValorSalarioBruto() { return valorSalarioBruto; }
    public void setValorSalarioBruto(BigDecimal v) { this.valorSalarioBruto = v; }
    public BigDecimal getValorSalarioLiquido() { return valorSalarioLiquido; }
    public void setValorSalarioLiquido(BigDecimal v) { this.valorSalarioLiquido = v; }
    public Integer getDiaPagamento() { return diaPagamento; }
    public void setDiaPagamento(Integer v) { this.diaPagamento = v; }
    public Boolean getPortabilidadeAtiva() { return portabilidadeAtiva; }
    public void setPortabilidadeAtiva(Boolean v) { this.portabilidadeAtiva = v; }
    public StatusContaSalario getStatus() { return status; }
    public void setStatus(StatusContaSalario v) { this.status = v; }
    public LocalDateTime getDataCriacao() { return dataCriacao; }
    public void setDataCriacao(LocalDateTime v) { this.dataCriacao = v; }
    public LocalDateTime getDataAtualizacao() { return dataAtualizacao; }
    public void setDataAtualizacao(LocalDateTime v) { this.dataAtualizacao = v; }
}
```

`ConvenioRequest.java`:
```java
package com.aurix.platform.salario.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

public class ConvenioRequest {
    @NotBlank
    private String cnpj;
    @NotBlank
    private String razaoSocial;
    @NotNull
    private Long contaCorrenteId;

    public ConvenioRequest() {}
    // getters/setters
    public String getCnpj() { return cnpj; }
    public void setCnpj(String v) { this.cnpj = v; }
    public String getRazaoSocial() { return razaoSocial; }
    public void setRazaoSocial(String v) { this.razaoSocial = v; }
    public Long getContaCorrenteId() { return contaCorrenteId; }
    public void setContaCorrenteId(Long v) { this.contaCorrenteId = v; }
}
```

`ConvenioResponse.java`:
```java
package com.aurix.platform.salario.dto;

import java.time.LocalDateTime;

public class ConvenioResponse {
    private Long id;
    private String cnpj;
    private String razaoSocial;
    private Long contaCorrenteId;
    private Boolean ativo;
    private LocalDateTime dataCriacao;
    private LocalDateTime dataAtualizacao;

    public ConvenioResponse() {}
    // getters/setters
    public Long getId() { return id; }
    public void setId(Long v) { this.id = v; }
    public String getCnpj() { return cnpj; }
    public void setCnpj(String v) { this.cnpj = v; }
    public String getRazaoSocial() { return razaoSocial; }
    public void setRazaoSocial(String v) { this.razaoSocial = v; }
    public Long getContaCorrenteId() { return contaCorrenteId; }
    public void setContaCorrenteId(Long v) { this.contaCorrenteId = v; }
    public Boolean getAtivo() { return ativo; }
    public void setAtivo(Boolean v) { this.ativo = v; }
    public LocalDateTime getDataCriacao() { return dataCriacao; }
    public void setDataCriacao(LocalDateTime v) { this.dataCriacao = v; }
    public LocalDateTime getDataAtualizacao() { return dataAtualizacao; }
    public void setDataAtualizacao(LocalDateTime v) { this.dataAtualizacao = v; }
}
```

`PortabilidadeRequest.java`:
```java
package com.aurix.platform.salario.dto;

import jakarta.validation.constraints.DecimalMax;
import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;

public class PortabilidadeRequest {
    @NotNull
    private Long contaSalarioId;
    @NotBlank
    private String codigoBancoDestino;
    @NotBlank
    private String agenciaDestino;
    @NotBlank
    private String contaDestino;
    @DecimalMin("0.01")
    @DecimalMax("100.00")
    private BigDecimal valorPercentual = new BigDecimal("100.00");

    public PortabilidadeRequest() {}
    // getters/setters
    public Long getContaSalarioId() { return contaSalarioId; }
    public void setContaSalarioId(Long v) { this.contaSalarioId = v; }
    public String getCodigoBancoDestino() { return codigoBancoDestino; }
    public void setCodigoBancoDestino(String v) { this.codigoBancoDestino = v; }
    public String getAgenciaDestino() { return agenciaDestino; }
    public void setAgenciaDestino(String v) { this.agenciaDestino = v; }
    public String getContaDestino() { return contaDestino; }
    public void setContaDestino(String v) { this.contaDestino = v; }
    public BigDecimal getValorPercentual() { return valorPercentual; }
    public void setValorPercentual(BigDecimal v) { this.valorPercentual = v; }
}
```

`PortabilidadeResponse.java`:
```java
package com.aurix.platform.salario.dto;

import com.aurix.platform.salario.entity.SolicitacaoPortabilidade.StatusPortabilidade;
import java.math.BigDecimal;
import java.time.LocalDateTime;

public class PortabilidadeResponse {
    private Long id;
    private Long contaSalarioId;
    private String codigoBancoDestino;
    private String agenciaDestino;
    private String contaDestino;
    private BigDecimal valorPercentual;
    private StatusPortabilidade status;
    private LocalDateTime dataSolicitacao;
    private LocalDateTime dataCriacao;

    public PortabilidadeResponse() {}
    // getters/setters
    public Long getId() { return id; }
    public void setId(Long v) { this.id = v; }
    public Long getContaSalarioId() { return contaSalarioId; }
    public void setContaSalarioId(Long v) { this.contaSalarioId = v; }
    public String getCodigoBancoDestino() { return codigoBancoDestino; }
    public void setCodigoBancoDestino(String v) { this.codigoBancoDestino = v; }
    public String getAgenciaDestino() { return agenciaDestino; }
    public void setAgenciaDestino(String v) { this.agenciaDestino = v; }
    public String getContaDestino() { return contaDestino; }
    public void setContaDestino(String v) { this.contaDestino = v; }
    public BigDecimal getValorPercentual() { return valorPercentual; }
    public void setValorPercentual(BigDecimal v) { this.valorPercentual = v; }
    public StatusPortabilidade getStatus() { return status; }
    public void setStatus(StatusPortabilidade v) { this.status = v; }
    public LocalDateTime getDataSolicitacao() { return dataSolicitacao; }
    public void setDataSolicitacao(LocalDateTime v) { this.dataSolicitacao = v; }
    public LocalDateTime getDataCriacao() { return dataCriacao; }
    public void setDataCriacao(LocalDateTime v) { this.dataCriacao = v; }
}
```

`FolhaResponse.java`:
```java
package com.aurix.platform.salario.dto;

import com.aurix.platform.salario.entity.FolhaPagamento.StatusFolha;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

public class FolhaResponse {
    private Long id;
    private Long empresaId;
    private String arquivoNome;
    private Integer totalFuncionarios;
    private BigDecimal valorTotal;
    private LocalDate dataReferencia;
    private LocalDateTime dataProcessamento;
    private StatusFolha status;
    private LocalDateTime dataCriacao;

    public FolhaResponse() {}
    // getters/setters
    public Long getId() { return id; }
    public void setId(Long v) { this.id = v; }
    public Long getEmpresaId() { return empresaId; }
    public void setEmpresaId(Long v) { this.empresaId = v; }
    public String getArquivoNome() { return arquivoNome; }
    public void setArquivoNome(String v) { this.arquivoNome = v; }
    public Integer getTotalFuncionarios() { return totalFuncionarios; }
    public void setTotalFuncionarios(Integer v) { this.totalFuncionarios = v; }
    public BigDecimal getValorTotal() { return valorTotal; }
    public void setValorTotal(BigDecimal v) { this.valorTotal = v; }
    public LocalDate getDataReferencia() { return dataReferencia; }
    public void setDataReferencia(LocalDate v) { this.dataReferencia = v; }
    public LocalDateTime getDataProcessamento() { return dataProcessamento; }
    public void setDataProcessamento(LocalDateTime v) { this.dataProcessamento = v; }
    public StatusFolha getStatus() { return status; }
    public void setStatus(StatusFolha v) { this.status = v; }
    public LocalDateTime getDataCriacao() { return dataCriacao; }
    public void setDataCriacao(LocalDateTime v) { this.dataCriacao = v; }
}
```

`ItemFolhaResponse.java`:
```java
package com.aurix.platform.salario.dto;

import com.aurix.platform.salario.entity.ItemFolhaPagamento.StatusItem;
import java.math.BigDecimal;

public class ItemFolhaResponse {
    private Long id;
    private Long folhaId;
    private Long contaSalarioId;
    private String cpfFuncionario;
    private BigDecimal valorLiquido;
    private String descontos;
    private StatusItem status;

    public ItemFolhaResponse() {}
    // getters/setters
    public Long getId() { return id; }
    public void setId(Long v) { this.id = v; }
    public Long getFolhaId() { return folhaId; }
    public void setFolhaId(Long v) { this.folhaId = v; }
    public Long getContaSalarioId() { return contaSalarioId; }
    public void setContaSalarioId(Long v) { this.contaSalarioId = v; }
    public String getCpfFuncionario() { return cpfFuncionario; }
    public void setCpfFuncionario(String v) { this.cpfFuncionario = v; }
    public BigDecimal getValorLiquido() { return valorLiquido; }
    public void setValorLiquido(BigDecimal v) { this.valorLiquido = v; }
    public String getDescontos() { return descontos; }
    public void setDescontos(String v) { this.descontos = v; }
    public StatusItem getStatus() { return status; }
    public void setStatus(StatusItem v) { this.status = v; }
}
```

`CreditoDiretoRequest.java`:
```java
package com.aurix.platform.salario.dto;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;

public class CreditoDiretoRequest {
    @NotNull
    private Long empresaId;
    @NotBlank
    private String cpfFuncionario;
    @NotNull
    @DecimalMin("0.01")
    private BigDecimal valorLiquido;
    private String descontos;

    public CreditoDiretoRequest() {}
    // getters/setters
    public Long getEmpresaId() { return empresaId; }
    public void setEmpresaId(Long v) { this.empresaId = v; }
    public String getCpfFuncionario() { return cpfFuncionario; }
    public void setCpfFuncionario(String v) { this.cpfFuncionario = v; }
    public BigDecimal getValorLiquido() { return valorLiquido; }
    public void setValorLiquido(BigDecimal v) { this.valorLiquido = v; }
    public String getDescontos() { return descontos; }
    public void setDescontos(String v) { this.descontos = v; }
}
```

- [ ] **Step 8: Create 3 event records**

`ContaSalarioCriadaEvent.java`:
```java
package com.aurix.platform.salario.event;

import java.math.BigDecimal;
import java.time.LocalDate;

public record ContaSalarioCriadaEvent(
    Long contaSalarioId,
    Long clienteId,
    Long empresaId,
    LocalDate dataAdmissao,
    BigDecimal valorSalarioLiquido
) {}
```

`SalarioCreditadoEvent.java`:
```java
package com.aurix.platform.salario.event;

import java.math.BigDecimal;
import java.time.LocalDate;

public record SalarioCreditadoEvent(
    Long contaSalarioId,
    BigDecimal valor,
    String tipo,
    Long empresaId,
    LocalDate dataReferencia
) {}
```

`PortabilidadeSolicitadaEvent.java`:
```java
package com.aurix.platform.salario.event;

import java.math.BigDecimal;

public record PortabilidadeSolicitadaEvent(
    Long portabilidadeId,
    Long contaSalarioId,
    String codigoBancoDestino,
    BigDecimal valorPercentual
) {}
```

- [ ] **Step 9: Create `package-info.java` files**

For each package (entity, repository, dto, event, client, service, controller, config, job):
```java
@NullMarked
package com.aurix.platform.salario.entity;

import org.jspecify.annotations.NullMarked;
```

- [ ] **Step 10: Verify compilation**

Run: `mvn compile -pl aurix-salario -am -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 11: Commit**

```bash
git add backend/aurix-salario/src/main/java/com/aurix/platform/salario/
git commit -m "feat(salario): add domain layer - entities, repos, DTOs, events"
```

---

### Task 3: HTTP Client + Config

**Files:**
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/client/ContaCorrenteClient.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/client/CreditoRequest.java` (inner DTO)
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/config/SalarioHttpConfig.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/config/SalarioKafkaConfig.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/config/SalarioSecurityConfig.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/config/CnabConfig.java`

- [ ] **Step 1: Create `ContaCorrenteClient.java`**

```java
package com.aurix.platform.salario.client;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.service.annotation.HttpExchange;
import org.springframework.web.service.annotation.PostExchange;
import java.math.BigDecimal;

@HttpExchange("/api/core/contas")
public interface ContaCorrenteClient {

    @PostExchange("/{contaId}/creditar")
    void creditar(@PathVariable Long contaId, @RequestBody CreditoRequest request);

    @PostExchange("/{contaId}/debitar")
    void debitar(@PathVariable Long contaId, @RequestBody DebitoRequest request);

    record CreditoRequest(
        @NotNull @DecimalMin("0.01") BigDecimal valor,
        @NotBlank String descricao
    ) {}

    record DebitoRequest(
        @NotNull @DecimalMin("0.01") BigDecimal valor,
        @NotBlank String descricao
    ) {}
}
```

- [ ] **Step 2: Create `SalarioHttpConfig.java`**

```java
package com.aurix.platform.salario.config;

import com.aurix.platform.salario.client.ContaCorrenteClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.service.registry.HttpServiceProxyFactory;
import org.springframework.web.service.registry.ImportHttpServices;

@Configuration
@ImportHttpServices
public class SalarioHttpConfig {

    @Bean
    public ContaCorrenteClient contaCorrenteClient(HttpServiceProxyFactory factory) {
        return factory.createClient(ContaCorrenteClient.class);
    }
}
```

- [ ] **Step 3: Create `SalarioKafkaConfig.java`**

```java
package com.aurix.platform.salario.config;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class SalarioKafkaConfig {

    public static final String TOPICO_CONTA_CRIADA = "salario-conta-criada";
    public static final String TOPICO_CREDITO = "salario-creditado";
    public static final String TOPICO_PORTABILIDADE = "salario-portabilidade-solicitada";

    @Bean
    public NewTopic topicContaCriada() {
        return TopicBuilder.name(TOPICO_CONTA_CRIADA).partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic topicCredito() {
        return TopicBuilder.name(TOPICO_CREDITO).partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic topicPortabilidade() {
        return TopicBuilder.name(TOPICO_PORTABILIDADE).partitions(3).replicas(1).build();
    }
}
```

- [ ] **Step 4: Create `SalarioSecurityConfig.java`**

Same pattern as poupanca — stateless, actuator/swagger public, business endpoints authenticated.

```java
package com.aurix.platform.salario.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SalarioSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/**").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
        return http.build();
    }
}
```

- [ ] **Step 5: Create `CnabConfig.java`**

```java
package com.aurix.platform.salario.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConfigurationProperties(prefix = "aurix.salario.cnab")
public class CnabConfig {
    private String dirUpload = "./data/cnab";
    private String maxFileSize = "10MB";

    public String getDirUpload() { return dirUpload; }
    public void setDirUpload(String dirUpload) { this.dirUpload = dirUpload; }
    public String getMaxFileSize() { return maxFileSize; }
    public void setMaxFileSize(String maxFileSize) { this.maxFileSize = maxFileSize; }
}
```

- [ ] **Step 6: Verify compilation**

Run: `mvn compile -pl aurix-salario -am -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add backend/aurix-salario/src/main/java/com/aurix/platform/salario/client/ backend/aurix-salario/src/main/java/com/aurix/platform/salario/config/
git commit -m "feat(salario): add @HttpExchange client and config (Kafka, Security, CNAB)"
```

---

### Task 4: ContaSalario Service + Controller + Tests

**Files:**
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/service/ContaSalarioService.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/controller/ContaSalarioController.java`
- Create: `backend/aurix-salario/src/test/java/com/aurix/platform/salario/controller/ContaSalarioControllerTest.java`
- Create: `backend/aurix-salario/src/test/resources/application-test.yml`

**Interfaces:**
- Consumes: `ContaSalarioRepository`, `ConvenioEmpresaRepository`, `KafkaTemplate<String, String>`, `ObjectMapper`
- Produces: `ContaSalarioService` with methods `criarConta()`, `buscarPorId()`, `listarPorCliente()` (delegates to core for client lookup), `listarPorEmpresa()`, `bloquearConta()`, `rescindirConta()`

- [ ] **Step 1: Create `ContaSalarioService.java`**

```java
package com.aurix.platform.salario.service;

import com.aurix.platform.salario.config.SalarioKafkaConfig;
import com.aurix.platform.salario.dto.ContaSalarioRequest;
import com.aurix.platform.salario.dto.ContaSalarioResponse;
import com.aurix.platform.salario.entity.ContaSalario;
import com.aurix.platform.salario.entity.ContaSalario.StatusContaSalario;
import com.aurix.platform.salario.entity.ConvenioEmpresa;
import com.aurix.platform.salario.event.ContaSalarioCriadaEvent;
import com.aurix.platform.salario.repository.ContaSalarioRepository;
import com.aurix.platform.salario.repository.ConvenioEmpresaRepository;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDate;
import java.util.List;
import java.util.stream.Collectors;

@Service
@Transactional
public class ContaSalarioService {

    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(ContaSalarioService.class);

    private final ContaSalarioRepository contaSalarioRepository;
    private final ConvenioEmpresaRepository convenioEmpresaRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    public ContaSalarioService(ContaSalarioRepository contaSalarioRepository,
                               ConvenioEmpresaRepository convenioEmpresaRepository,
                               KafkaTemplate<String, String> kafkaTemplate,
                               ObjectMapper objectMapper) {
        this.contaSalarioRepository = contaSalarioRepository;
        this.convenioEmpresaRepository = convenioEmpresaRepository;
        this.kafkaTemplate = kafkaTemplate;
        this.objectMapper = objectMapper;
    }

    public ContaSalarioResponse criarConta(ContaSalarioRequest request) {
        log.info("Criando conta salario para matricula: {}", request.getMatriculaFuncionario());

        ConvenioEmpresa empresa = convenioEmpresaRepository.findByTenantIdAndId(
            com.aurix.platform.shared.tenant.TenantContext.getTenantId(),
            request.getEmpresaId()
        ).orElseThrow(() -> new IllegalArgumentException("Empresa conveniada nao encontrada: " + request.getEmpresaId()));

        ContaSalario conta = new ContaSalario(
            request.getContaCorrenteId(),
            request.getEmpresaId(),
            request.getMatriculaFuncionario(),
            request.getDataAdmissao(),
            request.getValorSalarioBruto(),
            request.getValorSalarioLiquido(),
            request.getDiaPagamento()
        );
        conta.setTenantId(com.aurix.platform.shared.tenant.TenantContext.getTenantId());

        ContaSalario salva = contaSalarioRepository.save(conta);

        publicarEventoContaCriada(salva);

        log.info("Conta salario criada: id={}, empresa={}", salva.getId(), empresa.getRazaoSocial());
        return converterParaResponse(salva);
    }

    @Transactional(readOnly = true)
    public ContaSalarioResponse buscarPorId(Long id) {
        ContaSalario conta = contaSalarioRepository.findByTenantIdAndId(
            com.aurix.platform.shared.tenant.TenantContext.getTenantId(), id
        ).orElseThrow(() -> new IllegalArgumentException("Conta salario nao encontrada: " + id));
        return converterParaResponse(conta);
    }

    @Transactional(readOnly = true)
    public List<ContaSalarioResponse> listarPorEmpresa(Long empresaId) {
        return contaSalarioRepository.findByTenantIdAndEmpresaId(
            com.aurix.platform.shared.tenant.TenantContext.getTenantId(), empresaId
        ).stream().map(this::converterParaResponse).collect(Collectors.toList());
    }

    public void bloquearConta(Long id) {
        ContaSalario conta = contaSalarioRepository.findByTenantIdAndId(
            com.aurix.platform.shared.tenant.TenantContext.getTenantId(), id
        ).orElseThrow(() -> new IllegalArgumentException("Conta salario nao encontrada: " + id));
        conta.setStatus(StatusContaSalario.BLOQUEADA);
        contaSalarioRepository.save(conta);
        log.info("Conta salario bloqueada: {}", id);
    }

    public void rescindirConta(Long id) {
        ContaSalario conta = contaSalarioRepository.findByTenantIdAndId(
            com.aurix.platform.shared.tenant.TenantContext.getTenantId(), id
        ).orElseThrow(() -> new IllegalArgumentException("Conta salario nao encontrada: " + id));
        conta.setStatus(StatusContaSalario.RESCINDIDA);
        conta.setDataRescisao(LocalDate.now());
        contaSalarioRepository.save(conta);
        log.info("Conta salario rescindida: {}", id);
    }

    private void publicarEventoContaCriada(ContaSalario conta) {
        try {
            String json = objectMapper.writeValueAsString(new ContaSalarioCriadaEvent(
                conta.getId(), null, conta.getEmpresaId(), conta.getDataAdmissao(),
                conta.getValorSalarioLiquido()));
            kafkaTemplate.send(SalarioKafkaConfig.TOPICO_CONTA_CRIADA, json);
        } catch (JsonProcessingException e) {
            log.warn("Falha ao serializar evento conta-criada: {}", e.getMessage());
        } catch (Exception e) {
            log.warn("Falha ao publicar evento conta-criada: {}", e.getMessage());
        }
    }

    private ContaSalarioResponse converterParaResponse(ContaSalario conta) {
        ContaSalarioResponse resp = new ContaSalarioResponse();
        resp.setId(conta.getId());
        resp.setContaCorrenteId(conta.getContaCorrenteId());
        resp.setEmpresaId(conta.getEmpresaId());
        resp.setMatriculaFuncionario(conta.getMatriculaFuncionario());
        resp.setDataAdmissao(conta.getDataAdmissao());
        resp.setDataRescisao(conta.getDataRescisao());
        resp.setValorSalarioBruto(conta.getValorSalarioBruto());
        resp.setValorSalarioLiquido(conta.getValorSalarioLiquido());
        resp.setDiaPagamento(conta.getDiaPagamento());
        resp.setPortabilidadeAtiva(conta.getPortabilidadeAtiva());
        resp.setStatus(conta.getStatus());
        resp.setDataCriacao(conta.getDataCriacao());
        resp.setDataAtualizacao(conta.getDataAtualizacao());
        return resp;
    }
}
```

- [ ] **Step 2: Create `ContaSalarioController.java`**

```java
package com.aurix.platform.salario.controller;

import com.aurix.platform.salario.dto.ContaSalarioRequest;
import com.aurix.platform.salario.dto.ContaSalarioResponse;
import com.aurix.platform.salario.service.ContaSalarioService;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/contas")
public class ContaSalarioController {

    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(ContaSalarioController.class);
    private final ContaSalarioService contaSalarioService;

    public ContaSalarioController(ContaSalarioService contaSalarioService) {
        this.contaSalarioService = contaSalarioService;
    }

    @PostMapping
    public ResponseEntity<ContaSalarioResponse> criarConta(@Valid @RequestBody ContaSalarioRequest request) {
        log.info("Criando conta salario para matricula: {}", request.getMatriculaFuncionario());
        ContaSalarioResponse response = contaSalarioService.criarConta(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @GetMapping("/{id}")
    public ResponseEntity<ContaSalarioResponse> buscarPorId(@PathVariable Long id) {
        return ResponseEntity.ok(contaSalarioService.buscarPorId(id));
    }

    @GetMapping("/empresa/{empresaId}")
    public ResponseEntity<List<ContaSalarioResponse>> listarPorEmpresa(@PathVariable Long empresaId) {
        return ResponseEntity.ok(contaSalarioService.listarPorEmpresa(empresaId));
    }

    @PatchMapping("/{id}/bloquear")
    public ResponseEntity<Void> bloquearConta(@PathVariable Long id) {
        contaSalarioService.bloquearConta(id);
        return ResponseEntity.noContent().build();
    }

    @PatchMapping("/{id}/rescindir")
    public ResponseEntity<Void> rescindirConta(@PathVariable Long id) {
        contaSalarioService.rescindirConta(id);
        return ResponseEntity.noContent().build();
    }
}
```

- [ ] **Step 3: Create `application-test.yml`**

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
    username: sa
    password:
    driver-class-name: org.h2.Driver
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
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080

aurix:
  salario:
    cnab:
      dir-upload: ./target/test-cnab

logging:
  level:
    com.aurix.platform: DEBUG
```

- [ ] **Step 4: Create `ContaSalarioControllerTest.java`**

```java
package com.aurix.platform.salario.controller;

import com.aurix.platform.salario.dto.ContaSalarioRequest;
import com.aurix.platform.salario.dto.ContaSalarioResponse;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.context.ImportTestcontainers;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.web.client.RestTemplate;
import java.math.BigDecimal;
import java.time.LocalDate;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@ImportTestcontainers
class ContaSalarioControllerTest {

    @Autowired
    private RestTemplate restTemplate;

    @Test
    void deveCriarContaSalarioComSucesso() {
        ContaSalarioRequest request = new ContaSalarioRequest();
        request.setContaCorrenteId(1L);
        request.setEmpresaId(1L);
        request.setMatriculaFuncionario("FUNC001");
        request.setDataAdmissao(LocalDate.of(2026, 1, 15));
        request.setValorSalarioBruto(new BigDecimal("5000.00"));
        request.setValorSalarioLiquido(new BigDecimal("4250.00"));
        request.setDiaPagamento(5);

        ResponseEntity<ContaSalarioResponse> response = restTemplate.postForEntity(
            "/contas", request, ContaSalarioResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getMatriculaFuncionario()).isEqualTo("FUNC001");
        assertThat(response.getBody().getStatus().name()).isEqualTo("ATIVA");
    }

    @Test
    void deveBuscarContaPorId() {
        ContaSalarioRequest request = new ContaSalarioRequest();
        request.setContaCorrenteId(2L);
        request.setEmpresaId(1L);
        request.setMatriculaFuncionario("FUNC002");
        request.setDataAdmissao(LocalDate.of(2026, 3, 1));
        request.setValorSalarioBruto(new BigDecimal("8000.00"));
        request.setValorSalarioLiquido(new BigDecimal("6800.00"));
        request.setDiaPagamento(30);

        ResponseEntity<ContaSalarioResponse> created = restTemplate.postForEntity(
            "/contas", request, ContaSalarioResponse.class);
        Long id = created.getBody().getId();

        ResponseEntity<ContaSalarioResponse> response = restTemplate.getForEntity(
            "/contas/" + id, ContaSalarioResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().getId()).isEqualTo(id);
        assertThat(response.getBody().getMatriculaFuncionario()).isEqualTo("FUNC002");
    }
}
```

- [ ] **Step 5: Run tests**

Run: `mvn test -pl aurix-salario -am -Dtest=ContaSalarioControllerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: BUILD SUCCESS, 2/2 tests pass

- [ ] **Step 6: Commit**

```bash
git add backend/aurix-salario/src/main/java/com/aurix/platform/salario/service/ContaSalarioService.java backend/aurix-salario/src/main/java/com/aurix/platform/salario/controller/ContaSalarioController.java backend/aurix-salario/src/test/
git commit -m "feat(salario): add ContaSalarioService, controller, and tests"
```

---

### Task 5: Convenio + Portabilidade Services + Controllers

**Files:**
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/service/ConvenioService.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/controller/ConvenioController.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/service/PortabilidadeService.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/controller/PortabilidadeController.java`

**Interfaces:**
- Consumes: `ConvenioEmpresaRepository`, `ContaSalarioRepository`, `SolicitacaoPortabilidadeRepository`, `KafkaTemplate`, `ObjectMapper`
- Produces: CRUD for convenios, portabilidade request/cancel

- [ ] **Step 1: Create `ConvenioService.java`**

```java
package com.aurix.platform.salario.service;

import com.aurix.platform.salario.dto.ConvenioRequest;
import com.aurix.platform.salario.dto.ConvenioResponse;
import com.aurix.platform.salario.entity.ConvenioEmpresa;
import com.aurix.platform.salario.repository.ConvenioEmpresaRepository;
import com.aurix.platform.shared.tenant.TenantContext;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;
import java.util.stream.Collectors;

@Service
@Transactional
public class ConvenioService {

    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(ConvenioService.class);
    private final ConvenioEmpresaRepository repository;

    public ConvenioService(ConvenioEmpresaRepository repository) {
        this.repository = repository;
    }

    public ConvenioResponse cadastrar(ConvenioRequest request) {
        log.info("Cadastrando empresa conveniada CNPJ: {}", request.getCnpj());
        ConvenioEmpresa empresa = new ConvenioEmpresa(request.getCnpj(), request.getRazaoSocial(), request.getContaCorrenteId());
        empresa.setTenantId(TenantContext.getTenantId());
        ConvenioEmpresa salva = repository.save(empresa);
        return converterParaResponse(salva);
    }

    @Transactional(readOnly = true)
    public ConvenioResponse buscarPorId(Long id) {
        ConvenioEmpresa empresa = repository.findByTenantIdAndId(TenantContext.getTenantId(), id)
            .orElseThrow(() -> new IllegalArgumentException("Convenio nao encontrado: " + id));
        return converterParaResponse(empresa);
    }

    @Transactional(readOnly = true)
    public List<ConvenioResponse> listarAtivos() {
        return repository.findByTenantId(TenantContext.getTenantId()).stream()
            .filter(ConvenioEmpresa::getAtivo)
            .map(this::converterParaResponse)
            .collect(Collectors.toList());
    }

    public ConvenioResponse atualizar(Long id, ConvenioRequest request) {
        ConvenioEmpresa empresa = repository.findByTenantIdAndId(TenantContext.getTenantId(), id)
            .orElseThrow(() -> new IllegalArgumentException("Convenio nao encontrado: " + id));
        empresa.setCnpj(request.getCnpj());
        empresa.setRazaoSocial(request.getRazaoSocial());
        empresa.setContaCorrenteId(request.getContaCorrenteId());
        return converterParaResponse(repository.save(empresa));
    }

    private ConvenioResponse converterParaResponse(ConvenioEmpresa empresa) {
        ConvenioResponse resp = new ConvenioResponse();
        resp.setId(empresa.getId());
        resp.setCnpj(empresa.getCnpj());
        resp.setRazaoSocial(empresa.getRazaoSocial());
        resp.setContaCorrenteId(empresa.getContaCorrenteId());
        resp.setAtivo(empresa.getAtivo());
        resp.setDataCriacao(empresa.getDataCriacao());
        resp.setDataAtualizacao(empresa.getDataAtualizacao());
        return resp;
    }
}
```

- [ ] **Step 2: Create `ConvenioController.java`**

```java
package com.aurix.platform.salario.controller;

import com.aurix.platform.salario.dto.ConvenioRequest;
import com.aurix.platform.salario.dto.ConvenioResponse;
import com.aurix.platform.salario.service.ConvenioService;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/convenios")
public class ConvenioController {

    private final ConvenioService convenioService;

    public ConvenioController(ConvenioService convenioService) {
        this.convenioService = convenioService;
    }

    @PostMapping
    public ResponseEntity<ConvenioResponse> cadastrar(@Valid @RequestBody ConvenioRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(convenioService.cadastrar(request));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ConvenioResponse> buscarPorId(@PathVariable Long id) {
        return ResponseEntity.ok(convenioService.buscarPorId(id));
    }

    @PutMapping("/{id}")
    public ResponseEntity<ConvenioResponse> atualizar(@PathVariable Long id, @Valid @RequestBody ConvenioRequest request) {
        return ResponseEntity.ok(convenioService.atualizar(id, request));
    }

    @GetMapping
    public ResponseEntity<List<ConvenioResponse>> listarAtivos() {
        return ResponseEntity.ok(convenioService.listarAtivos());
    }
}
```

- [ ] **Step 3: Create `PortabilidadeService.java`**

```java
package com.aurix.platform.salario.service;

import com.aurix.platform.salario.config.SalarioKafkaConfig;
import com.aurix.platform.salario.dto.PortabilidadeRequest;
import com.aurix.platform.salario.dto.PortabilidadeResponse;
import com.aurix.platform.salario.entity.ContaSalario;
import com.aurix.platform.salario.entity.SolicitacaoPortabilidade;
import com.aurix.platform.salario.event.PortabilidadeSolicitadaEvent;
import com.aurix.platform.salario.repository.ContaSalarioRepository;
import com.aurix.platform.salario.repository.SolicitacaoPortabilidadeRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;
import java.util.stream.Collectors;

@Service
@Transactional
public class PortabilidadeService {

    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(PortabilidadeService.class);

    private final SolicitacaoPortabilidadeRepository solicitacaoRepository;
    private final ContaSalarioRepository contaSalarioRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    public PortabilidadeService(SolicitacaoPortabilidadeRepository solicitacaoRepository,
                                ContaSalarioRepository contaSalarioRepository,
                                KafkaTemplate<String, String> kafkaTemplate,
                                ObjectMapper objectMapper) {
        this.solicitacaoRepository = solicitacaoRepository;
        this.contaSalarioRepository = contaSalarioRepository;
        this.kafkaTemplate = kafkaTemplate;
        this.objectMapper = objectMapper;
    }

    public PortabilidadeResponse solicitar(PortabilidadeRequest request) {
        log.info("Solicitando portabilidade para conta salario: {}", request.getContaSalarioId());

        ContaSalario conta = contaSalarioRepository.findById(request.getContaSalarioId())
            .orElseThrow(() -> new IllegalArgumentException("Conta salario nao encontrada: " + request.getContaSalarioId()));

        SolicitacaoPortabilidade solicitacao = new SolicitacaoPortabilidade(
            request.getContaSalarioId(), request.getCodigoBancoDestino(),
            request.getAgenciaDestino(), request.getContaDestino());
        solicitacao.setValorPercentual(request.getValorPercentual());

        SolicitacaoPortabilidade salva = solicitacaoRepository.save(solicitacao);

        // Ativar portabilidade na conta
        conta.setPortabilidadeAtiva(true);
        contaSalarioRepository.save(conta);

        publicarEventoPortabilidade(salva);

        log.info("Portabilidade solicitada: id={}, banco={}", salva.getId(), request.getCodigoBancoDestino());
        return converterParaResponse(salva);
    }

    @Transactional(readOnly = true)
    public List<PortabilidadeResponse> listarPorConta(Long contaSalarioId) {
        return solicitacaoRepository.findByContaSalarioId(contaSalarioId).stream()
            .map(this::converterParaResponse).collect(Collectors.toList());
    }

    public void cancelar(Long id) {
        SolicitacaoPortabilidade solicitacao = solicitacaoRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Portabilidade nao encontrada: " + id));
        solicitacao.setStatus(SolicitacaoPortabilidade.StatusPortabilidade.CANCELADA);
        solicitacaoRepository.save(solicitacao);

        // Se for a única ativa, desativar portabilidade na conta
        List<SolicitacaoPortabilidade> ativas = solicitacaoRepository
            .findByContaSalarioIdAndStatus(solicitacao.getContaSalarioId(),
                SolicitacaoPortabilidade.StatusPortabilidade.ATIVA);
        if (ativas.isEmpty()) {
            ContaSalario conta = contaSalarioRepository.findById(solicitacao.getContaSalarioId())
                .orElseThrow();
            conta.setPortabilidadeAtiva(false);
            contaSalarioRepository.save(conta);
        }

        log.info("Portabilidade cancelada: {}", id);
    }

    private void publicarEventoPortabilidade(SolicitacaoPortabilidade solicitacao) {
        try {
            String json = objectMapper.writeValueAsString(new PortabilidadeSolicitadaEvent(
                solicitacao.getId(), solicitacao.getContaSalarioId(),
                solicitacao.getCodigoBancoDestino(), solicitacao.getValorPercentual()));
            kafkaTemplate.send(SalarioKafkaConfig.TOPICO_PORTABILIDADE, json);
        } catch (Exception e) {
            log.warn("Falha ao publicar evento portabilidade: {}", e.getMessage());
        }
    }

    private PortabilidadeResponse converterParaResponse(SolicitacaoPortabilidade s) {
        PortabilidadeResponse resp = new PortabilidadeResponse();
        resp.setId(s.getId());
        resp.setContaSalarioId(s.getContaSalarioId());
        resp.setCodigoBancoDestino(s.getCodigoBancoDestino());
        resp.setAgenciaDestino(s.getAgenciaDestino());
        resp.setContaDestino(s.getContaDestino());
        resp.setValorPercentual(s.getValorPercentual());
        resp.setStatus(s.getStatus());
        resp.setDataSolicitacao(s.getDataSolicitacao());
        resp.setDataCriacao(s.getDataCriacao());
        return resp;
    }
}
```

- [ ] **Step 4: Create `PortabilidadeController.java`**

```java
package com.aurix.platform.salario.controller;

import com.aurix.platform.salario.dto.PortabilidadeRequest;
import com.aurix.platform.salario.dto.PortabilidadeResponse;
import com.aurix.platform.salario.service.PortabilidadeService;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/portabilidade")
public class PortabilidadeController {

    private final PortabilidadeService portabilidadeService;

    public PortabilidadeController(PortabilidadeService portabilidadeService) {
        this.portabilidadeService = portabilidadeService;
    }

    @PostMapping
    public ResponseEntity<PortabilidadeResponse> solicitar(@Valid @RequestBody PortabilidadeRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(portabilidadeService.solicitar(request));
    }

    @GetMapping("/conta/{contaSalarioId}")
    public ResponseEntity<List<PortabilidadeResponse>> listarPorConta(@PathVariable Long contaSalarioId) {
        return ResponseEntity.ok(portabilidadeService.listarPorConta(contaSalarioId));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> cancelar(@PathVariable Long id) {
        portabilidadeService.cancelar(id);
        return ResponseEntity.noContent().build();
    }
}
```

- [ ] **Step 5: Verify compilation**

Run: `mvn compile -pl aurix-salario -am -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add backend/aurix-salario/src/main/java/com/aurix/platform/salario/service/ConvenioService.java backend/aurix-salario/src/main/java/com/aurix/platform/salario/controller/ConvenioController.java backend/aurix-salario/src/main/java/com/aurix/platform/salario/service/PortabilidadeService.java backend/aurix-salario/src/main/java/com/aurix/platform/salario/controller/PortabilidadeController.java
git commit -m "feat(salario): add Convenio and Portabilidade services and controllers"
```

---

### Task 6: CnabParser + CnabService

**Files:**
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/client/CnabParser.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/service/CnabService.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/controller/FolhaController.java` (upload endpoint)
- Create: `backend/aurix-salario/src/test/java/com/aurix/platform/salario/service/CnabParserTest.java`
- Create: `backend/aurix-salario/src/test/resources/cnab/folha-valida.txt`

**Interfaces:**
- Consumes: `CnabConfig`, `FolhaPagamentoRepository`, `ItemFolhaPagamentoRepository`, `ConvenioEmpresaRepository`, `ContaSalarioRepository`
- Produces: Parsed CNAB 240 file → FolhaPagamento + ItemFolhaPagamento entities

- [ ] **Step 1: Create `CnabParser.java`**

Simple parser for CNAB 240 layout (Febraban standard — lines of 240 chars).

```java
package com.aurix.platform.salario.client;

import org.springframework.stereotype.Component;
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.math.BigDecimal;
import java.nio.charset.StandardCharsets;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.List;

@Component
public class CnabParser {

    private static final int LINE_LENGTH = 240;

    public Resultado parse(String arquivoNome, InputStream inputStream) throws IOException {
        List<String> linhas = new ArrayList<>();
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(inputStream, StandardCharsets.UTF_8))) {
            String linha;
            while ((linha = reader.readLine()) != null) {
                if (linha.length() != LINE_LENGTH) {
                    throw new IllegalArgumentException(
                        "Linha CNAB invalida: esperado 240 caracteres, obtido " + linha.length());
                }
                linhas.add(linha);
            }
        }

        if (linhas.size() < 3) {
            throw new IllegalArgumentException("Arquivo CNAB muito curto: " + linhas.size() + " linhas");
        }

        String header = linhas.get(0);
        String trailer = linhas.get(linhas.size() - 1);

        String codigoBanco = header.substring(0, 3);
        String nomeEmpresa = header.substring(72, 102).trim();
        LocalDate dataGeracao = LocalDate.parse(header.substring(144, 152),
            DateTimeFormatter.ofPattern("ddMMyyyy"));
        int totalFuncionarios = Integer.parseInt(trailer.substring(17, 23).trim());
        BigDecimal valorTotal = new BigDecimal(trailer.substring(23, 39).trim())
            .divide(new BigDecimal("100"), 2, java.math.RoundingMode.HALF_EVEN);

        List<Detalhe> detalhes = new ArrayList<>();
        for (int i = 1; i < linhas.size() - 1; i++) {
            String det = linhas.get(i);
            String segmento = det.substring(13, 14);
            if ("A".equals(segmento)) {
                String cpf = det.substring(30, 41).trim();
                String matricula = det.substring(66, 86).trim();
                BigDecimal valor = new BigDecimal(det.substring(120, 135).trim())
                    .divide(new BigDecimal("100"), 2, java.math.RoundingMode.HALF_EVEN);
                detalhes.add(new Detalhe(cpf, matricula, valor));
            }
        }

        return new Resultado(codigoBanco, nomeEmpresa, dataGeracao, totalFuncionarios,
            valorTotal, detalhes, arquivoNome);
    }

    public record Resultado(
        String codigoBanco,
        String nomeEmpresa,
        LocalDate dataGeracao,
        int totalFuncionarios,
        BigDecimal valorTotal,
        List<Detalhe> detalhes,
        String arquivoNome
    ) {}

    public record Detalhe(
        String cpf,
        String matricula,
        BigDecimal valor
    ) {}
}
```

- [ ] **Step 2: Create test fixture `cnab/folha-valida.txt`**

A minimal valid CNAB 240 file with 1 header, 1 detail (segment A), and 1 trailer. Each line is exactly 240 characters. For testing, we generate a simplified fixture. Lines 0=header, 1=segment A detail, 2=trailer. The content uses spaces for unused fields.

```
3410000000000000EMPRESA TESTE LTDA          00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
3410001300000A0000000000000000112345678901   MATRICULA001         00000000000000000000150000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
34100019999999990000000000015000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

Note: The fixture above is illustrative. Each line must be exactly 240 characters. When generating the test fixture file, ensure correct padding with spaces.

- [ ] **Step 3: Create `CnabParserTest.java`**

```java
package com.aurix.platform.salario.service;

import com.aurix.platform.salario.client.CnabParser;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.core.io.ClassPathResource;
import org.springframework.test.context.ActiveProfiles;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@ActiveProfiles("test")
class CnabParserTest {

    @Autowired
    private CnabParser cnabParser;

    @Test
    void deveParsearArquivoCNABValido() throws Exception {
        var resource = new ClassPathResource("cnab/folha-valida.txt");
        var resultado = cnabParser.parse("folha-valida.txt", resource.getInputStream());

        assertThat(resultado).isNotNull();
        assertThat(resultado.arquivoNome()).isEqualTo("folha-valida.txt");
        assertThat(resultado.totalFuncionarios()).isPositive();
        assertThat(resultado.valorTotal()).isPositive();
        assertThat(resultado.detalhes()).isNotEmpty();
        assertThat(resultado.detalhes().get(0).cpf()).isNotBlank();
    }
}
```

- [ ] **Step 4: Create `CnabService.java`**

```java
package com.aurix.platform.salario.service;

import com.aurix.platform.salario.client.CnabParser;
import com.aurix.platform.salario.config.CnabConfig;
import com.aurix.platform.salario.entity.FolhaPagamento;
import com.aurix.platform.salario.entity.ItemFolhaPagamento;
import com.aurix.platform.salario.repository.ContaSalarioRepository;
import com.aurix.platform.salario.repository.ConvenioEmpresaRepository;
import com.aurix.platform.salario.repository.FolhaPagamentoRepository;
import com.aurix.platform.salario.repository.ItemFolhaPagamentoRepository;
import com.aurix.platform.shared.tenant.TenantContext;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.io.InputStream;
import java.time.LocalDate;

@Service
@Transactional
public class CnabService {

    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(CnabService.class);

    private final CnabParser cnabParser;
    private final CnabConfig cnabConfig;
    private final FolhaPagamentoRepository folhaRepository;
    private final ItemFolhaPagamentoRepository itemRepository;
    private final ConvenioEmpresaRepository convenioRepository;
    private final ContaSalarioRepository contaSalarioRepository;

    public CnabService(CnabParser cnabParser, CnabConfig cnabConfig,
                       FolhaPagamentoRepository folhaRepository,
                       ItemFolhaPagamentoRepository itemRepository,
                       ConvenioEmpresaRepository convenioRepository,
                       ContaSalarioRepository contaSalarioRepository) {
        this.cnabParser = cnabParser;
        this.cnabConfig = cnabConfig;
        this.folhaRepository = folhaRepository;
        this.itemRepository = itemRepository;
        this.convenioRepository = convenioRepository;
        this.contaSalarioRepository = contaSalarioRepository;
    }

    public FolhaPagamento processarUpload(String arquivoNome, InputStream inputStream) {
        log.info("Processando upload CNAB: {}", arquivoNome);

        try {
            CnabParser.Resultado resultado = cnabParser.parse(arquivoNome, inputStream);

            FolhaPagamento folha = new FolhaPagamento(
                null,
                arquivoNome,
                resultado.totalFuncionarios(),
                resultado.valorTotal(),
                resultado.dataGeracao()
            );
            folha.setTenantId(TenantContext.getTenantId());
            folha.setStatus(FolhaPagamento.StatusFolha.VALIDADO);

            FolhaPagamento salva = folhaRepository.save(folha);

            for (CnabParser.Detalhe detalhe : resultado.detalhes()) {
                ItemFolhaPagamento item = new ItemFolhaPagamento(
                    salva.getId(), null, detalhe.cpf(), detalhe.valor()
                );
                item.setTenantId(TenantContext.getTenantId());
                itemRepository.save(item);
            }

            log.info("CNAB processado: {} funcionarios, total R$ {}", resultado.totalFuncionarios(), resultado.valorTotal());
            return salva;

        } catch (Exception e) {
            log.error("Erro ao processar CNAB: {}", e.getMessage());

            FolhaPagamento folha = new FolhaPagamento(
                null, arquivoNome, 0, java.math.BigDecimal.ZERO, LocalDate.now()
            );
            folha.setTenantId(TenantContext.getTenantId());
            folha.setStatus(FolhaPagamento.StatusFolha.ERRO_ESTRUTURA);
            return folhaRepository.save(folha);
        }
    }
}
```

Note: The `empresaId` in `FolhaPagamento` is null in the initial implementation above since the CNAB parser doesn't extract it from the header. Update the `CnabService` to look up the empresa by CPF/CNPJ or add empresaId to `CnabParser.Resultado` extracted from the header (positions 72-102 are the company name). The empresa lookup should be done by name in the service layer.

Revised approach for `CnabService.processarUpload`: extract the empresa name from parsed result, find matching `ConvenioEmpresa` by `razaoSocial`, and set `empresaId` on the `FolhaPagamento`.

- [ ] **Step 5: Add upload endpoint to `FolhaController.java`**

First pass — just the upload endpoint (more endpoints added in Task 7):

```java
package com.aurix.platform.salario.controller;

import com.aurix.platform.salario.dto.FolhaResponse;
import com.aurix.platform.salario.entity.FolhaPagamento;
import com.aurix.platform.salario.service.CnabService;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;
import java.io.IOException;

@RestController
@RequestMapping("/folhas")
public class FolhaController {

    private final CnabService cnabService;

    public FolhaController(CnabService cnabService) {
        this.cnabService = cnabService;
    }

    @PostMapping("/upload")
    public ResponseEntity<FolhaResponse> uploadCnab(@RequestParam("arquivo") MultipartFile arquivo) throws IOException {
        FolhaPagamento folha = cnabService.processarUpload(
            arquivo.getOriginalFilename(), arquivo.getInputStream());

        FolhaResponse response = new FolhaResponse();
        response.setId(folha.getId());
        response.setArquivoNome(folha.getArquivoNome());
        response.setTotalFuncionarios(folha.getTotalFuncionarios());
        response.setValorTotal(folha.getValorTotal());
        response.setDataReferencia(folha.getDataReferencia());
        response.setDataProcessamento(folha.getDataProcessamento());
        response.setStatus(folha.getStatus());
        response.setDataCriacao(folha.getDataCriacao());

        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}
```

- [ ] **Step 6: Verify compilation**

Run: `mvn compile -pl aurix-salario -am -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 7: Run parser test**

Run: `mvn test -pl aurix-salario -am -Dtest=CnabParserTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: BUILD SUCCESS, test passes

- [ ] **Step 8: Commit**

```bash
git add backend/aurix-salario/src/main/java/com/aurix/platform/salario/client/CnabParser.java backend/aurix-salario/src/main/java/com/aurix/platform/salario/service/CnabService.java backend/aurix-salario/src/main/java/com/aurix/platform/salario/controller/FolhaController.java backend/aurix-salario/src/test/
git commit -m "feat(salario): add CNAB parser, upload service, and CNAB test"
```

---

### Task 7: FolhaService + ProcessamentoFolhaJob

**Files:**
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/service/FolhaService.java`
- Create: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/job/ProcessamentoFolhaJob.java`
- Modify: `backend/aurix-salario/src/main/java/com/aurix/platform/salario/controller/FolhaController.java` (add list, status, itens, credito-direto endpoints)

**Interfaces:**
- Consumes: `FolhaPagamentoRepository`, `ItemFolhaPagamentoRepository`, `ContaSalarioRepository`, `ContaCorrenteClient`, `KafkaTemplate`, `ObjectMapper`
- `FolhaService.creditarDireto()`: find conta by CPF+empresa, credit via ContaCorrenteClient if portabilidade or via local save, emit Kafka event
- `ProcessamentoFolhaJob`: @Scheduled every hour, process VALIDATED folhas

- [ ] **Step 1: Create `FolhaService.java`**

```java
package com.aurix.platform.salario.service;

import com.aurix.platform.salario.client.ContaCorrenteClient;
import com.aurix.platform.salario.config.SalarioKafkaConfig;
import com.aurix.platform.salario.dto.CreditoDiretoRequest;
import com.aurix.platform.salario.entity.ContaSalario;
import com.aurix.platform.salario.entity.FolhaPagamento;
import com.aurix.platform.salario.entity.ItemFolhaPagamento;
import com.aurix.platform.salario.event.SalarioCreditadoEvent;
import com.aurix.platform.salario.repository.ContaSalarioRepository;
import com.aurix.platform.salario.repository.FolhaPagamentoRepository;
import com.aurix.platform.salario.repository.ItemFolhaPagamentoRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.List;

@Service
@Transactional
public class FolhaService {

    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(FolhaService.class);

    private final FolhaPagamentoRepository folhaRepository;
    private final ItemFolhaPagamentoRepository itemRepository;
    private final ContaSalarioRepository contaSalarioRepository;
    private final ContaCorrenteClient contaCorrenteClient;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    public FolhaService(FolhaPagamentoRepository folhaRepository,
                        ItemFolhaPagamentoRepository itemRepository,
                        ContaSalarioRepository contaSalarioRepository,
                        ContaCorrenteClient contaCorrenteClient,
                        KafkaTemplate<String, String> kafkaTemplate,
                        ObjectMapper objectMapper) {
        this.folhaRepository = folhaRepository;
        this.itemRepository = itemRepository;
        this.contaSalarioRepository = contaSalarioRepository;
        this.contaCorrenteClient = contaCorrenteClient;
        this.kafkaTemplate = kafkaTemplate;
        this.objectMapper = objectMapper;
    }

    public void creditarDireto(CreditoDiretoRequest request) {
        log.info("Credito direto para CPF: {}", request.getCpfFuncionario());

        ContaSalario conta = contaSalarioRepository.findByTenantIdAndEmpresaIdAndMatriculaFuncionario(
            com.aurix.platform.shared.tenant.TenantContext.getTenantId(),
            request.getEmpresaId(), request.getCpfFuncionario()
        ).orElseThrow(() -> new IllegalArgumentException(
            "Conta salario nao encontrada para CPF " + request.getCpfFuncionario()));

        if (conta.getPortabilidadeAtiva()) {
            contaCorrenteClient.creditar(conta.getContaCorrenteId(),
                new ContaCorrenteClient.CreditoRequest(
                    request.getValorLiquido(), "Salario - credito direto"));
        }

        publicarEventoCredito(conta, request.getValorLiquido(), "DIRETO", request.getEmpresaId());

        log.info("Credito direto realizado: conta={}, valor={}", conta.getId(), request.getValorLiquido());
    }

    public void creditarItem(ItemFolhaPagamento item, ContaSalario conta, Long empresaId) {
        try {
            if (conta.getPortabilidadeAtiva()) {
                contaCorrenteClient.creditar(conta.getContaCorrenteId(),
                    new ContaCorrenteClient.CreditoRequest(
                        item.getValorLiquido(), "Salario - CNAB"));
                item.setStatus(ItemFolhaPagamento.StatusItem.PORTADO);
            } else {
                item.setStatus(ItemFolhaPagamento.StatusItem.CREDITADO);
            }
            itemRepository.save(item);

            publicarEventoCredito(conta, item.getValorLiquido(), "CNAB", empresaId);

        } catch (Exception e) {
            log.error("Erro ao creditar item folha {}: {}", item.getId(), e.getMessage());
            item.setStatus(ItemFolhaPagamento.StatusItem.ERRO);
            itemRepository.save(item);
        }
    }

    public void processarFolha(FolhaPagamento folha) {
        List<ItemFolhaPagamento> itens = itemRepository.findByFolhaId(folha.getId());

        for (ItemFolhaPagamento item : itens) {
            contaSalarioRepository.findByTenantIdAndEmpresaIdAndMatriculaFuncionario(
                folha.getTenantId(), folha.getEmpresaId(), item.getCpfFuncionario()
            ).ifPresentOrElse(
                conta -> creditarItem(item, conta, folha.getEmpresaId()),
                () -> {
                    log.warn("Conta salario nao encontrada para CPF {} na folha {}", item.getCpfFuncionario(), folha.getId());
                    item.setStatus(ItemFolhaPagamento.StatusItem.ERRO);
                    itemRepository.save(item);
                }
            );
        }

        folha.setStatus(FolhaPagamento.StatusFolha.PROCESSADO);
        folhaRepository.save(folha);
        log.info("Folha processada: {} itens, {} funcionarios", folha.getId(), itens.size());
    }

    @Transactional(readOnly = true)
    public List<FolhaPagamento> listarFolhasPendentes() {
        return folhaRepository.findByTenantIdAndStatus(
            com.aurix.platform.shared.tenant.TenantContext.getTenantId(),
            FolhaPagamento.StatusFolha.VALIDADO);
    }

    private void publicarEventoCredito(ContaSalario conta, BigDecimal valor, String tipo, Long empresaId) {
        try {
            String json = objectMapper.writeValueAsString(new SalarioCreditadoEvent(
                conta.getId(), valor, tipo, empresaId, LocalDate.now()));
            kafkaTemplate.send(SalarioKafkaConfig.TOPICO_CREDITO, json);
        } catch (Exception e) {
            log.warn("Falha ao publicar evento credito: {}", e.getMessage());
        }
    }
}
```

- [ ] **Step 2: Create `ProcessamentoFolhaJob.java`**

```java
package com.aurix.platform.salario.job;

import com.aurix.platform.salario.entity.FolhaPagamento;
import com.aurix.platform.salario.service.FolhaService;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import java.util.List;

@Component
public class ProcessamentoFolhaJob {

    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(ProcessamentoFolhaJob.class);

    private final FolhaService folhaService;

    public ProcessamentoFolhaJob(FolhaService folhaService) {
        this.folhaService = folhaService;
    }

    @Scheduled(cron = "0 3 * * * *")
    public void executarProcessamento() {
        log.info("Iniciando processamento de folhas pendentes...");

        List<FolhaPagamento> pendentes = folhaService.listarFolhasPendentes();

        if (pendentes.isEmpty()) {
            log.info("Nenhuma folha pendente para processar");
            return;
        }

        for (FolhaPagamento folha : pendentes) {
            try {
                folhaService.processarFolha(folha);
            } catch (Exception e) {
                log.error("Erro ao processar folha {}: {}", folha.getId(), e.getMessage());
            }
        }

        log.info("Processamento concluido: {} folhas processadas", pendentes.size());
    }
}
```

- [ ] **Step 3: Update `FolhaController.java` with remaining endpoints**

Add methods to the existing FolhaController:

```java
@GetMapping("/{id}")
public ResponseEntity<FolhaResponse> buscarPorId(@PathVariable Long id) {
    FolhaPagamento folha = folhaRepository.findById(id)
        .orElseThrow(() -> new IllegalArgumentException("Folha nao encontrada: " + id));
    // converter para response
    return ResponseEntity.ok(converterParaResponse(folha));
}

@GetMapping
public ResponseEntity<List<FolhaResponse>> listarFolhas() {
    List<FolhaPagamento> folhas = folhaRepository.findByTenantIdAndEmpresaId(
        com.aurix.platform.shared.tenant.TenantContext.getTenantId(), null);
    return ResponseEntity.ok(folhas.stream().map(this::converterParaResponse).toList());
}

@GetMapping("/{id}/itens")
public ResponseEntity<List<ItemFolhaResponse>> listarItens(@PathVariable Long id) {
    // ...
}

@PostMapping("/credito-direto")
public ResponseEntity<Void> creditoDireto(@Valid @RequestBody CreditoDiretoRequest request) {
    folhaService.creditarDireto(request);
    return ResponseEntity.ok().build();
}
```

Full controller content will be generated by the implementer.

- [ ] **Step 4: Create `FolhaServiceTest.java`**

```java
package com.aurix.platform.salario.service;

import com.aurix.platform.salario.dto.CreditoDiretoRequest;
import com.aurix.platform.salario.entity.FolhaPagamento;
import com.aurix.platform.salario.entity.ItemFolhaPagamento;
import com.aurix.platform.salario.repository.FolhaPagamentoRepository;
import com.aurix.platform.salario.repository.ItemFolhaPagamentoRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import java.math.BigDecimal;
import java.time.LocalDate;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@ActiveProfiles("test")
class FolhaServiceTest {

    @Autowired
    private FolhaPagamentoRepository folhaRepository;

    @Autowired
    private ItemFolhaPagamentoRepository itemRepository;

    @Test
    void deveListarFolhasPendentes() {
        FolhaPagamento folha = new FolhaPagamento(null, "teste.txt", 1,
            new BigDecimal("1000.00"), LocalDate.now());
        folha.setTenantId("default");
        folhaRepository.save(folha);

        var pendentes = folhaRepository.findByTenantIdAndStatus("default",
            FolhaPagamento.StatusFolha.RECEBIDO);

        assertThat(pendentes).isNotEmpty();
    }

    @Test
    void deveSalvarItensDaFolha() {
        FolhaPagamento folha = new FolhaPagamento(null, "folha.txt", 2,
            new BigDecimal("5000.00"), LocalDate.now());
        folha.setTenantId("default");
        FolhaPagamento salva = folhaRepository.save(folha);

        ItemFolhaPagamento item = new ItemFolhaPagamento(
            salva.getId(), 1L, "12345678901", new BigDecimal("2500.00"));
        item.setTenantId("default");
        ItemFolhaPagamento salvo = itemRepository.save(item);

        assertThat(salvo.getId()).isNotNull();
        assertThat(salvo.getStatus()).isEqualTo(ItemFolhaPagamento.StatusItem.PENDENTE);
    }
}
```

- [ ] **Step 5: Verify compilation and tests**

Run: `mvn compile -pl aurix-salario -am -DskipTests -q`
Expected: BUILD SUCCESS

Run: `mvn test -pl aurix-salario -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add backend/aurix-salario/src/main/java/com/aurix/platform/salario/service/FolhaService.java backend/aurix-salario/src/main/java/com/aurix/platform/salario/job/ backend/aurix-salario/src/main/java/com/aurix/platform/salario/controller/FolhaController.java backend/aurix-salario/src/test/
git commit -m "feat(salario): add FolhaService, ProcessamentoFolhaJob, and FolhaController"
```

---

### Task 8: Full Build Verification

- [ ] **Step 1: Compile full project**

Run: `mvn compile -DskipTests -q`
Expected: BUILD SUCCESS (all 30+ modules including aurix-salario)

- [ ] **Step 2: Run all salario tests**

Run: `mvn test -pl aurix-salario -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 3: Verify gateway config**

Check `backend/aurix-gateway/src/main/resources/application.yml` has the salario route:
```yaml
- id: aurix-salario
  uri: http://localhost:8112
  predicates:
    - Path=/api/salario/**
  filters:
    - StripPrefix=0
```

- [ ] **Step 4: Verify parent pom has module**

Check `backend/pom.xml` includes `<module>aurix-salario</module>` in `<modules>`.

- [ ] **Step 5: Final commit if any changes needed**

```bash
git add -A
git commit -m "chore: full build verification for aurix-salario module"
```
