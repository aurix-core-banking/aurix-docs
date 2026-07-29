# Camada 1: Infra + Pipeline — Plano de Implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Criar a infraestrutura alvo (Docker Compose, Traefik, Dockerfiles, CI/CD, Makefile) para 10 domínios consolidados, rodando em paralelo com os 43 módulos legados.

**Architecture:** 10 novos serviços Spring Boot placeholder (só healthcheck) em portas 8200–8209, com roteamento Traefik dedicado, pipelines CI independentes, e Makefile para orquestração da migração. Convive lado a lado com a infra legada — nenhum serviço existente é tocado.

**Tech Stack:** Docker Compose v3.8+, Traefik v3.0, Spring Boot 4.1, GitHub Actions, Maven, Eclipse Temurin 25.

## Global Constraints

- Todos os 10 serviços expõem `/actuator/health` como healthcheck
- Healthcheck management port = server.port + 1000 (9200–9209)
- Portas 8200–8209 usadas exclusivamente pelos novos serviços
- Docker Compose v2 usa rede `aurix-network` existente
- Nenhum módulo legado é modificado nesta fase
- CI pipelines usam working-directory dentro de `backend/svc-*`
- Docker image tag: `aurix/svc-{domain}:latest`

---

## File Structure

```
backend/
  svc-banking/                  → placeholder Spring Boot (porta 8200)
    pom.xml
    Dockerfile
    src/main/java/com/aurix/platform/banking/
      AurixBankingApplication.java
    src/main/resources/
      application.yml
  svc-payments/                 → placeholder Spring Boot (porta 8201)
    ... (mesma estrutura, porta 8201)
  svc-credit/                   → placeholder (8202)
  svc-products/                 → placeholder (8203)
  svc-customer/                 → placeholder (8204)
  svc-fraud/                    → placeholder (8205)
  svc-compliance/               → placeholder (8206)
  svc-finance-mgmt/             → placeholder (8207)
  svc-platform/                 → placeholder (8208)
  svc-intelligence/             → placeholder (8209)

infrastructure/
  docker-compose.v2.yml         → 10 serviços placeholder
  traefik/dynamic.v2.yml        → 10 routers + 10 services Traefik

.github/workflows/
  ci-banking.yml                → CI pipeline svc-banking
  ci-payments.yml               → CI pipeline svc-payments
  ci-credit.yml                 → CI pipeline svc-credit
  ci-products.yml               → CI pipeline svc-products
  ci-customer.yml               → CI pipeline svc-customer
  ci-fraud.yml                  → CI pipeline svc-fraud
  ci-compliance.yml             → CI pipeline svc-compliance
  ci-finance-mgmt.yml           → CI pipeline svc-finance-mgmt
  ci-platform.yml               → CI pipeline svc-platform
  ci-intelligence.yml           → CI pipeline svc-intelligence

Makefile                        → targets adicionados (infra-up, domain-up, v2-up, legacy-down, check-ports)
```

---

### Task 1: Scaffold — criar 10 serviços Spring Boot placeholder

**Files:**
- Create: `backend/svc-banking/pom.xml`
- Create: `backend/svc-banking/src/main/java/com/aurix/platform/banking/AurixBankingApplication.java`
- Create: `backend/svc-banking/Dockerfile`
- Create (identical pattern para svc-payments, svc-credit, svc-products, svc-customer, svc-fraud, svc-compliance, svc-finance-mgmt, svc-platform, svc-intelligence)

**Interfaces:**
- Consumes: estrutura Maven existente (`backend/pom.xml` parent)
- Produces: 10 módulos Maven compiláveis com `mvn clean compile -pl svc-banking`

- [ ] **Step 1: Criar diretórios dos 10 serviços**

```bash
mkdir -p backend/svc-banking/src/main/java/com/aurix/platform/banking
mkdir -p backend/svc-banking/src/main/resources
mkdir -p backend/svc-payments/src/main/java/com/aurix/platform/payments
mkdir -p backend/svc-payments/src/main/resources
mkdir -p backend/svc-credit/src/main/java/com/aurix/platform/credit
mkdir -p backend/svc-credit/src/main/resources
mkdir -p backend/svc-products/src/main/java/com/aurix/platform/products
mkdir -p backend/svc-products/src/main/resources
mkdir -p backend/svc-customer/src/main/java/com/aurix/platform/customer
mkdir -p backend/svc-customer/src/main/resources
mkdir -p backend/svc-fraud/src/main/java/com/aurix/platform/fraud
mkdir -p backend/svc-fraud/src/main/resources
mkdir -p backend/svc-compliance/src/main/java/com/aurix/platform/compliance
mkdir -p backend/svc-compliance/src/main/resources
mkdir -p backend/svc-finance-mgmt/src/main/java/com/aurix/platform/financemgmt
mkdir -p backend/svc-finance-mgmt/src/main/resources
mkdir -p backend/svc-platform/src/main/java/com/aurix/platform/platform
mkdir -p backend/svc-platform/src/main/resources
mkdir -p backend/svc-intelligence/src/main/java/com/aurix/platform/intelligence
mkdir -p backend/svc-intelligence/src/main/resources
```

- [ ] **Step 2: Criar pom.xml do svc-banking**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.aurix.platform</groupId>
        <artifactId>aurix-backend</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <relativePath>../pom.xml</relativePath>
    </parent>

    <artifactId>svc-banking</artifactId>
    <packaging>jar</packaging>
    <name>AURIX — Banking Core</name>
    <description>Banking Core: contas, transações, saldo, boleto, poupança, salário, settlement, pricing</description>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
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

- [ ] **Step 3: Criar AurixBankingApplication.java**

```java
package com.aurix.platform.banking;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class AurixBankingApplication {

    public static void main(String[] args) {
        SpringApplication.run(AurixBankingApplication.class, args);
    }
}
```

- [ ] **Step 4: Criar application.yml para svc-banking**

```yaml
server:
  port: 8200

spring:
  application:
    name: svc-banking
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
      show-components: always
  server:
    port: 9200
  health:
    db:
      enabled: false
    redis:
      enabled: false
    kafka:
      enabled: false
```

- [ ] **Step 5: Criar Dockerfile para svc-banking**

```dockerfile
FROM eclipse-temurin:25-jdk-jammy AS builder
WORKDIR /build
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw mvnw
RUN chmod +x mvnw && ./mvnw dependency:go-offline -q
COPY src src
RUN ./mvnw package -DskipTests -q

FROM eclipse-temurin:25-jre-jammy
WORKDIR /app
RUN groupadd -r aurix && useradd -r -g aurix aurix
USER aurix
COPY --from=builder /build/target/*.jar app.jar
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:+ExitOnOutOfMemoryError -Djava.security.egd=file:/dev/./urandom"
EXPOSE 8200
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

- [ ] **Step 6: Repetir para os 9 domínios restantes**

Criar os mesmos 4 arquivos (pom.xml, Application.java, application.yml, Dockerfile) para cada domínio, alterando:

| Domínio | Diretório | Pacote | Application class | server.port | management.port |
|---------|-----------|--------|-------------------|:-----------:|:---------------:|
| svc-payments | `backend/svc-payments` | `com.aurix.platform.payments` | `AurixPaymentsApplication` | 8201 | 9201 |
| svc-credit | `backend/svc-credit` | `com.aurix.platform.credit` | `AurixCreditApplication` | 8202 | 9202 |
| svc-products | `backend/svc-products` | `com.aurix.platform.products` | `AurixProductsApplication` | 8203 | 9203 |
| svc-customer | `backend/svc-customer` | `com.aurix.platform.customer` | `AurixCustomerApplication` | 8204 | 9204 |
| svc-fraud | `backend/svc-fraud` | `com.aurix.platform.fraud` | `AurixFraudApplication` | 8205 | 9205 |
| svc-compliance | `backend/svc-compliance` | `com.aurix.platform.compliance` | `AurixComplianceApplication` | 8206 | 9206 |
| svc-finance-mgmt | `backend/svc-finance-mgmt` | `com.aurix.platform.financemgmt` | `AurixFinanceMgmtApplication` | 8207 | 9207 |
| svc-platform | `backend/svc-platform` | `com.aurix.platform.platform` | `AurixPlatformApplication` | 8208 | 9208 |
| svc-intelligence | `backend/svc-intelligence` | `com.aurix.platform.intelligence` | `AurixIntelligenceApplication` | 8209 | 9209 |

O conteúdo de cada arquivo é idêntico ao svc-banking, apenas alterando:
- `artifactId` no pom.xml
- `name` e `description` no pom.xml
- Nome da classe Application
- `server.port` e `management.server.port` no application.yml
- `EXPOSE` no Dockerfile

**Teste:** Para cada domínio, executar:

```bash
cd backend
./mvnw clean compile -pl svc-banking -q
```

Expected output: `BUILD SUCCESS` (sem erros). O mesmo comando funciona para `svc-payments`, etc.

- [ ] **Step 7: Commit**

```bash
git add backend/svc-banking/ backend/svc-payments/ backend/svc-credit/ backend/svc-products/ backend/svc-customer/ backend/svc-fraud/ backend/svc-compliance/ backend/svc-finance-mgmt/ backend/svc-platform/ backend/svc-intelligence/
git commit -m "feat: scaffold 10 placeholder Spring Boot services (F0)"
```

---

### Task 2: Docker Compose v2 — 10 serviços placeholder

**Files:**
- Create: `infrastructure/docker-compose.v2.yml`

**Interfaces:**
- Consumes: 10 diretórios `backend/svc-*` com Dockerfile (Task 1)
- Produces: arquivo Compose validável com `docker compose config`

- [ ] **Step 1: Escrever docker-compose.v2.yml**

```yaml
# infrastructure/docker-compose.v2.yml
# Docker Compose alvo: 10 domínios consolidados
# Usado EM PARALELO com docker-compose.yml (legado) durante migração
# Para subir APENAS os domínios migrados:
#   docker compose -f docker-compose.yml -f docker-compose.v2.yml up <service>

services:

  svc-banking:
    build:
      context: ../backend/svc-banking
      dockerfile: Dockerfile
    container_name: svc-banking
    ports:
      - "8200:8200"
    environment:
      SERVER_PORT: 8200
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aurix_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aurix_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aurix_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
      SPRING_REDIS_PORT: 6379
      SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI: http://keycloak:8080/realms/aurix
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
      redis: { condition: service_started }
    networks:
      - aurix-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8200/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  svc-payments:
    build:
      context: ../backend/svc-payments
      dockerfile: Dockerfile
    container_name: svc-payments
    ports:
      - "8201:8201"
    environment:
      SERVER_PORT: 8201
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aurix_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aurix_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aurix_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
    networks:
      - aurix-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8201/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  svc-credit:
    build:
      context: ../backend/svc-credit
      dockerfile: Dockerfile
    container_name: svc-credit
    ports:
      - "8202:8202"
    environment:
      SERVER_PORT: 8202
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aurix_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aurix_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aurix_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
    networks:
      - aurix-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8202/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  svc-products:
    build:
      context: ../backend/svc-products
      dockerfile: Dockerfile
    container_name: svc-products
    ports:
      - "8203:8203"
    environment:
      SERVER_PORT: 8203
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aurix_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aurix_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aurix_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
    networks:
      - aurix-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8203/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  svc-customer:
    build:
      context: ../backend/svc-customer
      dockerfile: Dockerfile
    container_name: svc-customer
    ports:
      - "8204:8204"
    environment:
      SERVER_PORT: 8204
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aurix_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aurix_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aurix_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
      SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI: http://keycloak:8080/realms/aurix
      SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_JWK_SET_URI: http://keycloak:8080/realms/aurix/protocol/openid-connect/certs
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
      keycloak: { condition: service_healthy }
    networks:
      - aurix-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8204/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  svc-fraud:
    build:
      context: ../backend/svc-fraud
      dockerfile: Dockerfile
    container_name: svc-fraud
    ports:
      - "8205:8205"
    environment:
      SERVER_PORT: 8205
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aurix_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aurix_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aurix_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
      redis: { condition: service_started }
    networks:
      - aurix-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8205/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  svc-compliance:
    build:
      context: ../backend/svc-compliance
      dockerfile: Dockerfile
    container_name: svc-compliance
    ports:
      - "8206:8206"
    environment:
      SERVER_PORT: 8206
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aurix_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aurix_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aurix_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
    networks:
      - aurix-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8206/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  svc-finance-mgmt:
    build:
      context: ../backend/svc-finance-mgmt
      dockerfile: Dockerfile
    container_name: svc-finance-mgmt
    ports:
      - "8207:8207"
    environment:
      SERVER_PORT: 8207
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aurix_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aurix_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aurix_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
    networks:
      - aurix-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8207/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  svc-platform:
    build:
      context: ../backend/svc-platform
      dockerfile: Dockerfile
    container_name: svc-platform
    ports:
      - "8208:8208"
    environment:
      SERVER_PORT: 8208
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aurix_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aurix_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aurix_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
    networks:
      - aurix-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8208/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  svc-intelligence:
    build:
      context: ../backend/svc-intelligence
      dockerfile: Dockerfile
    container_name: svc-intelligence
    ports:
      - "8209:8209"
    environment:
      SERVER_PORT: 8209
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aurix_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aurix_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aurix_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
    networks:
      - aurix-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8209/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

networks:
  aurix-network:
    external: true
```

- [ ] **Step 2: Validar arquivo Compose**

```bash
docker compose -f infrastructure/docker-compose.v2.yml config
```

Expected output: saída YAML completa, sem erros.

- [ ] **Step 3: Commit**

```bash
git add infrastructure/docker-compose.v2.yml
git commit -m "feat(docker-compose): add v2 with 10 placeholder services (F0)"
```

---

### Task 3: Traefik v2 — 10 routers + services

**Files:**
- Create: `infrastructure/traefik/dynamic.v2.yml`

**Interfaces:**
- Consumes: docker-compose.v2.yml com svc-*:8200-8209 (Task 2)
- Produces: configuração Traefik validável com `traefik healthcheck`

- [ ] **Step 1: Escrever dynamic.v2.yml**

```yaml
# infrastructure/traefik/dynamic.v2.yml
# Configuração ALVO: 10 domínios consolidados
# Ativado em paralelo com dynamic.yml durante migração via feature flags
# Traefik suporta múltiplos arquivos em providers.file.directory

http:
  middlewares:
    ratelimit-default:
      rateLimit:
        average: 200
        burst: 400

    ratelimit-payments:
      rateLimit:
        average: 500
        burst: 1000

    ratelimit-fraud:
      rateLimit:
        average: 1000
        burst: 2000

    ratelimit-compliance:
      rateLimit:
        average: 50
        burst: 100

    cors-headers:
      headers:
        accessControlAllowOriginList:
          - "http://localhost:3000"
          - "http://localhost:3001"
        accessControlAllowMethods:
          - GET
          - POST
          - PUT
          - PATCH
          - DELETE
          - OPTIONS
        accessControlAllowHeaders:
          - "Content-Type"
          - "Authorization"
          - "X-Tenant-Id"
          - "X-Migration-Domain"

  routers:
    banking-core:
      rule: "PathPrefix(`/api/core`) || PathPrefix(`/api/contas`) || PathPrefix(`/api/transacoes`) || PathPrefix(`/api/settlement`) || PathPrefix(`/api/pricing`)"
      service: svc-banking
      middlewares:
        - ratelimit-default
        - cors-headers
      entrypoints:
        - web

    banking-poupanca:
      rule: "PathPrefix(`/api/poupanca`)"
      service: svc-banking
      middlewares:
        - ratelimit-default
      entrypoints:
        - web

    banking-salario:
      rule: "PathPrefix(`/api/salario`)"
      service: svc-banking
      middlewares:
        - ratelimit-default
      entrypoints:
        - web

    payments:
      rule: "PathPrefix(`/api/pix`) || PathPrefix(`/api/boleto`) || PathPrefix(`/api/agendamento`)"
      service: svc-payments
      middlewares:
        - ratelimit-payments
        - cors-headers
      entrypoints:
        - web

    credit:
      rule: "PathPrefix(`/api/credit`) || PathPrefix(`/api/consignado`) || PathPrefix(`/api/financiamento`)"
      service: svc-credit
      middlewares:
        - ratelimit-default
        - cors-headers
      entrypoints:
        - web

    products:
      rule: "PathPrefix(`/api/investimento`) || PathPrefix(`/api/cambio`) || PathPrefix(`/api/seguros`) || PathPrefix(`/api/cartoes`) || PathPrefix(`/api/treasury`)"
      service: svc-products
      middlewares:
        - ratelimit-default
        - cors-headers
      entrypoints:
        - web

    customer:
      rule: "PathPrefix(`/api/customer`) || PathPrefix(`/api/kyc`) || PathPrefix(`/api/onboarding`) || PathPrefix(`/api/auth`) || PathPrefix(`/api/mfa`) || PathPrefix(`/api/security`)"
      service: svc-customer
      middlewares:
        - ratelimit-default
        - cors-headers
      entrypoints:
        - web

    fraud:
      rule: "PathPrefix(`/api/fraud`) || PathPrefix(`/api/risk`)"
      service: svc-fraud
      middlewares:
        - ratelimit-fraud
      entrypoints:
        - web

    compliance:
      rule: "PathPrefix(`/api/compliance`) || PathPrefix(`/api/audit`) || PathPrefix(`/api/bacen`) || PathPrefix(`/api/tax`) || PathPrefix(`/api/accounting`)"
      service: svc-compliance
      middlewares:
        - ratelimit-compliance
      entrypoints:
        - web

    finance-mgmt:
      rule: "PathPrefix(`/api/controller`) || PathPrefix(`/api/budget`) || PathPrefix(`/api/cost`) || PathPrefix(`/api/financial`)"
      service: svc-finance-mgmt
      middlewares:
        - ratelimit-default
      entrypoints:
        - web

    platform:
      rule: "PathPrefix(`/api/billing`) || PathPrefix(`/api/provisioning`) || PathPrefix(`/api/webhooks`) || PathPrefix(`/api/notification`) || PathPrefix(`/api/catalog`) || PathPrefix(`/api/baas`) || PathPrefix(`/aurix-organization`)"
      service: svc-platform
      middlewares:
        - ratelimit-default
        - cors-headers
      entrypoints:
        - web

    intelligence:
      rule: "PathPrefix(`/api/ai`) || PathPrefix(`/api/analytics`) || PathPrefix(`/api/openfinance`) || PathPrefix(`/api/internet-banking`) || PathPrefix(`/api/mobile-banking`)"
      service: svc-intelligence
      middlewares:
        - ratelimit-default
        - cors-headers
      entrypoints:
        - web

  services:
    svc-banking:
      loadBalancer:
        servers:
          - url: "http://svc-banking:8200"
        healthCheck:
          path: /actuator/health
          interval: 30s
          timeout: 5s

    svc-payments:
      loadBalancer:
        servers:
          - url: "http://svc-payments:8201"
        healthCheck:
          path: /actuator/health
          interval: 30s
          timeout: 5s

    svc-credit:
      loadBalancer:
        servers:
          - url: "http://svc-credit:8202"
        healthCheck:
          path: /actuator/health
          interval: 30s

    svc-products:
      loadBalancer:
        servers:
          - url: "http://svc-products:8203"
        healthCheck:
          path: /actuator/health
          interval: 30s

    svc-customer:
      loadBalancer:
        servers:
          - url: "http://svc-customer:8204"
        healthCheck:
          path: /actuator/health
          interval: 30s

    svc-fraud:
      loadBalancer:
        servers:
          - url: "http://svc-fraud:8205"
        healthCheck:
          path: /actuator/health
          interval: 10s

    svc-compliance:
      loadBalancer:
        servers:
          - url: "http://svc-compliance:8206"
        healthCheck:
          path: /actuator/health
          interval: 30s

    svc-finance-mgmt:
      loadBalancer:
        servers:
          - url: "http://svc-finance-mgmt:8207"
        healthCheck:
          path: /actuator/health
          interval: 30s

    svc-platform:
      loadBalancer:
        servers:
          - url: "http://svc-platform:8208"
        healthCheck:
          path: /actuator/health
          interval: 30s

    svc-intelligence:
      loadBalancer:
        servers:
          - url: "http://svc-intelligence:8209"
        healthCheck:
          path: /actuator/health
          interval: 30s
```

- [ ] **Step 2: Validar sintaxe YAML**

```bash
python3 -c "import yaml; yaml.safe_load(open('infrastructure/traefik/dynamic.v2.yml')); print('✓ YAML válido')"
```

Expected output: `✓ YAML válido`

- [ ] **Step 3: Commit**

```bash
git add infrastructure/traefik/dynamic.v2.yml
git commit -m "feat(traefik): add v2 config with 10 routers and services (F0)"
```

---

### Task 4: Makefile — targets de migração

**Files:**
- Modify: `Makefile` (adicionar targets ao final)

**Interfaces:**
- Consumes: docker-compose.v2.yml, dynamic.v2.yml
- Produces: comandos `make infra-up`, `make domain-up`, `make v2-up`, `make legacy-down`, `make check-ports`

- [ ] **Step 1: Ler Makefile existente**

```bash
cat Makefile
```

Verificar estrutura existente para manter estilo.

- [ ] **Step 2: Adicionar targets de migração ao final do Makefile**

Adicionar ao Makefile existente:
```makefile
# ─────────────────────────────────────────────
# Migration targets — 10-domain consolidation
# ─────────────────────────────────────────────

# Subir apenas infraestrutura (sem nenhum serviço de aplicação)
infra-up:
	docker compose -f infrastructure/docker-compose.yml up -d \
	  traefik postgres redis kafka kafka-ui keycloak

# Subir legado completo (43 módulos atuais)
legacy-up:
	docker compose -f infrastructure/docker-compose.yml up -d

# Subir um domínio específico em paralelo ao legado
# Uso: make domain-up DOMAIN=svc-customer
domain-up:
	docker compose \
	  -f infrastructure/docker-compose.yml \
	  -f infrastructure/docker-compose.v2.yml \
	  up -d $(DOMAIN)

# Subir todos os 10 novos domínios (sem legado)
v2-up:
	docker compose -f infrastructure/docker-compose.v2.yml up -d

# Derrubar serviços legados por domínio conforme migração avança
# Uso: make legacy-down SERVICES="aurix-customer aurix-kyc"
legacy-down:
	docker compose -f infrastructure/docker-compose.yml stop $(SERVICES)

# Build completo dos 10 domínios
build-all:
	cd backend && ./mvnw clean install -DskipTests

# Testes de todos os domínios
test-all:
	cd backend && ./mvnw test

# Verificar portas 8200-8209 (detectar conflitos)
check-ports:
	@for port in 8200 8201 8202 8203 8204 8205 8206 8207 8208 8209; do \
	  lsof -i :$$port > /dev/null 2>&1 && echo "OCUPADA: $$port" || echo "livre: $$port"; \
	done

.PHONY: infra-up legacy-up domain-up v2-up legacy-down build-all test-all check-ports
```

- [ ] **Step 3: Verificar sintaxe do Makefile**

```bash
make -n check-ports
```

Expected output: linhas com "livre: 8200" a "livre: 8209" (se portas livres) ou "OCUPADA: 820X" (se ocupadas).

- [ ] **Step 4: Commit**

```bash
git add Makefile
git commit -m "feat(makefile): add migration targets (F0)"
```

---

### Task 5: CI/CD — 10 pipelines GitHub Actions

**Files:**
- Create: `.github/workflows/ci-banking.yml`
- Create: `.github/workflows/ci-payments.yml`
- Create: `.github/workflows/ci-credit.yml`
- Create: `.github/workflows/ci-products.yml`
- Create: `.github/workflows/ci-customer.yml`
- Create: `.github/workflows/ci-fraud.yml`
- Create: `.github/workflows/ci-compliance.yml`
- Create: `.github/workflows/ci-finance-mgmt.yml`
- Create: `.github/workflows/ci-platform.yml`
- Create: `.github/workflows/ci-intelligence.yml`

- [ ] **Step 1: Verificar estrutura de CI existente**

```bash
ls .github/workflows/
```

Verificar naming convention e estrutura de outros workflows no repositório.

- [ ] **Step 2: Criar ci-banking.yml**

```yaml
name: CI — Banking Core

on:
  push:
    paths:
      - 'backend/svc-banking/**'
      - '.github/workflows/ci-banking.yml'
  pull_request:
    paths:
      - 'backend/svc-banking/**'

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend/svc-banking

    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: aurix_test
          POSTGRES_USER: aurix
          POSTGRES_PASSWORD: aurix
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '25'
          distribution: 'temurin'
          cache: 'maven'

      - name: Build
        run: ./mvnw clean compile -q

      - name: Unit Tests
        run: ./mvnw test -q

      - name: Integration Tests
        run: ./mvnw verify -Pintegration-tests -q
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/aurix_test
          SPRING_DATASOURCE_USERNAME: aurix
          SPRING_DATASOURCE_PASSWORD: aurix

      - name: Static Analysis
        run: ./mvnw pmd:check spotbugs:check checkstyle:check -q

  docker-build:
    needs: build-and-test
    if: github.ref == 'refs/heads/develop' || github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: |
          docker build -t aurix/svc-banking:${{ github.sha }} \
            backend/svc-banking/

      - name: Push to registry
        run: |
          echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login -u "${{ secrets.REGISTRY_USER }}" --password-stdin
          docker push aurix/svc-banking:${{ github.sha }}
          docker tag aurix/svc-banking:${{ github.sha }} aurix/svc-banking:latest
          docker push aurix/svc-banking:latest
```

- [ ] **Step 3: Criar os 9 pipelines restantes (ci-payments.yml a ci-intelligence.yml)**

Para cada um, copiar ci-banking.yml alterando:
- `name:` → `CI — Payments`, `CI — Credit`, etc.
- `paths:` → `backend/svc-payments/**`, `backend/svc-credit/**`, etc.
- `defaults.run.working-directory:` → `backend/svc-payments`, etc.
- `docker build -t aurix/svc-payments:${{ github.sha }}` etc.
- `docker push aurix/svc-payments:${{ github.sha }}` etc.
- `docker tag` e `docker push` para o nome correto

Criar arquivos: `ci-payments.yml`, `ci-credit.yml`, `ci-products.yml`, `ci-customer.yml`, `ci-fraud.yml`, `ci-compliance.yml`, `ci-finance-mgmt.yml`, `ci-platform.yml`, `ci-intelligence.yml`.

- [ ] **Step 4: Validar YAML dos workflows**

```bash
python3 -c "
import yaml, glob
for f in glob.glob('.github/workflows/ci-*.yml'):
    with open(f) as fh:
        yaml.safe_load(fh)
    print(f'✓ {f}')
"
```

Expected output: ✓ para todos os 10 arquivos.

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/ci-*.yml
git commit -m "feat(ci): add 10 domain-specific pipeline workflows (F0)"
```

---

### Task 6: Maven parent POM — adicionar novos módulos

**Files:**
- Modify: `backend/pom.xml`

- [ ] **Step 1: Ler o pom.xml raiz existente**

```bash
head -60 backend/pom.xml
```

Localizar a seção `<modules>` para saber a posição exata de inserção.

- [ ] **Step 2: Adicionar módulos placeholder ao pom.xml**

Localizar o bloco `<modules>` e adicionar após o último módulo existente:

```xml
        <!-- ════════════════════════════════════════════ -->
        <!-- 10 DOMÍNIOS CONSOLIDADOS (migração Fase 0)  -->
        <!-- Descomentar conforme cada fase de migração   -->
        <!-- ════════════════════════════════════════════ -->
        <!--
        <module>svc-banking</module>
        <module>svc-payments</module>
        <module>svc-credit</module>
        <module>svc-products</module>
        <module>svc-customer</module>
        <module>svc-fraud</module>
        <module>svc-compliance</module>
        <module>svc-finance-mgmt</module>
        <module>svc-platform</module>
        <module>svc-intelligence</module>
        -->
```

Os módulos ficam comentados — o placeholder `svc-*` modules estão compiláveis independentemente com `-pl`, mas não no build completo até serem descomentados.

- [ ] **Step 3: Validar que o build existente continua funcional**

```bash
cd backend && ./mvnw clean compile -pl aurix-shared -q
```

Expected output: `BUILD SUCCESS`. Se houver erro, reverter e ajustar.

- [ ] **Step 4: Commit**

```bash
git add backend/pom.xml
git commit -m "chore(pom): add commented module slots for 10 new domains (F0)"
```

---

### Task 7: Validação final — F0 completo

**Files:** Nenhum — script de validação único

- [ ] **Step 1: Verificar checklist de conclusão do spec**

```bash
echo "=== F0 Completion Checklist ==="
echo ""

echo "1. docker-compose.v2.yml criado?"
ls -la infrastructure/docker-compose.v2.yml && echo "  ✓" || echo "  ✗"

echo "2. dynamic.v2.yml criado?"
ls -la infrastructure/traefik/dynamic.v2.yml && echo "  ✓" || echo "  ✗"

echo "3. Dockerfiles existem?"
for d in svc-banking svc-payments svc-credit svc-products svc-customer svc-fraud svc-compliance svc-finance-mgmt svc-platform svc-intelligence; do
  ls backend/$d/Dockerfile > /dev/null 2>&1 && echo "  ✓ $d" || echo "  ✗ $d"
done

echo "4. Pipelines CI existem?"
for d in banking payments credit products customer fraud compliance finance-mgmt platform intelligence; do
  ls .github/workflows/ci-$d.yml > /dev/null 2>&1 && echo "  ✓ ci-$d.yml" || echo "  ✗ ci-$d.yml"
done

echo "5. Makefile com targets de migração?"
grep -q "infra-up:" Makefile && echo "  ✓ infra-up" || echo "  ✗ infra-up"
grep -q "domain-up:" Makefile && echo "  ✓ domain-up" || echo "  ✗ domain-up"
grep -q "v2-up:" Makefile && echo "  ✓ v2-up" || echo "  ✗ v2-up"
grep -q "check-ports:" Makefile && echo "  ✓ check-ports" || echo "  ✗ check-ports"

echo "6. pom.xml com slots comentados?"
grep -q "svc-banking" backend/pom.xml && echo "  ✓" || echo "  ✗"

echo "7. Portas 8200-8209 livres?"
for port in 8200 8201 8202 8203 8204 8205 8206 8207 8208 8209; do
  lsof -i :$port > /dev/null 2>&1 && echo "  OCUPADA: $port" || echo "  livre: $port"
done

echo ""
echo "=== Validação de build ==="
cd backend && ./mvnw clean compile -pl svc-banking -q && echo "✓ svc-banking compila" || echo "✗ svc-banking falhou"

echo ""
echo "=== F0 Complete ==="
```
