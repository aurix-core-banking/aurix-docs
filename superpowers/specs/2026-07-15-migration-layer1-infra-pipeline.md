# Migração para 10 Domínios — Camada 1: Infra + Pipeline

> **Escopo:** Docker Compose alvo (10 serviços), roteamento Traefik, CI/CD por domínio, estratégia de migração incremental com feature flags. Não inclui movimento de código — isso é Camada 2.
> **Pré-requisito:** Conflito de porta 8090 corrigido (✅ feito).
> **Não cobre:** Mapa de entidades, merge de módulos Java, schema de banco, tópicos Kafka — essas são Camadas 2 e 3.

---

## Estado Atual vs. Alvo

| | Atual | Alvo |
|---|---|---|
| Serviços backend | 43 Spring Boot | 10 Spring Boot |
| JVMs em dev | 43 processos | 10 processos |
| RAM mínima dev | ~18 GB | ~5 GB |
| Entrypoints Traefik | 38 routers | 10 routers |
| Rotas Traefik | 38 services | 10 services |
| Portas expostas | 8082–8129+ | 8200–8209 (novos) |
| Pipelines CI | 1 pipeline multi-módulo | 10 pipelines independentes |

---

## Tabela de Portas — 10 Domínios

| Domínio | Serviço | Porta interna | Porta host (dev) | Context paths |
|---------|---------|:---:|:---:|---|
| Banking Core | `svc-banking` | 8200 | 8200 | `/api/core`, `/api/contas`, `/api/transacoes`, `/api/poupanca`, `/api/salario`, `/api/settlement`, `/api/pricing` |
| Payments | `svc-payments` | 8201 | 8201 | `/api/pix`, `/api/boleto`, `/api/agendamento` |
| Credit | `svc-credit` | 8202 | 8202 | `/api/credit`, `/api/consignado`, `/api/financiamento` |
| Products | `svc-products` | 8203 | 8203 | `/api/investimento`, `/api/cambio`, `/api/seguros`, `/api/cartoes`, `/api/treasury` |
| Customer Identity | `svc-customer` | 8204 | 8204 | `/api/customer`, `/api/kyc`, `/api/onboarding`, `/api/auth`, `/api/mfa` |
| Fraud & Risk | `svc-fraud` | 8205 | 8205 | `/api/fraud`, `/api/risk` |
| Compliance | `svc-compliance` | 8206 | 8206 | `/api/compliance`, `/api/audit`, `/api/bacen`, `/api/tax`, `/api/accounting` |
| Finance Mgmt | `svc-finance-mgmt` | 8207 | 8207 | `/api/controller`, `/api/budget`, `/api/cost`, `/api/financial` |
| Platform | `svc-platform` | 8208 | 8208 | `/api/billing`, `/api/provisioning`, `/api/webhooks`, `/api/notification`, `/api/catalog`, `/api/baas`, `/api/organization` |
| Intelligence | `svc-intelligence` | 8209 | 8209 | `/api/ai`, `/api/analytics`, `/api/openfinance`, `/api/internet-banking`, `/api/mobile-banking` |

> **Nota sobre context paths:** durante a migração, os path prefixes antigos (`/api/pix`, `/api/kyc` etc.) são mantidos no Traefik como aliases — o código cliente não muda.

---

## Estratégia de Migração: Parallel Run com Feature Flags

A migração é **incremental e reversível**. Nunca derruba os 43 serviços de uma vez.

### Princípio: Strangler Fig Pattern

```
Fase 0 (hoje)           Fase 1 (novo+legado)       Fase 2 (alvo)
                                                    
Cliente → Traefik       Cliente → Traefik           Cliente → Traefik
              ↓                       ↓                          ↓
         [43 serviços]    ┌─[feature flag]─┐          [10 domínios]
                          │               │
                    [novo domínio]  [legado] ← desligado por domínio
```

### Feature Flag via Header Traefik

O Traefik suporta roteamento condicional por header. Exemplo para `svc-customer`:

```yaml
# infrastructure/traefik/dynamic.yml
routers:
  # Rota NEW — ativada por header X-Migration-Domain: customer
  customer-new:
    rule: "PathPrefix(`/api/customer`) && Headers(`X-Migration-Domain`, `customer`)"
    service: svc-customer        # novo serviço consolidado
    priority: 200                # maior prioridade vence

  # Rota LEGACY — fallback quando header ausente
  customer-legacy:
    rule: "PathPrefix(`/api/customer`)"
    service: customer            # aureus-customer atual
    priority: 100
```

Ativação incremental por domínio, sem downtime. Rollback = remover o header ou mudar priority.

### Fases de Migração por Domínio

| Fase | Domínio | Duração estimada | Pré-requisito |
|------|---------|:---:|---|
| **F0** | Infra base (este spec) | 1 semana | — |
| **F1** | `svc-customer` | 2 semanas | F0 |
| **F2** | `svc-fraud` | 1 semana | F1 (depende de customer) |
| **F3** | `svc-payments` | 2 semanas | F1, F2 |
| **F4** | `svc-credit` | 2 semanas | F1, F3 |
| **F5** | `svc-products` | 3 semanas | F1, F3, F4 |
| **F6** | `svc-compliance` | 2 semanas | F1, F2, F3 |
| **F7** | `svc-finance-mgmt` | 1 semana | F0 (independente) |
| **F8** | `svc-platform` | 2 semanas | F1, F6 |
| **F9** | `svc-intelligence` | 2 semanas | F1, F3, F6 |
| **F10** | `svc-banking` (core) | 3 semanas | Todos anteriores |

> Banking Core é o último a migrar — é a fundação. Migra quando todos os seus consumidores já estiverem nos novos domínios.

---

## Docker Compose Alvo (`docker-compose.v2.yml`)

Arquivo paralelo ao atual — convive com `docker-compose.yml` (legado) durante a migração.

```yaml
# infrastructure/docker-compose.v2.yml
# Docker Compose alvo: 10 domínios + infraestrutura compartilhada
# Usado JUNTO com docker-compose.yml (legado) durante migração
# Para subir APENAS os domínios migrados:
#   docker compose -f docker-compose.yml -f docker-compose.v2.yml up

services:

  # ─────────────────────────────────────────────
  # SVC-BANKING — Banking Core
  # ─────────────────────────────────────────────
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
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aureus_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aureus_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aureus_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
      SPRING_REDIS_PORT: 6379
      SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI: http://keycloak:8080/realms/aureus
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
      redis: { condition: service_started }
    networks:
      - aureus-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8200/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # ─────────────────────────────────────────────
  # SVC-PAYMENTS — Pagamentos (PIX, TED, Boleto)
  # ─────────────────────────────────────────────
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
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aureus_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aureus_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aureus_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
      SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI: http://keycloak:8080/realms/aureus
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
    networks:
      - aureus-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8201/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # ─────────────────────────────────────────────
  # SVC-CREDIT — Crédito, Consignado, Financiamento
  # ─────────────────────────────────────────────
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
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aureus_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aureus_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aureus_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
      SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI: http://keycloak:8080/realms/aureus
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
    networks:
      - aureus-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8202/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # ─────────────────────────────────────────────
  # SVC-PRODUCTS — Investimento, Câmbio, Seguros, Cartões
  # ─────────────────────────────────────────────
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
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aureus_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aureus_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aureus_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
      SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI: http://keycloak:8080/realms/aureus
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
    networks:
      - aureus-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8203/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # ─────────────────────────────────────────────
  # SVC-CUSTOMER — Customer, KYC, Onboarding, Auth/MFA
  # ─────────────────────────────────────────────
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
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aureus_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aureus_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aureus_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
      SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI: http://keycloak:8080/realms/aureus
      SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_JWK_SET_URI: http://keycloak:8080/realms/aureus/protocol/openid-connect/certs
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
      keycloak: { condition: service_healthy }
    networks:
      - aureus-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8204/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # ─────────────────────────────────────────────
  # SVC-FRAUD — Fraude e Risco em Tempo Real
  # ─────────────────────────────────────────────
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
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aureus_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aureus_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aureus_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
      redis: { condition: service_started }
    networks:
      - aureus-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8205/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # ─────────────────────────────────────────────
  # SVC-COMPLIANCE — COAF, Basileia, LGPD, Audit, BACEN, Fiscal
  # ─────────────────────────────────────────────
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
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aureus_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aureus_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aureus_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
    networks:
      - aureus-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8206/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # ─────────────────────────────────────────────
  # SVC-FINANCE-MGMT — Controladoria, Orçamento, Custo, Financeiro
  # ─────────────────────────────────────────────
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
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aureus_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aureus_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aureus_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
    networks:
      - aureus-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8207/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # ─────────────────────────────────────────────
  # SVC-PLATFORM — Billing, Provisioning, Webhooks, Notification, Catalog, BaaS
  # ─────────────────────────────────────────────
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
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aureus_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aureus_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aureus_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
    networks:
      - aureus-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8208/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # ─────────────────────────────────────────────
  # SVC-INTELLIGENCE — AI, Analytics, Open Finance, Canais Digitais
  # ─────────────────────────────────────────────
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
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aureus_db
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME:-aureus_user}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD:-aureus_dev_password}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
    networks:
      - aureus-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8209/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s
```

---

## Traefik — `dynamic.v2.yml` (Alvo)

Arquivo separado, ativado em paralelo. O Traefik suporta múltiplos arquivos de configuração via `providers.file.directory`.

```yaml
# infrastructure/traefik/dynamic.v2.yml
# Configuração ALVO: 10 domínios consolidados
# Ativado em paralelo com dynamic.yml durante migração via feature flags

http:
  middlewares:
    # Rate limit diferenciado por domínio (substituir global 100 req/min)
    ratelimit-default:
      rateLimit:
        average: 200
        burst: 400

    ratelimit-payments:        # PIX tem SLA estrito
      rateLimit:
        average: 500
        burst: 1000

    ratelimit-fraud:           # Fraud precisa ser rápido, não limitado artificialmente
      rateLimit:
        average: 1000
        burst: 2000

    ratelimit-compliance:      # Compliance: baixo volume, crítico
      rateLimit:
        average: 50
        burst: 100

    # Header CORS para todos os serviços
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

    # Strip prefix para legacy paths que foram consolidados
    strip-prefix-poupanca:
      stripPrefix:
        prefixes:
          - "/api/poupanca"
    strip-prefix-salario:
      stripPrefix:
        prefixes:
          - "/api/salario"

  routers:
    # ──────── BANKING CORE ────────
    banking-core:
      rule: "PathPrefix(`/api/core`) || PathPrefix(`/api/contas`) || PathPrefix(`/api/transacoes`) || PathPrefix(`/api/settlement`) || PathPrefix(`/api/pricing`)"
      service: svc-banking
      middlewares:
        - ratelimit-default
        - cors-headers
      entrypoints:
        - web

    # Alias legado — poupança e salário viram subpaths de banking-core
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

    # ──────── PAYMENTS ────────
    payments:
      rule: "PathPrefix(`/api/pix`) || PathPrefix(`/api/boleto`) || PathPrefix(`/api/agendamento`)"
      service: svc-payments
      middlewares:
        - ratelimit-payments
        - cors-headers
      entrypoints:
        - web

    # ──────── CREDIT ────────
    credit:
      rule: "PathPrefix(`/api/credit`) || PathPrefix(`/api/consignado`) || PathPrefix(`/api/financiamento`)"
      service: svc-credit
      middlewares:
        - ratelimit-default
        - cors-headers
      entrypoints:
        - web

    # ──────── PRODUCTS ────────
    products:
      rule: "PathPrefix(`/api/investimento`) || PathPrefix(`/api/cambio`) || PathPrefix(`/api/seguros`) || PathPrefix(`/api/cartoes`) || PathPrefix(`/api/treasury`)"
      service: svc-products
      middlewares:
        - ratelimit-default
        - cors-headers
      entrypoints:
        - web

    # ──────── CUSTOMER IDENTITY ────────
    customer:
      rule: "PathPrefix(`/api/customer`) || PathPrefix(`/api/kyc`) || PathPrefix(`/api/onboarding`) || PathPrefix(`/api/auth`) || PathPrefix(`/api/mfa`) || PathPrefix(`/api/security`)"
      service: svc-customer
      middlewares:
        - ratelimit-default
        - cors-headers
      entrypoints:
        - web

    # ──────── FRAUD & RISK ────────
    fraud:
      rule: "PathPrefix(`/api/fraud`) || PathPrefix(`/api/risk`)"
      service: svc-fraud
      middlewares:
        - ratelimit-fraud
      entrypoints:
        - web

    # ──────── COMPLIANCE ────────
    compliance:
      rule: "PathPrefix(`/api/compliance`) || PathPrefix(`/api/audit`) || PathPrefix(`/api/bacen`) || PathPrefix(`/api/tax`) || PathPrefix(`/api/accounting`)"
      service: svc-compliance
      middlewares:
        - ratelimit-compliance
      entrypoints:
        - web

    # ──────── FINANCE MGMT ────────
    finance-mgmt:
      rule: "PathPrefix(`/api/controller`) || PathPrefix(`/api/budget`) || PathPrefix(`/api/cost`) || PathPrefix(`/api/financial`)"
      service: svc-finance-mgmt
      middlewares:
        - ratelimit-default
      entrypoints:
        - web

    # ──────── PLATFORM ────────
    platform:
      rule: "PathPrefix(`/api/billing`) || PathPrefix(`/api/provisioning`) || PathPrefix(`/api/webhooks`) || PathPrefix(`/api/notification`) || PathPrefix(`/api/catalog`) || PathPrefix(`/api/baas`) || PathPrefix(`/aureus-organization`)"
      service: svc-platform
      middlewares:
        - ratelimit-default
        - cors-headers
      entrypoints:
        - web

    # ──────── INTELLIGENCE ────────
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
          interval: 10s   # mais frequente — serviço crítico

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

---

## Dockerfile Padrão (todos os 10 domínios)

```dockerfile
# backend/svc-{domain}/Dockerfile
# Mesmo padrão para todos os 10 — build otimizado com cache de dependências

FROM eclipse-temurin:25-jdk-jammy AS builder
WORKDIR /build

# Cache de dependências Maven (invalidado só quando pom.xml muda)
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw mvnw
RUN chmod +x mvnw && ./mvnw dependency:go-offline -q

# Build
COPY src src
RUN ./mvnw package -DskipTests -q

# ── Runtime image ──
FROM eclipse-temurin:25-jre-jammy
WORKDIR /app

RUN groupadd -r aureus && useradd -r -g aureus aureus
USER aureus

COPY --from=builder /build/target/*.jar app.jar

# JVM otimizada para container
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -XX:+ExitOnOutOfMemoryError \
               -Djava.security.egd=file:/dev/./urandom"

EXPOSE 820X  # substituir pela porta do domínio

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

---

## CI/CD — Pipeline por Domínio

### Estrutura de pipelines (GitHub Actions)

```
.github/workflows/
  ci-banking.yml           # trigger: backend/svc-banking/**
  ci-payments.yml          # trigger: backend/svc-payments/**
  ci-credit.yml            # trigger: backend/svc-credit/**
  ci-products.yml          # trigger: backend/svc-products/**
  ci-customer.yml          # trigger: backend/svc-customer/**
  ci-fraud.yml             # trigger: backend/svc-fraud/**
  ci-compliance.yml        # trigger: backend/svc-compliance/**
  ci-finance-mgmt.yml      # trigger: backend/svc-finance-mgmt/**
  ci-platform.yml          # trigger: backend/svc-platform/**
  ci-intelligence.yml      # trigger: backend/svc-intelligence/**
  ci-infra.yml             # trigger: infrastructure/**
  ci-frontend.yml          # trigger: frontend/** (mantido)
```

### Template de pipeline (`ci-{domain}.yml`)

```yaml
# .github/workflows/ci-banking.yml (exemplo)
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
          POSTGRES_DB: aureus_test
          POSTGRES_USER: aureus
          POSTGRES_PASSWORD: aureus
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
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/aureus_test
          SPRING_DATASOURCE_USERNAME: aureus
          SPRING_DATASOURCE_PASSWORD: aureus

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
          docker build -t aureus/svc-banking:${{ github.sha }} \
            backend/svc-banking/

      - name: Push to registry
        run: |
          echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login -u "${{ secrets.REGISTRY_USER }}" --password-stdin
          docker push aureus/svc-banking:${{ github.sha }}
          docker tag aureus/svc-banking:${{ github.sha }} aureus/svc-banking:latest
          docker push aureus/svc-banking:latest
```

### Durante a migração: pipeline híbrido

Enquanto os módulos legados ainda existem, o CI atual (`backend/**`) continua rodando. Novos pipelines por domínio sobem em paralelo, sem afetar o CI existente.

---

## `pom.xml` — Maven Parent Alvo

O parent raiz muda de 43 módulos para 10:

```xml
<!-- backend/pom.xml (ALVO — após migração completa) -->
<modules>
    <!-- Biblioteca compartilhada -->
    <module>aureus-shared</module>

    <!-- 10 domínios consolidados -->
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
</modules>

<!-- Durante a migração: manter os 43 módulos legados + adicionar os novos -->
<!-- Remover módulos legados conforme cada fase é concluída -->
```

---

## Health Check Dashboard

Substituir o check por `curl` (que retorna 200 mesmo com dependências down) por Spring Boot Actuator com verificação real:

```yaml
# application.yml padrão dos 10 novos domínios
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
      show-components: always
  health:
    db:
      enabled: true       # verifica PostgreSQL
    redis:
      enabled: true       # verifica Redis
    kafka:
      enabled: true       # verifica Kafka

# Portas separadas para app vs management
server:
  port: 820X  # porta do domínio
management:
  server:
    port: 920X  # porta de management (+1000 na porta do domínio)
    # 9200 → banking, 9201 → payments, ... 9209 → intelligence
```

Benefício: healthcheck do Docker Compose passa a verificar `/actuator/health/readiness` (que falha se banco está down) em vez de `/health` (que sempre retorna 200).

---

## Makefile — Comandos de Migração

```makefile
# Makefile (raiz do projeto)

# Subir apenas infra (sem nenhum serviço de aplicação)
infra-up:
	docker compose -f infrastructure/docker-compose.yml up -d \
	  traefik postgres redis kafka kafka-ui keycloak

# Subir legado (43 módulos atuais)
legacy-up:
	docker compose -f infrastructure/docker-compose.yml up -d

# Subir novo domínio específico em paralelo ao legado
domain-up:
	docker compose \
	  -f infrastructure/docker-compose.yml \
	  -f infrastructure/docker-compose.v2.yml \
	  up -d $(DOMAIN)
# Uso: make domain-up DOMAIN=svc-customer

# Ativar feature flag no Traefik (roteamento para novo domínio)
# Adiciona header X-Migration-Domain no middleware do domínio
flag-on:
	@echo "Ativando feature flag para $(DOMAIN)"
	# Editar dynamic.yml via script para adicionar roteamento com header

# Subir todos os 10 novos domínios (sem legado)
v2-up:
	docker compose -f infrastructure/docker-compose.v2.yml up -d

# Derrubar legado por domínio conforme migração avança
legacy-down:
	docker compose -f infrastructure/docker-compose.yml stop $(SERVICES)
# Uso: make legacy-down SERVICES="aureus-customer aureus-kyc aureus-onboarding aureus-security"

# Build completo dos 10 domínios
build-all:
	cd backend && ./mvnw clean install -DskipTests

# Testes de todos os domínios
test-all:
	cd backend && ./mvnw test

# Verificar portas em uso (detectar conflitos)
check-ports:
	@for port in 8200 8201 8202 8203 8204 8205 8206 8207 8208 8209; do \
	  lsof -i :$$port > /dev/null 2>&1 && echo "OCUPADA: $$port" || echo "livre: $$port"; \
	done
```

---

## Critérios de Conclusão da Fase 0 (este spec)

Antes de começar Fase F1 (migração de `svc-customer`):

- [ ] `docker-compose.v2.yml` criado e validado (`docker compose config` sem erros)
- [ ] `infrastructure/traefik/dynamic.v2.yml` criado e testado localmente
- [ ] Dockerfile padrão definido e buildando sem erro para ao menos 1 domínio
- [ ] Pipelines CI criados para todos os 10 domínios (podem estar vazios, mas existem)
- [ ] `Makefile` com targets `infra-up`, `domain-up`, `v2-up`, `legacy-down`
- [ ] `pom.xml` raiz com módulos legados + slots para os 10 novos (comentados)
- [ ] `application.yml` padrão com Actuator configurado (health com db/redis/kafka)
- [ ] `make check-ports` mostrando 8200–8209 livres

---

## O que esta camada NÃO define

| Decisão | Camada |
|---|---|
| Qual código Java vai para qual domínio | Camada 2 |
| Shared library vs. duplicação de código | Camada 2 |
| Schema separado vs. banco separado | Camada 3 |
| Migração de dados (Flyway cross-schema) | Camada 3 |
| Tópicos Kafka por domínio | Camada 3 |
| Eventos de integração entre domínios | Camada 3 |

*Referência: architecture_critique.md — design dos 10 domínios aprovado em Julho/2026.*
