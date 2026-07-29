# Foundation Modules (Fase 1) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement 4 foundation microservices (Customer, KYC, Fraud, Notification) for the AURIX banking platform.

**Architecture:** Each module is an independent Spring Boot 4.1.0 microservice with PostgreSQL, Redis, Kafka. Communication via REST (synchronous) and Kafka events (asynchronous). Same conventions as existing 37 modules: `permitAll` security, `/api/<name>` context path, health endpoints, Docker images.

**Tech Stack:** Java 25, Spring Boot 4.1.0, Spring Data JPA, Spring Kafka, PostgreSQL, Redis, Testcontainers, JUnit 5, Mockito

## Global Constraints

- Java 25 (`maven.compiler.source/target = 25`)
- Spring Boot 4.1.0, Spring Cloud 2025.1.2
- Parent POM: `com.aurix.platform:aurix-platform:1.0.0`
- Shared lib: `com.aurix.platform:aurix-shared`
- Base image: `eclipse-temurin:25-jdk-jammy`
- All entities extend `com.aurix.platform.shared.entity.BaseEntity`
- All entities use `@Table(schema = "aurix")`
- All repositories extend `JpaRepository<Entity, Long>` with `@Repository`
- Services use `@Service` + `@Transactional`, constructor injection (no Lombok annotations — use delombok-style `@java.lang.SuppressWarnings("all")`)
- Controllers use `@RestController`, explicit constructors, Swagger annotations (`@Tag`, `@Operation`)
- Security: `@EnableWebSecurity` + `SecurityFilterChain` with `.anyRequest().permitAll()` + `csrf.disable()`
- Application class: `@SpringBootApplication(scanBasePackages = { "com.aurix.platform.<name>", "com.aurix.platform.shared" })`
- Kafka events are plain Strings (JSON payload), deserialized manually
- Docker image name: `infrastructure-aurix-<name>` (matches existing convention)
- Ports: 8123 (customer), 8124 (kyc), 8125 (fraud), 8126 (notification)

---

## File Structure

### New module files (4 modules × ~15 files each = ~60 files)

Each module follows this structure:
```
backend/aurix-<name>/
  pom.xml
  Dockerfile
  src/main/java/com/aurix/platform/<name>/
    Aurix<Name>Application.java
    config/SecurityConfig.java
    controller/HealthController.java
    controller/<Entity>Controller.java       (1-3 per module)
    service/<Entity>Service.java             (1-3 per module)
    repository/<Entity>Repository.java       (1-3 per module)
    entity/<Entity>.java                     (1-5 per module)
    config/KafkaConfig.java                  (if module has Kafka)
    consumer/<Event>Consumer.java            (if module consumes Kafka)
  src/main/resources/application.yml
  src/test/java/com/aurix/platform/<name>/
    <Module>IntegrationTest.java
    service/<Entity>ServiceTest.java         (1 per service)
```

### Modified files

- `backend/pom.xml` — add 4 `<module>` entries
- `infrastructure/docker-compose.yml` — add 4 services
- `infrastructure/traefik/dynamic.yml` — add 4 routers + services
- `aurix-tests/e2e/config.py` — add 4 health endpoints

---

### Task 0: Scaffold — Project registration + infra config

**Files:**
- Modify: `backend/pom.xml` (add 4 module entries)
- Modify: `infrastructure/docker-compose.yml` (add 4 services with ports, env, healthcheck, depends_on)
- Modify: `infrastructure/traefik/dynamic.yml` (add 4 routers + services)
- Modify: `aurix-tests/e2e/config.py` (add 4 health endpoints)

**Interfaces:**
- Consumes: nothing
- Produces: all infra wiring that later tasks depend on

- [ ] **Step 1: Add modules to parent POM**

Open `backend/pom.xml`. Add these `<module>` entries in alphabetical order (before `</modules>`):
```xml
        <module>aurix-customer</module>
        <module>aurix-fraud</module>
        <module>aurix-kyc</module>
        <module>aurix-notification</module>
```

- [ ] **Step 2: Add Traefik routes**

Open `infrastructure/traefik/dynamic.yml`. Under `http.routers:` add:
```yaml
    customer:
      rule: "PathPrefix(`/api/customer`)"
      service: customer
      entrypoints:
        - web
    kyc:
      rule: "PathPrefix(`/api/kyc`)"
      service: kyc
      entrypoints:
        - web
    fraud:
      rule: "PathPrefix(`/api/fraud`)"
      service: fraud
      entrypoints:
        - web
    notification:
      rule: "PathPrefix(`/api/notification`)"
      service: notification
      entrypoints:
        - web
```

Under `http.services:` add:
```yaml
    customer:
      loadBalancer:
        servers:
          - url: "http://aurix-customer:8123"
    kyc:
      loadBalancer:
        servers:
          - url: "http://aurix-kyc:8124"
    fraud:
      loadBalancer:
        servers:
          - url: "http://aurix-fraud:8125"
    notification:
      loadBalancer:
        servers:
          - url: "http://aurix-notification:8126"
```

- [ ] **Step 3: Add docker-compose services**

Open `infrastructure/docker-compose.yml`. Find the last `aurix-*` service (e.g., `aurix-cambio`). After its closing `networks:` line (or last key), insert 4 new service definitions:

```yaml
  aurix-customer:
    build:
      context: ../backend/aurix-customer
      dockerfile: Dockerfile
    container_name: aurix-customer
    ports:
      - "8123:8123"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SERVER_PORT: 8123
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aurix_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aurix_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aurix_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8123/api/customer/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 40s
    networks:
      - aurix-network

  aurix-kyc:
    build:
      context: ../backend/aurix-kyc
      dockerfile: Dockerfile
    container_name: aurix-kyc
    ports:
      - "8124:8124"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SERVER_PORT: 8124
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aurix_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aurix_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aurix_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8124/api/kyc/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 40s
    networks:
      - aurix-network

  aurix-fraud:
    build:
      context: ../backend/aurix-fraud
      dockerfile: Dockerfile
    container_name: aurix-fraud
    ports:
      - "8125:8125"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SERVER_PORT: 8125
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aurix_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aurix_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aurix_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8125/api/fraud/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 40s
    networks:
      - aurix-network

  aurix-notification:
    build:
      context: ../backend/aurix-notification
      dockerfile: Dockerfile
    container_name: aurix-notification
    ports:
      - "8126:8126"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SERVER_PORT: 8126
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aurix_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aurix_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aurix_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8126/api/notification/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 40s
    networks:
      - aurix-network
```

- [ ] **Step 4: Add E2E health endpoints**

Open `aurix-tests/e2e/config.py`. Add inside `SERVICE_HEALTH_ENDPOINTS` dict (before closing `}`):
```python
    "aurix-customer": "http://localhost:8123/api/customer/health",
    "aurix-kyc": "http://localhost:8124/api/kyc/health",
    "aurix-fraud": "http://localhost:8125/api/fraud/health",
    "aurix-notification": "http://localhost:8126/api/notification/health",
```

- [ ] **Step 5: Commit**

```bash
git add backend/pom.xml infrastructure/docker-compose.yml infrastructure/traefik/dynamic.yml aurix-tests/e2e/config.py
git commit -m "feat: register Foundation modules (customer, kyc, fraud, notification) in infra"
```

---

### Task 1: aurix-customer — Scaffold + Entities + Repository

**Files:**
- Create: `backend/aurix-customer/pom.xml`
- Create: `backend/aurix-customer/Dockerfile`
- Create: `backend/aurix-customer/src/main/java/com/aurix/platform/customer/AurixCustomerApplication.java`
- Create: `backend/aurix-customer/src/main/java/com/aurix/platform/customer/config/SecurityConfig.java`
- Create: `backend/aurix-customer/src/main/java/com/aurix/platform/customer/controller/HealthController.java`
- Create: `backend/aurix-customer/src/main/resources/application.yml`
- Create: `backend/aurix-customer/src/main/java/com/aurix/platform/customer/entity/Cliente.java`
- Create: `backend/aurix-customer/src/main/java/com/aurix/platform/customer/entity/ClientePF.java`
- Create: `backend/aurix-customer/src/main/java/com/aurix/platform/customer/entity/ClientePJ.java`
- Create: `backend/aurix-customer/src/main/java/com/aurix/platform/customer/entity/Endereco.java`
- Create: `backend/aurix-customer/src/main/java/com/aurix/platform/customer/entity/Contato.java`
- Create: `backend/aurix-customer/src/main/java/com/aurix/platform/customer/repository/ClienteRepository.java`
- Create: `backend/aurix-customer/src/main/java/com/aurix/platform/customer/repository/EnderecoRepository.java`
- Create: `backend/aurix-customer/src/main/java/com/aurix/platform/customer/repository/ContatoRepository.java`

**Interfaces:**
- Consumes: nothing
- Produces: `Cliente`, `Endereco`, `Contato` entities; `ClienteRepository`, `EnderecoRepository`, `ContatoRepository` interfaces

- [ ] **Step 1: Create module directory**

```bash
mkdir -p backend/aurix-customer/src/main/java/com/aurix/platform/customer/{config,controller,entity,repository,service}
mkdir -p backend/aurix-customer/src/main/resources
mkdir -p backend/aurix-customer/src/test/java/com/aurix/platform/customer
```

- [ ] **Step 2: Create pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.aurix.platform</groupId>
        <artifactId>aurix-platform</artifactId>
        <version>1.0.0</version>
    </parent>
    <artifactId>aurix-customer</artifactId>
    <packaging>jar</packaging>
    <name>AURIX Customer</name>
    <description>Cadastro e CRM de clientes</description>
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
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.3.0</version>
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

- [ ] **Step 3: Create Dockerfile**

```dockerfile
FROM eclipse-temurin:25-jdk-jammy
LABEL maintainer="AURIX Platform Team <dev@aurix.platform>"
LABEL description="AURIX Customer - Cadastro e CRM de clientes"
LABEL version="1.0.0"
RUN groupadd -r aurix && useradd -r -g aurix aurix
WORKDIR /app
COPY target/aurix-customer-1.0.0.jar app.jar
RUN chown aurix:aurix app.jar
USER aurix
EXPOSE 8123
ENV JAVA_OPTS="-Xmx512m -Xms256m -XX:+UseG1GC -XX:+UseContainerSupport"
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

- [ ] **Step 4: Create Application class**

```java
package com.aurix.platform.customer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication(scanBasePackages = { "com.aurix.platform.customer", "com.aurix.platform.shared" })
@EntityScan(basePackages = { "com.aurix.platform.customer.entity", "com.aurix.platform.shared.entity" })
@EnableJpaRepositories(basePackages = { "com.aurix.platform.customer.repository" })
@EnableCaching
@EnableScheduling
public class AurixCustomerApplication {
    public static void main(String[] args) {
        SpringApplication.run(AurixCustomerApplication.class, args);
    }
}
```

- [ ] **Step 5: Create SecurityConfig**

```java
package com.aurix.platform.customer.config;

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
        return http
                .authorizeHttpRequests(authz -> authz.anyRequest().permitAll())
                .csrf(csrf -> csrf.disable())
                .build();
    }
}
```

- [ ] **Step 6: Create HealthController**

```java
package com.aurix.platform.customer.controller;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import java.time.LocalDateTime;
import java.util.Map;

@RestController
@RequestMapping("/health")
@Tag(name = "Health", description = "API para verificacao de saude do servico Customer")
public class HealthController {
    @GetMapping
    @Operation(summary = "Health check", description = "Verifica se o servico Customer esta funcionando")
    public ResponseEntity<Map<String, Object>> health() {
        return ResponseEntity.ok(Map.of(
            "status", "UP",
            "service", "aurix-customer",
            "timestamp", LocalDateTime.now().toString(),
            "version", "1.0.0"
        ));
    }
}
```

- [ ] **Step 7: Create application.yml**

```yaml
server:
  port: 8123
  servlet:
    context-path: /api/customer

spring:
  application:
    name: aurix-customer
  profiles:
    active: dev
  datasource:
    url: jdbc:postgresql://localhost:5432/aurix_db
    username: aurix_user
    password: aurix_dev_password
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
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
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: aurix-customer-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer

logging:
  level:
    com.aurix.platform: DEBUG

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always

aurix:
  customer:
    version: "1.0.0"
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

---
spring:
  config:
    activate:
      on-profile: prod
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
```

- [ ] **Step 8: Create entities**

Cliente.java:
```java
package com.aurix.platform.customer.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.*;
import java.time.LocalDate;

@Entity
@Table(name = "clientes", schema = "aurix")
public class Cliente extends BaseEntity {
    @Column(nullable = false, length = 20)
    private String tipoPessoa;

    @Column(nullable = false, length = 20)
    private String segmento;

    @Column(nullable = false, length = 20)
    private String status;

    @Column(nullable = false, length = 200)
    private String nomeCompleto;

    @Column(length = 20)
    private String documento;

    @Column(length = 100)
    private String email;

    @Column(length = 20)
    private String telefone;

    @Column(length = 500)
    private String observacao;

    // --- PF fields ---
    @Column(length = 20)
    private String rg;

    private LocalDate dataNascimento;

    @Column(length = 30)
    private String estadoCivil;

    @Column(length = 50)
    private String nacionalidade;

    @Column(length = 100)
    private String profissao;

    private Double rendaMensal;

    // --- PJ fields ---
    @Column(length = 200)
    private String razaoSocial;

    @Column(length = 200)
    private String nomeFantasia;

    @Column(length = 50)
    private String inscricaoEstadual;

    private Double capitalSocial;

    public String getTipoPessoa() { return tipoPessoa; }
    public void setTipoPessoa(String tipoPessoa) { this.tipoPessoa = tipoPessoa; }
    public String getSegmento() { return segmento; }
    public void setSegmento(String segmento) { this.segmento = segmento; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public String getNomeCompleto() { return nomeCompleto; }
    public void setNomeCompleto(String nomeCompleto) { this.nomeCompleto = nomeCompleto; }
    public String getDocumento() { return documento; }
    public void setDocumento(String documento) { this.documento = documento; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getTelefone() { return telefone; }
    public void setTelefone(String telefone) { this.telefone = telefone; }
    public String getObservacao() { return observacao; }
    public void setObservacao(String observacao) { this.observacao = observacao; }
    public String getRg() { return rg; }
    public void setRg(String rg) { this.rg = rg; }
    public LocalDate getDataNascimento() { return dataNascimento; }
    public void setDataNascimento(LocalDate dataNascimento) { this.dataNascimento = dataNascimento; }
    public String getEstadoCivil() { return estadoCivil; }
    public void setEstadoCivil(String estadoCivil) { this.estadoCivil = estadoCivil; }
    public String getNacionalidade() { return nacionalidade; }
    public void setNacionalidade(String nacionalidade) { this.nacionalidade = nacionalidade; }
    public String getProfissao() { return profissao; }
    public void setProfissao(String profissao) { this.profissao = profissao; }
    public Double getRendaMensal() { return rendaMensal; }
    public void setRendaMensal(Double rendaMensal) { this.rendaMensal = rendaMensal; }
    public String getRazaoSocial() { return razaoSocial; }
    public void setRazaoSocial(String razaoSocial) { this.razaoSocial = razaoSocial; }
    public String getNomeFantasia() { return nomeFantasia; }
    public void setNomeFantasia(String nomeFantasia) { this.nomeFantasia = nomeFantasia; }
    public String getInscricaoEstadual() { return inscricaoEstadual; }
    public void setInscricaoEstadual(String inscricaoEstadual) { this.inscricaoEstadual = inscricaoEstadual; }
    public Double getCapitalSocial() { return capitalSocial; }
    public void setCapitalSocial(Double capitalSocial) { this.capitalSocial = capitalSocial; }
}
```

Endereco.java:
```java
package com.aurix.platform.customer.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.*;

@Entity
@Table(name = "enderecos", schema = "aurix")
public class Endereco extends BaseEntity {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cliente_id", nullable = false)
    private Cliente cliente;

    @Column(nullable = false, length = 20)
    private String tipo;

    @Column(nullable = false, length = 10)
    private String cep;

    @Column(nullable = false, length = 200)
    private String logradouro;

    @Column(length = 20)
    private String numero;

    @Column(length = 100)
    private String complemento;

    @Column(length = 100)
    private String bairro;

    @Column(nullable = false, length = 100)
    private String cidade;

    @Column(nullable = false, length = 2)
    private String uf;

    @Column(name = "is_principal")
    private Boolean principal;

    public Cliente getCliente() { return cliente; }
    public void setCliente(Cliente cliente) { this.cliente = cliente; }
    public String getTipo() { return tipo; }
    public void setTipo(String tipo) { this.tipo = tipo; }
    public String getCep() { return cep; }
    public void setCep(String cep) { this.cep = cep; }
    public String getLogradouro() { return logradouro; }
    public void setLogradouro(String logradouro) { this.logradouro = logradouro; }
    public String getNumero() { return numero; }
    public void setNumero(String numero) { this.numero = numero; }
    public String getComplemento() { return complemento; }
    public void setComplemento(String complemento) { this.complemento = complemento; }
    public String getBairro() { return bairro; }
    public void setBairro(String bairro) { this.bairro = bairro; }
    public String getCidade() { return cidade; }
    public void setCidade(String cidade) { this.cidade = cidade; }
    public String getUf() { return uf; }
    public void setUf(String uf) { this.uf = uf; }
    public Boolean getPrincipal() { return principal; }
    public void setPrincipal(Boolean principal) { this.principal = principal; }
}
```

Contato.java:
```java
package com.aurix.platform.customer.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.*;

@Entity
@Table(name = "contatos", schema = "aurix")
public class Contato extends BaseEntity {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cliente_id", nullable = false)
    private Cliente cliente;

    @Column(nullable = false, length = 20)
    private String tipo;

    @Column(nullable = false, length = 200)
    private String valor;

    @Column(name = "is_preferencial")
    private Boolean preferencial;

    public Cliente getCliente() { return cliente; }
    public void setCliente(Cliente cliente) { this.cliente = cliente; }
    public String getTipo() { return tipo; }
    public void setTipo(String tipo) { this.tipo = tipo; }
    public String getValor() { return valor; }
    public void setValor(String valor) { this.valor = valor; }
    public Boolean getPreferencial() { return preferencial; }
    public void setPreferencial(Boolean preferencial) { this.preferencial = preferencial; }
}
```

- [ ] **Step 9: Create repositories**

```java
package com.aurix.platform.customer.repository;

import com.aurix.platform.customer.entity.Cliente;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;
import java.util.Optional;

@Repository
public interface ClienteRepository extends JpaRepository<Cliente, Long> {
    Optional<Cliente> findByDocumento(String documento);
    List<Cliente> findBySegmento(String segmento);
    List<Cliente> findByStatus(String status);
    List<Cliente> findByNomeCompletoContainingIgnoreCase(String nome);
}
```

```java
package com.aurix.platform.customer.repository;

import com.aurix.platform.customer.entity.Endereco;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface EnderecoRepository extends JpaRepository<Endereco, Long> {
    List<Endereco> findByClienteId(Long clienteId);
}
```

```java
package com.aurix.platform.customer.repository;

import com.aurix.platform.customer.entity.Contato;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface ContatoRepository extends JpaRepository<Contato, Long> {
    List<Contato> findByClienteId(Long clienteId);
}
```

- [ ] **Step 10: Compile to verify**

```bash
mvn clean compile -pl aurix-customer -am -q
```
Expected: BUILD SUCCESS (no errors)

- [ ] **Step 11: Commit**

```bash
git add backend/aurix-customer/
git commit -m "feat(customer): scaffold, entities, and repositories"
```

---

### Task 2: aurix-customer — Service + Controllers + Tests

**Files:**
- Create: `backend/aurix-customer/src/main/java/com/aurix/platform/customer/service/ClienteService.java`
- Create: `backend/aurix-customer/src/main/java/com/aurix/platform/customer/service/ClienteProducer.java`
- Create: `backend/aurix-customer/src/main/java/com/aurix/platform/customer/controller/ClienteController.java`
- Create: `backend/aurix-customer/src/test/java/com/aurix/platform/customer/service/ClienteServiceTest.java`
- Create: `backend/aurix-customer/src/test/java/com/aurix/platform/customer/AurixCustomerApplicationTest.java`
- Create: `backend/aurix-customer/src/main/java/com/aurix/platform/customer/config/KafkaConfig.java`

**Interfaces:**
- Consumes: `ClienteRepository`, `EnderecoRepository`, `ContatoRepository`
- Produces: REST endpoints POST/GET/PATCH `/api/clientes`, Kafka event `cliente.criado`

- [ ] **Step 1: Create KafkaConfig**

```java
package com.aurix.platform.customer.config;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class KafkaConfig {
    @Bean
    public NewTopic clienteCriadoTopic() {
        return TopicBuilder.name("cliente.criado").partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic clienteAtualizadoTopic() {
        return TopicBuilder.name("cliente.atualizado").partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic clienteStatusAlteradoTopic() {
        return TopicBuilder.name("cliente.status.alterado").partitions(3).replicas(1).build();
    }
}
```

- [ ] **Step 2: Create ClienteProducer**

```java
package com.aurix.platform.customer.service;

import com.aurix.platform.customer.entity.Cliente;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class ClienteProducer {
    private final KafkaTemplate<String, String> kafkaTemplate;

    public ClienteProducer(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void clienteCriado(Cliente cliente) {
        String payload = String.format(
            "{\"clienteId\":%d,\"documento\":\"%s\",\"nome\":\"%s\",\"tipoPessoa\":\"%s\",\"segmento\":\"%s\"}",
            cliente.getId(), cliente.getDocumento(), cliente.getNomeCompleto(),
            cliente.getTipoPessoa(), cliente.getSegmento()
        );
        kafkaTemplate.send("cliente.criado", String.valueOf(cliente.getId()), payload);
    }

    public void clienteAtualizado(Cliente cliente) {
        String payload = String.format(
            "{\"clienteId\":%d,\"documento\":\"%s\",\"status\":\"%s\"}",
            cliente.getId(), cliente.getDocumento(), cliente.getStatus()
        );
        kafkaTemplate.send("cliente.atualizado", String.valueOf(cliente.getId()), payload);
    }

    public void clienteStatusAlterado(Cliente cliente, String statusAnterior) {
        String payload = String.format(
            "{\"clienteId\":%d,\"statusAnterior\":\"%s\",\"statusAtual\":\"%s\"}",
            cliente.getId(), statusAnterior, cliente.getStatus()
        );
        kafkaTemplate.send("cliente.status.alterado", String.valueOf(cliente.getId()), payload);
    }
}
```

- [ ] **Step 3: Create ClienteService**

```java
package com.aurix.platform.customer.service;

import com.aurix.platform.customer.entity.Cliente;
import com.aurix.platform.customer.entity.Endereco;
import com.aurix.platform.customer.entity.Contato;
import com.aurix.platform.customer.repository.ClienteRepository;
import com.aurix.platform.customer.repository.EnderecoRepository;
import com.aurix.platform.customer.repository.ContatoRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Service
@Transactional
public class ClienteService {
    private final ClienteRepository clienteRepository;
    private final EnderecoRepository enderecoRepository;
    private final ContatoRepository contatoRepository;
    private final ClienteProducer clienteProducer;

    public ClienteService(ClienteRepository clienteRepository, EnderecoRepository enderecoRepository,
                          ContatoRepository contatoRepository, ClienteProducer clienteProducer) {
        this.clienteRepository = clienteRepository;
        this.enderecoRepository = enderecoRepository;
        this.contatoRepository = contatoRepository;
        this.clienteProducer = clienteProducer;
    }

    @Transactional(readOnly = true)
    public List<Cliente> listar(String segmento, String status) {
        if (segmento != null) return clienteRepository.findBySegmento(segmento);
        if (status != null) return clienteRepository.findByStatus(status);
        return clienteRepository.findAll();
    }

    @Transactional(readOnly = true)
    public Cliente buscarPorId(Long id) {
        return clienteRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Cliente nao encontrado: " + id));
    }

    @Transactional(readOnly = true)
    public Cliente buscarPorDocumento(String documento) {
        return clienteRepository.findByDocumento(documento)
                .orElseThrow(() -> new IllegalArgumentException("Cliente nao encontrado: " + documento));
    }

    public Cliente criar(Cliente cliente) {
        cliente.setStatus("ATIVO");
        Cliente saved = clienteRepository.save(cliente);
        clienteProducer.clienteCriado(saved);
        return saved;
    }

    public Cliente atualizar(Long id, Cliente dados) {
        Cliente existente = buscarPorId(id);
        String statusAnterior = existente.getStatus();
        if (dados.getNomeCompleto() != null) existente.setNomeCompleto(dados.getNomeCompleto());
        if (dados.getEmail() != null) existente.setEmail(dados.getEmail());
        if (dados.getTelefone() != null) existente.setTelefone(dados.getTelefone());
        if (dados.getSegmento() != null) existente.setSegmento(dados.getSegmento());
        if (dados.getStatus() != null) existente.setStatus(dados.getStatus());
        if (dados.getObservacao() != null) existente.setObservacao(dados.getObservacao());
        Cliente saved = clienteRepository.save(existente);
        clienteProducer.clienteAtualizado(saved);
        if (dados.getStatus() != null && !dados.getStatus().equals(statusAnterior)) {
            clienteProducer.clienteStatusAlterado(saved, statusAnterior);
        }
        return saved;
    }

    @Transactional(readOnly = true)
    public List<Endereco> listarEnderecos(Long clienteId) {
        return enderecoRepository.findByClienteId(clienteId);
    }

    public Endereco adicionarEndereco(Long clienteId, Endereco endereco) {
        Cliente cliente = buscarPorId(clienteId);
        endereco.setCliente(cliente);
        return enderecoRepository.save(endereco);
    }

    @Transactional(readOnly = true)
    public List<Contato> listarContatos(Long clienteId) {
        return contatoRepository.findByClienteId(clienteId);
    }

    public Contato adicionarContato(Long clienteId, Contato contato) {
        Cliente cliente = buscarPorId(clienteId);
        contato.setCliente(cliente);
        return contatoRepository.save(contato);
    }
}
```

- [ ] **Step 4: Create ClienteController**

```java
package com.aurix.platform.customer.controller;

import com.aurix.platform.customer.entity.Cliente;
import com.aurix.platform.customer.entity.Contato;
import com.aurix.platform.customer.entity.Endereco;
import com.aurix.platform.customer.service.ClienteService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/clientes")
@Tag(name = "Clientes", description = "Gerenciamento de clientes")
public class ClienteController {
    private final ClienteService clienteService;

    public ClienteController(ClienteService clienteService) {
        this.clienteService = clienteService;
    }

    @PostMapping
    @Operation(summary = "Criar cliente")
    public ResponseEntity<Cliente> criar(@Valid @RequestBody Cliente cliente) {
        return ResponseEntity.status(HttpStatus.CREATED).body(clienteService.criar(cliente));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Buscar cliente por ID")
    public ResponseEntity<Cliente> buscarPorId(@PathVariable Long id) {
        return ResponseEntity.ok(clienteService.buscarPorId(id));
    }

    @GetMapping("/documento/{documento}")
    @Operation(summary = "Buscar cliente por CPF/CNPJ")
    public ResponseEntity<Cliente> buscarPorDocumento(@PathVariable String documento) {
        return ResponseEntity.ok(clienteService.buscarPorDocumento(documento));
    }

    @PatchMapping("/{id}")
    @Operation(summary = "Atualizar cliente")
    public ResponseEntity<Cliente> atualizar(@PathVariable Long id, @RequestBody Cliente cliente) {
        return ResponseEntity.ok(clienteService.atualizar(id, cliente));
    }

    @GetMapping
    @Operation(summary = "Listar clientes")
    public ResponseEntity<List<Cliente>> listar(
            @RequestParam(required = false) String segmento,
            @RequestParam(required = false) String status) {
        return ResponseEntity.ok(clienteService.listar(segmento, status));
    }

    @GetMapping("/{id}/enderecos")
    @Operation(summary = "Listar enderecos do cliente")
    public ResponseEntity<List<Endereco>> listarEnderecos(@PathVariable Long id) {
        return ResponseEntity.ok(clienteService.listarEnderecos(id));
    }

    @PostMapping("/{id}/enderecos")
    @Operation(summary = "Adicionar endereco")
    public ResponseEntity<Endereco> adicionarEndereco(@PathVariable Long id, @Valid @RequestBody Endereco endereco) {
        return ResponseEntity.status(HttpStatus.CREATED).body(clienteService.adicionarEndereco(id, endereco));
    }

    @GetMapping("/{id}/contatos")
    @Operation(summary = "Listar contatos do cliente")
    public ResponseEntity<List<Contato>> listarContatos(@PathVariable Long id) {
        return ResponseEntity.ok(clienteService.listarContatos(id));
    }

    @PostMapping("/{id}/contatos")
    @Operation(summary = "Adicionar contato")
    public ResponseEntity<Contato> adicionarContato(@PathVariable Long id, @Valid @RequestBody Contato contato) {
        return ResponseEntity.status(HttpStatus.CREATED).body(clienteService.adicionarContato(id, contato));
    }
}
```

- [ ] **Step 5: Create unit test for ClienteService**

```java
package com.aurix.platform.customer.service;

import com.aurix.platform.customer.entity.Cliente;
import com.aurix.platform.customer.repository.ClienteRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import java.util.Optional;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class ClienteServiceTest {
    @Mock private ClienteRepository clienteRepository;
    @Mock private com.aurix.platform.customer.repository.EnderecoRepository enderecoRepository;
    @Mock private com.aurix.platform.customer.repository.ContatoRepository contatoRepository;
    @Mock private ClienteProducer clienteProducer;
    @InjectMocks private ClienteService clienteService;

    @Test
    void deveCriarClienteComStatusAtivo() {
        Cliente cliente = new Cliente();
        cliente.setNomeCompleto("Joao Silva");
        cliente.setDocumento("12345678901");
        cliente.setTipoPessoa("PF");
        cliente.setSegmento("PF");

        when(clienteRepository.save(any())).thenAnswer(inv -> {
            Cliente saved = inv.getArgument(0);
            saved.setId(1L);
            return saved;
        });

        Cliente resultado = clienteService.criar(cliente);

        assertEquals("ATIVO", resultado.getStatus());
        assertEquals("Joao Silva", resultado.getNomeCompleto());
        verify(clienteProducer).clienteCriado(any());
    }

    @Test
    void deveLancarExcecaoQuandoClienteNaoEncontrado() {
        when(clienteRepository.findById(99L)).thenReturn(Optional.empty());
        assertThrows(IllegalArgumentException.class, () -> clienteService.buscarPorId(99L));
    }
}
```

- [ ] **Step 6: Create integration test**

```java
package com.aurix.platform.customer;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

@SpringBootTest
@ActiveProfiles("test")
class AurixCustomerApplicationTest {
    @Test
    void contextLoads() {
    }
}
```

- [ ] **Step 7: Build and run tests**

```bash
mvn clean test -pl aurix-customer -am
```
Expected: BUILD SUCCESS, tests pass

- [ ] **Step 8: Commit**

```bash
git add backend/aurix-customer/
git commit -m "feat(customer): service, controller, Kafka producer, and tests"
```

---

### Task 3: aurix-kyc — Full module

**Files:** (same structure as Task 1+2, but for KYC)
- Create: `backend/aurix-kyc/pom.xml`
- Create: `backend/aurix-kyc/Dockerfile`
- Create: `backend/aurix-kyc/src/main/java/com/aurix/platform/kyc/AurixKycApplication.java`
- Create: `backend/aurix-kyc/src/main/java/com/aurix/platform/kyc/config/SecurityConfig.java`
- Create: `backend/aurix-kyc/src/main/java/com/aurix/platform/kyc/config/KafkaConfig.java`
- Create: `backend/aurix-kyc/src/main/java/com/aurix/platform/kyc/controller/HealthController.java`
- Create: `backend/aurix-kyc/src/main/java/com/aurix/platform/kyc/controller/SolicitacaoKycController.java`
- Create: `backend/aurix-kyc/src/main/java/com/aurix/platform/kyc/service/SolicitacaoKycService.java`
- Create: `backend/aurix-kyc/src/main/java/com/aurix/platform/kyc/service/KycConsumer.java`
- Create: `backend/aurix-kyc/src/main/java/com/aurix/platform/kyc/service/KycProducer.java`
- Create: `backend/aurix-kyc/src/main/java/com/aurix/platform/kyc/entity/SolicitacaoKYC.java`
- Create: `backend/aurix-kyc/src/main/java/com/aurix/platform/kyc/entity/DocumentoKYC.java`
- Create: `backend/aurix-kyc/src/main/java/com/aurix/platform/kyc/entity/ScoreKYC.java`
- Create: `backend/aurix-kyc/src/main/java/com/aurix/platform/kyc/repository/SolicitacaoKycRepository.java`
- Create: `backend/aurix-kyc/src/main/java/com/aurix/platform/kyc/repository/DocumentoKycRepository.java`
- Create: `backend/aurix-kyc/src/main/java/com/aurix/platform/kyc/repository/ScoreKycRepository.java`
- Create: `backend/aurix-kyc/src/main/resources/application.yml`
- Create: `backend/aurix-kyc/src/test/java/com/aurix/platform/kyc/AurixKycApplicationTest.java`
- Create: `backend/aurix-kyc/src/test/java/com/aurix/platform/kyc/service/SolicitacaoKycServiceTest.java`

**Interfaces:**
- Consumes: Kafka `cliente.criado` event
- Produces: Kafka `kyc.aprovado`, `kyc.rejeitado` events
- REST endpoints for KYC solicitation management

- [ ] **Step 1: Create directory structure**

```bash
mkdir -p backend/aurix-kyc/src/main/java/com/aurix/platform/kyc/{config,controller,entity,repository,service}
mkdir -p backend/aurix-kyc/src/main/resources
mkdir -p backend/aurix-kyc/src/test/java/com/aurix/platform/kyc
```

- [ ] **Step 2: Create scaffold files (pom.xml, Dockerfile, Application, SecurityConfig, HealthController, application.yml)**

Follow the exact same patterns as Task 1 but substitute:
- `aurix-customer` → `aurix-kyc`
- `customer` → `kyc`
- `Customer` → `Kyc`
- `8123` → `8124`
- `/api/customer` → `/api/kyc`
- `aurix-customer-group` → `aurix-kyc-group`

pom.xml: same as customer but with `artifactId: aurix-kyc`, `name: AURIX KYC`, `description: Validacao documental e compliance`

Dockerfile: same as customer but `EXPOSE 8124`

Application class:
```java
package com.aurix.platform.kyc;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication(scanBasePackages = { "com.aurix.platform.kyc", "com.aurix.platform.shared" })
@EntityScan(basePackages = { "com.aurix.platform.kyc.entity", "com.aurix.platform.shared.entity" })
@EnableJpaRepositories(basePackages = { "com.aurix.platform.kyc.repository" })
@EnableCaching
@EnableScheduling
public class AurixKycApplication {
    public static void main(String[] args) {
        SpringApplication.run(AurixKycApplication.class, args);
    }
}
```

application.yml: same structure as customer, with `server.port: 8124`, `context-path: /api/kyc`, `group-id: aurix-kyc-group`

- [ ] **Step 3: Create entities**

SolicitacaoKYC.java:
```java
package com.aurix.platform.kyc.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "solicitacoes_kyc", schema = "aurix")
public class SolicitacaoKYC extends BaseEntity {
    @Column(name = "cliente_id", nullable = false)
    private Long clienteId;

    @Column(nullable = false, length = 20)
    private String status;

    @Column(name = "score_risco")
    private Integer scoreRisco;

    @Column(name = "data_solicitacao", nullable = false)
    private LocalDateTime dataSolicitacao;

    @Column(name = "data_conclusao")
    private LocalDateTime dataConclusao;

    @Column(length = 500)
    private String observacao;

    public Long getClienteId() { return clienteId; }
    public void setClienteId(Long clienteId) { this.clienteId = clienteId; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public Integer getScoreRisco() { return scoreRisco; }
    public void setScoreRisco(Integer scoreRisco) { this.scoreRisco = scoreRisco; }
    public LocalDateTime getDataSolicitacao() { return dataSolicitacao; }
    public void setDataSolicitacao(LocalDateTime dataSolicitacao) { this.dataSolicitacao = dataSolicitacao; }
    public LocalDateTime getDataConclusao() { return dataConclusao; }
    public void setDataConclusao(LocalDateTime dataConclusao) { this.dataConclusao = dataConclusao; }
    public String getObservacao() { return observacao; }
    public void setObservacao(String observacao) { this.observacao = observacao; }
}
```

DocumentoKYC.java:
```java
package com.aurix.platform.kyc.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.*;

@Entity
@Table(name = "documentos_kyc", schema = "aurix")
public class DocumentoKYC extends BaseEntity {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "solicitacao_id", nullable = false)
    private SolicitacaoKYC solicitacao;

    @Column(nullable = false, length = 30)
    private String tipo;

    @Column(name = "arquivo_ref", length = 500)
    private String arquivoRef;

    @Column(nullable = false, length = 20)
    private String status;

    @Column(length = 500)
    private String motivoRejeicao;

    public SolicitacaoKYC getSolicitacao() { return solicitacao; }
    public void setSolicitacao(SolicitacaoKYC solicitacao) { this.solicitacao = solicitacao; }
    public String getTipo() { return tipo; }
    public void setTipo(String tipo) { this.tipo = tipo; }
    public String getArquivoRef() { return arquivoRef; }
    public void setArquivoRef(String arquivoRef) { this.arquivoRef = arquivoRef; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public String getMotivoRejeicao() { return motivoRejeicao; }
    public void setMotivoRejeicao(String motivoRejeicao) { this.motivoRejeicao = motivoRejeicao; }
}
```

ScoreKYC.java:
```java
package com.aurix.platform.kyc.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.*;

@Entity
@Table(name = "scores_kyc", schema = "aurix")
public class ScoreKYC extends BaseEntity {
    @Column(name = "cliente_id", nullable = false)
    private Long clienteId;

    @Column(name = "score_geral")
    private Integer scoreGeral;

    @Column(name = "score_documento")
    private Integer scoreDocumento;

    @Column(name = "score_biometria")
    private Integer scoreBiometria;

    @Column(name = "score_pep")
    private Integer scorePep;

    @Column(name = "score_origem_fundos")
    private Integer scoreOrigemFundos;

    public Long getClienteId() { return clienteId; }
    public void setClienteId(Long clienteId) { this.clienteId = clienteId; }
    public Integer getScoreGeral() { return scoreGeral; }
    public void setScoreGeral(Integer scoreGeral) { this.scoreGeral = scoreGeral; }
    public Integer getScoreDocumento() { return scoreDocumento; }
    public void setScoreDocumento(Integer scoreDocumento) { this.scoreDocumento = scoreDocumento; }
    public Integer getScoreBiometria() { return scoreBiometria; }
    public void setScoreBiometria(Integer scoreBiometria) { this.scoreBiometria = scoreBiometria; }
    public Integer getScorePep() { return scorePep; }
    public void setScorePep(Integer scorePep) { this.scorePep = scorePep; }
    public Integer getScoreOrigemFundos() { return scoreOrigemFundos; }
    public void setScoreOrigemFundos(Integer scoreOrigemFundos) { this.scoreOrigemFundos = scoreOrigemFundos; }
}
```

- [ ] **Step 4: Create repositories**

```java
package com.aurix.platform.kyc.repository;

import com.aurix.platform.kyc.entity.SolicitacaoKYC;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;
import java.util.Optional;

@Repository
public interface SolicitacaoKycRepository extends JpaRepository<SolicitacaoKYC, Long> {
    List<SolicitacaoKYC> findByClienteId(Long clienteId);
    List<SolicitacaoKYC> findByStatus(String status);
    Optional<SolicitacaoKYC> findTopByClienteIdOrderByDataSolicitacaoDesc(Long clienteId);
}
```

```java
package com.aurix.platform.kyc.repository;

import com.aurix.platform.kyc.entity.DocumentoKYC;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface DocumentoKycRepository extends JpaRepository<DocumentoKYC, Long> {
    List<DocumentoKYC> findBySolicitacaoId(Long solicitacaoId);
}
```

```java
package com.aurix.platform.kyc.repository;

import com.aurix.platform.kyc.entity.ScoreKYC;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.Optional;

@Repository
public interface ScoreKycRepository extends JpaRepository<ScoreKYC, Long> {
    Optional<ScoreKYC> findByClienteId(Long clienteId);
}
```

- [ ] **Step 5: Create Kafka config, consumer, and producer**

KafkaConfig.java:
```java
package com.aurix.platform.kyc.config;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class KafkaConfig {
    @Bean
    public NewTopic kycAprovadoTopic() {
        return TopicBuilder.name("kyc.aprovado").partitions(3).replicas(1).build();
    }
    @Bean
    public NewTopic kycRejeitadoTopic() {
        return TopicBuilder.name("kyc.rejeitado").partitions(3).replicas(1).build();
    }
}
```

KycConsumer.java:
```java
package com.aurix.platform.kyc.service;

import com.aurix.platform.kyc.entity.SolicitacaoKYC;
import com.aurix.platform.kyc.repository.SolicitacaoKycRepository;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;
import java.time.LocalDateTime;

@Service
public class KycConsumer {
    private final SolicitacaoKycRepository repository;
    private final ObjectMapper objectMapper;

    public KycConsumer(SolicitacaoKycRepository repository, ObjectMapper objectMapper) {
        this.repository = repository;
        this.objectMapper = objectMapper;
    }

    @KafkaListener(topics = "cliente.criado", groupId = "aurix-kyc-group")
    public void onClienteCriado(String message) {
        try {
            JsonNode json = objectMapper.readTree(message);
            Long clienteId = json.get("clienteId").asLong();

            SolicitacaoKYC solicitacao = new SolicitacaoKYC();
            solicitacao.setClienteId(clienteId);
            solicitacao.setStatus("PENDENTE");
            solicitacao.setDataSolicitacao(LocalDateTime.now());
            repository.save(solicitacao);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Erro ao processar mensagem Kafka", e);
        }
    }
}
```

KycProducer.java:
```java
package com.aurix.platform.kyc.service;

import com.aurix.platform.kyc.entity.SolicitacaoKYC;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class KycProducer {
    private final KafkaTemplate<String, String> kafkaTemplate;

    public KycProducer(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void kycAprovado(SolicitacaoKYC solicitacao) {
        String payload = String.format(
            "{\"clienteId\":%d,\"solicitacaoId\":%d,\"scoreRisco\":%d,\"status\":\"APROVADO\"}",
            solicitacao.getClienteId(), solicitacao.getId(), solicitacao.getScoreRisco()
        );
        kafkaTemplate.send("kyc.aprovado", String.valueOf(solicitacao.getClienteId()), payload);
    }

    public void kycRejeitado(SolicitacaoKYC solicitacao, String motivo) {
        String payload = String.format(
            "{\"clienteId\":%d,\"solicitacaoId\":%d,\"motivo\":\"%s\",\"status\":\"REJEITADO\"}",
            solicitacao.getClienteId(), solicitacao.getId(), motivo
        );
        kafkaTemplate.send("kyc.rejeitado", String.valueOf(solicitacao.getClienteId()), payload);
    }
}
```

- [ ] **Step 6: Create service**

SolicitacaoKycService.java:
```java
package com.aurix.platform.kyc.service;

import com.aurix.platform.kyc.entity.DocumentoKYC;
import com.aurix.platform.kyc.entity.ScoreKYC;
import com.aurix.platform.kyc.entity.SolicitacaoKYC;
import com.aurix.platform.kyc.repository.DocumentoKycRepository;
import com.aurix.platform.kyc.repository.ScoreKycRepository;
import com.aurix.platform.kyc.repository.SolicitacaoKycRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDateTime;
import java.util.List;

@Service
@Transactional
public class SolicitacaoKycService {
    private final SolicitacaoKycRepository solicitacaoRepository;
    private final DocumentoKycRepository documentoRepository;
    private final ScoreKycRepository scoreRepository;
    private final KycProducer kycProducer;

    public SolicitacaoKycService(SolicitacaoKycRepository solicitacaoRepository,
                                  DocumentoKycRepository documentoRepository,
                                  ScoreKycRepository scoreRepository,
                                  KycProducer kycProducer) {
        this.solicitacaoRepository = solicitacaoRepository;
        this.documentoRepository = documentoRepository;
        this.scoreRepository = scoreRepository;
        this.kycProducer = kycProducer;
    }

    @Transactional(readOnly = true)
    public SolicitacaoKYC buscarPorId(Long id) {
        return solicitacaoRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Solicitacao KYC nao encontrada: " + id));
    }

    @Transactional(readOnly = true)
    public List<SolicitacaoKYC> listarPorCliente(Long clienteId) {
        return solicitacaoRepository.findByClienteId(clienteId);
    }

    @Transactional(readOnly = true)
    public List<SolicitacaoKYC> listarPorStatus(String status) {
        return solicitacaoRepository.findByStatus(status);
    }

    public SolicitacaoKYC criarSolicitacao(Long clienteId) {
        SolicitacaoKYC solicitacao = new SolicitacaoKYC();
        solicitacao.setClienteId(clienteId);
        solicitacao.setStatus("PENDENTE");
        solicitacao.setDataSolicitacao(LocalDateTime.now());
        return solicitacaoRepository.save(solicitacao);
    }

    public DocumentoKYC anexarDocumento(Long solicitacaoId, DocumentoKYC documento) {
        SolicitacaoKYC solicitacao = buscarPorId(solicitacaoId);
        documento.setSolicitacao(solicitacao);
        documento.setStatus("PENDENTE");
        return documentoRepository.save(documento);
    }

    public SolicitacaoKYC aprovar(Long id) {
        SolicitacaoKYC solicitacao = buscarPorId(id);
        solicitacao.setStatus("APROVADO");
        solicitacao.setDataConclusao(LocalDateTime.now());
        solicitacao.setScoreRisco(10);
        solicitacaoRepository.save(solicitacao);

        ScoreKYC score = new ScoreKYC();
        score.setClienteId(solicitacao.getClienteId());
        score.setScoreGeral(85);
        score.setScoreDocumento(90);
        score.setScoreBiometria(80);
        score.setScorePep(100);
        score.setScoreOrigemFundos(75);
        scoreRepository.save(score);

        kycProducer.kycAprovado(solicitacao);
        return solicitacao;
    }

    public SolicitacaoKYC rejeitar(Long id, String motivo) {
        SolicitacaoKYC solicitacao = buscarPorId(id);
        solicitacao.setStatus("REJEITADO");
        solicitacao.setDataConclusao(LocalDateTime.now());
        solicitacao.setObservacao(motivo);
        solicitacaoRepository.save(solicitacao);
        kycProducer.kycRejeitado(solicitacao, motivo);
        return solicitacao;
    }

    @Transactional(readOnly = true)
    public ScoreKYC consultarScore(Long clienteId) {
        return scoreRepository.findByClienteId(clienteId)
                .orElseThrow(() -> new IllegalArgumentException("Score nao encontrado para cliente: " + clienteId));
    }
}
```

- [ ] **Step 7: Create controller**

SolicitacaoKycController.java:
```java
package com.aurix.platform.kyc.controller;

import com.aurix.platform.kyc.entity.DocumentoKYC;
import com.aurix.platform.kyc.entity.ScoreKYC;
import com.aurix.platform.kyc.entity.SolicitacaoKYC;
import com.aurix.platform.kyc.service.SolicitacaoKycService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/kyc")
@Tag(name = "KYC", description = "Validacao documental e compliance")
public class SolicitacaoKycController {
    private final SolicitacaoKycService service;

    public SolicitacaoKycController(SolicitacaoKycService service) {
        this.service = service;
    }

    @PostMapping("/solicitacoes")
    @Operation(summary = "Iniciar KYC para cliente")
    public ResponseEntity<SolicitacaoKYC> criar(@RequestParam Long clienteId) {
        return ResponseEntity.status(HttpStatus.CREATED).body(service.criarSolicitacao(clienteId));
    }

    @GetMapping("/solicitacoes/{id}")
    @Operation(summary = "Status da solicitacao")
    public ResponseEntity<SolicitacaoKYC> buscar(@PathVariable Long id) {
        return ResponseEntity.ok(service.buscarPorId(id));
    }

    @GetMapping("/solicitacoes/cliente/{clienteId}")
    @Operation(summary = "Historico KYC do cliente")
    public ResponseEntity<List<SolicitacaoKYC>> listarPorCliente(@PathVariable Long clienteId) {
        return ResponseEntity.ok(service.listarPorCliente(clienteId));
    }

    @PostMapping("/documentos")
    @Operation(summary = "Anexar documento")
    public ResponseEntity<DocumentoKYC> anexarDocumento(@RequestParam Long solicitacaoId,
                                                         @Valid @RequestBody DocumentoKYC documento) {
        return ResponseEntity.status(HttpStatus.CREATED).body(service.anexarDocumento(solicitacaoId, documento));
    }

    @PostMapping("/solicitacoes/{id}/aprovar")
    @Operation(summary = "Aprovar KYC")
    public ResponseEntity<SolicitacaoKYC> aprovar(@PathVariable Long id) {
        return ResponseEntity.ok(service.aprovar(id));
    }

    @PostMapping("/solicitacoes/{id}/rejeitar")
    @Operation(summary = "Rejeitar KYC")
    public ResponseEntity<SolicitacaoKYC> rejeitar(@PathVariable Long id, @RequestParam String motivo) {
        return ResponseEntity.ok(service.rejeitar(id, motivo));
    }

    @GetMapping("/score/{clienteId}")
    @Operation(summary = "Consultar score KYC")
    public ResponseEntity<ScoreKYC> consultarScore(@PathVariable Long clienteId) {
        return ResponseEntity.ok(service.consultarScore(clienteId));
    }
}
```

- [ ] **Step 8: Build and run tests**

```bash
mvn clean test -pl aurix-kyc -am
```
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add backend/aurix-kyc/
git commit -m "feat(kyc): full module with Kafka consumer/producer, entities, service, controller"
```

---

### Task 4: aurix-fraud — Full module

**Files:** (follow same pattern as Task 3)
- Create `backend/aurix-fraud/` with full scaffold (pom.xml, Dockerfile, Application, SecurityConfig, HealthController, application.yml)
- Create entities: `RegraFraude`, `ScoreTransacao`, `OcorrenciaFraude`, `BloqueioPreventivo`
- Create repositories for each entity
- Create Kafka topics: `fraude.transacao.bloqueada`, `fraude.ocorrencia.criada`, `fraude.score.alterado`
- Create consumers: listens to `cliente.criado`, `kyc.aprovado`, `pix.transferencia.criada`, `credito.solicitacao.criada`, `cartoes.transacao.autorizada`
- Create service: `FraudScoringService` with rule evaluation engine
- Create controller: `FraudController` with endpoints for rule CRUD, transaction evaluation, occurrence management
- Create unit tests + integration test

**Port:** 8125
**Context path:** `/api/fraud`
**Kafka group:** `aurix-fraud-group`

- [ ] **Step 1: Create directory structure**

```bash
mkdir -p backend/aurix-fraud/src/main/java/com/aurix/platform/fraud/{config,controller,entity,repository,service}
mkdir -p backend/aurix-fraud/src/main/resources
mkdir -p backend/aurix-fraud/src/test/java/com/aurix/platform/fraud
```

- [ ] **Step 2: Create scaffold files** (same pattern as Task 1/3, substitute: `fraud`, `8125`, `/api/fraud`, `aurix-fraud-group`)

- [ ] **Step 3: Create entities + repositories** (RegraFraude, ScoreTransacao, OcorrenciaFraude, BloqueioPreventivo)

- [ ] **Step 4: Create KafkaConfig + FraudScoringService + FraudController**

- [ ] **Step 5: Create tests + build**

```bash
mvn clean test -pl aurix-fraud -am
```

- [ ] **Step 6: Commit**

```bash
git add backend/aurix-fraud/
git commit -m "feat(fraud): full module with rule engine, scoring, Kafka integration"
```

---

### Task 5: aurix-notification — Full module

**Files:** (follow same pattern)
- Create `backend/aurix-notification/` with full scaffold
- Create entities: `TemplateNotificacao`, `FilaNotificacao`, `ConfirmacaoRecebimento`, `PreferenciaCliente`
- Create repositories
- Create Kafka topics: `notificacao.enviada`, `notificacao.falhou`
- Create consumers: listens to `cliente.criado`, `kyc.aprovado`, `kyc.rejeitado`, `fraude.transacao.bloqueada`
- Create service: `NotificacaoService` with template rendering and channel dispatch
- Create controller: `NotificacaoController` with endpoints for send, schedule, templates, preferences
- Create tests

**Port:** 8126
**Context path:** `/api/notification`
**Kafka group:** `aurix-notification-group`

- [ ] **Step 1: Create directory structure**

```bash
mkdir -p backend/aurix-notification/src/main/java/com/aurix/platform/notification/{config,controller,entity,repository,service}
mkdir -p backend/aurix-notification/src/main/resources
mkdir -p backend/aurix-notification/src/test/java/com/aurix/platform/notification
```

- [ ] **Step 2: Create scaffold files** (substitute: `notification`, `8126`, `/api/notification`, `aurix-notification-group`)

- [ ] **Step 3: Create entities + repositories**

- [ ] **Step 4: Create KafkaConfig + NotificacaoService + NotificacaoController**

- [ ] **Step 5: Create tests + build**

```bash
mvn clean test -pl aurix-notification -am
```

- [ ] **Step 6: Commit**

```bash
git add backend/aurix-notification/
git commit -m "feat(notification): full module with multi-channel notification and Kafka integration"
```

---

### Task 6: Final infra — Docker images + verify build

**Files:** (none new — build all modules)

- [ ] **Step 1: Build all 4 new modules together**

```bash
mvn clean package -DskipTests -pl aurix-customer,aurix-kyc,aurix-fraud,aurix-notification -am
```
Expected: BUILD SUCCESS, 4 JARs in target/

- [ ] **Step 2: Build Docker images**

```bash
docker compose build aurix-customer aurix-kyc aurix-fraud aurix-notification
```
Expected: 4 images created: `infrastructure-aurix-customer`, `infrastructure-aurix-kyc`, etc.

- [ ] **Step 3: Verify Traefik config**

```bash
python3 -c "import yaml; yaml.safe_load(open('infrastructure/traefik/dynamic.yml')); print('✅ Traefik config valid')"
```

- [ ] **Step 4: Commit final infra**

```bash
git add infrastructure/docker-compose.yml infrastructure/traefik/dynamic.yml aurix-tests/e2e/config.py
git commit -m "feat: add customer, kyc, fraud, notification to docker-compose, traefik, and e2e"
```

- [ ] **Step 5: Push all commits**

```bash
git push origin main
```

---

## Self-Review Checklist

- [ ] Task 0 covers all infra registration (parent POM, compose, Traefik, E2E)
- [ ] Task 1 covers customer module scaffold + entities + repos
- [ ] Task 2 covers customer service + controller + Kafka + tests
- [ ] Task 3 covers KYC module (full)
- [ ] Task 4 covers Fraud module (full)
- [ ] Task 5 covers Notification module (full)
- [ ] Task 6 covers final build + Docker images
- [ ] All ports (8123-8126) are unique and don't conflict
- [ ] All context paths follow `/api/<name>` pattern
- [ ] All Kafka group IDs follow `aurix-<name>-group` pattern
- [ ] Spec endpoints match plan endpoints (POST `/api/clientes`, GET `/api/kyc/solicitacoes/{id}`, etc.)
- [ ] No TODOs, placeholders, or vague steps
