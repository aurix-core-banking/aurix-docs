# Tasks 6, 7, 8 — Endpoint Mapping, Swagger UI, Contract Tests

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix Traefik route mismatches, add Swagger UI to all services, and create contract tests for cross-service integrations.

**Architecture:** Three independent workstreams: (1) Fix routing — align Traefik paths with actual controller `@RequestMapping` prefixes and resolve port conflicts; (2) Enable Swagger UI via springdoc for each service; (3) Add Spring Cloud Contract tests for key service interactions.

**Tech Stack:** Traefik v3, Spring Boot 4.1, springdoc-openapi 2.x, Spring Cloud Contract 4.x, Pact.

---

## Task 1: Fix Port Conflicts and Traefik Routes

**Problem:** Two pairs of services share ports, and Traefik routes don't match actual controller paths.

**Port conflicts:**
- `svc-credit` (8203) and `svc-products` (8203) — same port
- `svc-customer` (8204) and `svc-fraud` (8204) — same port

**Traefik route mismatches:**
- `/api/cambio` → `svc-products` (wrong, should be `svc-cambio`)
- `/api/cartoes` → `svc-products` (wrong, should be `svc-cards`)
- `/api/analytics` → `svc-intelligence` (ok, but `/api/openfinance` → `svc-intelligence` is wrong, should be `svc-platform`)
- `svc-cambio` controllers have NO `/api/cambio` prefix (use `/clientes`, `/contratos`, etc.)

**Files:**
- Modify: `apps/backend/svc-products/src/main/resources/application.yml` — change port to 8084
- Modify: `apps/backend/svc-customer/src/main/resources/application.yml` — keep port 8083
- Modify: `apps/backend/svc-fraud/src/main/resources/application.yml` — change port to 8204→8206
- Modify: `infra/traefik/dynamic.v2.yml` — fix all routes and ports
- Modify: `apps/backend/svc-cambio/src/main/java/com/aurix/platform/cambio/controller/*.java` — add `/api/cambio` prefix

- [ ] **Step 1: Fix port conflicts**

```bash
# svc-products: 8203 → 8084
# svc-customer: keep 8204 → 8083
# svc-fraud: 8204 → 8206
# svc-compliance: 8205 → 8205 (ok)
# svc-credit: 8203 → 8082
```

Edit `apps/backend/svc-products/src/main/resources/application.yml`:
```yaml
server:
  port: 8084
```

Edit `apps/backend/svc-credit/src/main/resources/application.yml`:
```yaml
server:
  port: 8082
```

Edit `apps/backend/svc-customer/src/main/resources/application.yml`:
```yaml
server:
  port: 8083
```

Edit `apps/backend/svc-fraud/src/main/resources/application.yml`:
```yaml
server:
  port: 8206
```

- [ ] **Step 2: Fix svc-cambio controller prefixes**

All 6 controllers need `/api/cambio` prefix added:

Edit `svc-cambio/src/main/java/com/aurix/platform/cambio/controller/AdminController.java`:
```java
@RequestMapping("/api/cambio/clientes")
```

Edit `svc-cambio/src/main/java/com/aurix/platform/cambio/controller/BacenController.java`:
```java
@RequestMapping("/api/cambio")
```

Edit `svc-cambio/src/main/java/com/aurix/platform/cambio/controller/ContratoController.java`:
```java
@RequestMapping("/api/cambio/contratos")
```

Edit `svc-cambio/src/main/java/com/aurix/platform/cambio/controller/CotacaoController.java`:
```java
@RequestMapping("/api/cambio/cotacoes")
```

Edit `svc-cambio/src/main/java/com/aurix/platform/cambio/controller/RemessaController.java`:
```java
@RequestMapping("/api/cambio/remessas")
```

Edit `svc-cambio/src/main/java/com/aurix/platform/cambio/controller/SpiStrController.java`:
```java
@RequestMapping("/api/cambio/spi-str")
```

- [ ] **Step 3: Rewrite Traefik dynamic config**

Replace `infra/traefik/dynamic.v2.yml` with corrected routes:

```yaml
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

  routers:
    # ── svc-banking (8200) ──
    banking-core:
      rule: "PathPrefix(`/api/core`) || PathPrefix(`/api/transacoes`) || PathPrefix(`/api/settlement`) || PathPrefix(`/api/pricing`) || PathPrefix(`/api/controle-saldos`) || PathPrefix(`/api/gestao-risco`) || PathPrefix(`/api/liquidacao`) || PathPrefix(`/api/motor-tarifas`) || PathPrefix(`/api/remuneracao`)"
      service: svc-banking
      middlewares: [ratelimit-default, cors-headers]
      entrypoints: [web]

    banking-empresas:
      rule: "PathPrefix(`/api/banking`)"
      service: svc-banking
      middlewares: [ratelimit-default, cors-headers]
      entrypoints: [web]

    banking-poupanca:
      rule: "PathPrefix(`/api/poupanca`)"
      service: svc-banking
      middlewares: [ratelimit-default, cors-headers]
      entrypoints: [web]

    banking-salario:
      rule: "PathPrefix(`/api/salario`)"
      service: svc-banking
      middlewares: [ratelimit-default, cors-headers]
      entrypoints: [web]

    # ── svc-payments (8201) ──
    payments-pix:
      rule: "PathPrefix(`/api/pix`)"
      service: svc-payments
      middlewares: [ratelimit-payments, cors-headers]
      entrypoints: [web]

    # ── svc-credit (8082) ──
    credit:
      rule: "PathPrefix(`/api/credit`) || PathPrefix(`/api/consignado`) || PathPrefix(`/api/financiamento`)"
      service: svc-credit
      middlewares: [ratelimit-default, cors-headers]
      entrypoints: [web]

    # ── svc-customer (8083) ──
    customer:
      rule: "PathPrefix(`/api/customer`) || PathPrefix(`/api/mfa`) || PathPrefix(`/api/security`) || PathPrefix(`/api/onboarding`) || PathPrefix(`/auth`) || PathPrefix(`/api/clientes`) || PathPrefix(`/solicitacoes`)"
      service: svc-customer
      middlewares: [ratelimit-default, cors-headers]
      entrypoints: [web]

    # ── svc-cambio (8093) ──
    cambio:
      rule: "PathPrefix(`/api/cambio`)"
      service: svc-cambio
      middlewares: [ratelimit-default, cors-headers]
      entrypoints: [web]

    # ── svc-cards (8094) ──
    cards:
      rule: "PathPrefix(`/api/cards`)"
      service: svc-cards
      middlewares: [ratelimit-default, cors-headers]
      entrypoints: [web]

    # ── svc-finance-mgmt (8089) ──
    finance-mgmt:
      rule: "PathPrefix(`/api/finance`)"
      service: svc-finance-mgmt
      middlewares: [ratelimit-default]
      entrypoints: [web]

    # ── svc-intelligence (8091) ──
    intelligence:
      rule: "PathPrefix(`/api/intelligence`)"
      service: svc-intelligence
      middlewares: [ratelimit-default, cors-headers]
      entrypoints: [web]

    # ── svc-platform (8092) ──
    platform:
      rule: "PathPrefix(`/api/platform`) || PathPrefix(`/api/openfinance`) || PathPrefix(`/api/webhooks`) || PathPrefix(`/api/notificacoes`)"
      service: svc-platform
      middlewares: [ratelimit-default, cors-headers]
      entrypoints: [web]

    # ── svc-fraud (8206) ──
    fraud:
      rule: "PathPrefix(`/api/fraud`)"
      service: svc-fraud
      middlewares: [ratelimit-fraud]
      entrypoints: [web]

    # ── svc-compliance (8205) ──
    compliance:
      rule: "PathPrefix(`/api/compliance`)"
      service: svc-compliance
      middlewares: [ratelimit-compliance]
      entrypoints: [web]

    # ── svc-ai (8206) ──
    ai:
      rule: "PathPrefix(`/api/ai`)"
      service: svc-ai
      middlewares: [ratelimit-default, cors-headers]
      entrypoints: [web]

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
          - url: "http://svc-credit:8082"
        healthCheck:
          path: /actuator/health
          interval: 30s

    svc-customer:
      loadBalancer:
        servers:
          - url: "http://svc-customer:8083"
        healthCheck:
          path: /actuator/health
          interval: 30s

    svc-cambio:
      loadBalancer:
        servers:
          - url: "http://svc-cambio:8093"
        healthCheck:
          path: /actuator/health
          interval: 30s

    svc-cards:
      loadBalancer:
        servers:
          - url: "http://svc-cards:8094"
        healthCheck:
          path: /actuator/health
          interval: 30s

    svc-finance-mgmt:
      loadBalancer:
        servers:
          - url: "http://svc-finance-mgmt:8089"
        healthCheck:
          path: /actuator/health
          interval: 30s

    svc-intelligence:
      loadBalancer:
        servers:
          - url: "http://svc-intelligence:8091"
        healthCheck:
          path: /actuator/health
          interval: 30s

    svc-platform:
      loadBalancer:
        servers:
          - url: "http://svc-platform:8092"
        healthCheck:
          path: /actuator/health
          interval: 30s

    svc-fraud:
      loadBalancer:
        servers:
          - url: "http://svc-fraud:8206"
        healthCheck:
          path: /actuator/health
          interval: 10s

    svc-compliance:
      loadBalancer:
        servers:
          - url: "http://svc-compliance:8205"
        healthCheck:
          path: /actuator/health
          interval: 30s

    svc-ai:
      loadBalancer:
        servers:
          - url: "http://svc-ai:8206"
        healthCheck:
          path: /actuator/health
          interval: 30s
```

- [ ] **Step 4: Verify no port conflicts remain**

Run: `grep -r 'port:' apps/backend/svc-*/src/main/resources/application.yml | grep -v '#'