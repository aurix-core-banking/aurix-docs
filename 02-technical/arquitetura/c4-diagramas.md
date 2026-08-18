# Diagramas C4 – AURIX Core Banking Platform

Diagramas C4 (Context, Container, Component) em Mermaid, mantidos em sincronia com a arquitetura real (14 serviços `svc-*` + `aurix-gateway` + `aurix-shared`).

> Portas de referência (fonte: `aurix-infrastructure/docker-compose.yml` e `application.yml` de cada serviço).

## 1. Context (L0)

```mermaid
flowchart TB
  PF[Cliente Pessoa Física] --> WEB[aurix-web]
  PJ[Cliente Pessoa Jurídica] --> WEB
  WEB --> G[API Gateway Traefik :8080]
  ADM[Operação / Back-office] --> ADMIN[aurix-admin]
  ADMIN --> G
  MOB[Cliente Mobile] --> MOBILE[aurix-mobile]
  MOBILE --> G

  subgraph Sistema[AURIX Core Banking Platform]
    G --> SVC[Serviços svc-* (14 domínios)]
    SVC --> PG[(PostgreSQL 15)]
    SVC --> RD[(Redis 7)]
    SVC --> K[Kafka]
  end

  SVC --> BACEN[BACEN / SPB - mock :8095]
  SVC --> RF[Receita Federal]
  SVC --> BUR[Bureaus / ClearSale / Quod]
  SVC --> KC[Keycloak IAM :8443]

  K --> DP[Pipelines Spark/Flink]
  DP --> CH[(ClickHouse)]
  DP --> TS[(TimescaleDB)]
  DP --> ES[(Elasticsearch)]
  DP --> ML[Modelos de ML - FastAPI :50051 gRPC]
```

## 2. Container (L1)

```mermaid
flowchart TB
  subgraph Canais[Canais / UI - React]
    WEB[aurix-web :3000]
    ADMIN[aurix-admin :3001]
    MOBILE[aurix-mobile]
  end

  G[API Gateway - Traefik :8080]

  subgraph Backend["Backend - Java 25 / Spring Boot 4.1 (Maven multi-module)"]
    GW[aurix-gateway :8080<br/>API key + rate limit]
    BNK[svc-banking :8200<br/>contas, transações, poupança, salário, pricing, liquidação]
    PAY[svc-payments :8201<br/>PIX]
    CRD[svc-credit :8082<br/>empréstimos, financiamento, garantias]
    CUS[svc-customer :8083<br/>onboarding PF/PJ, KYC, MFA]
    PRD[svc-products :8084<br/>catálogo de produtos]
    CTR[svc-contracts :8085<br/>contratos, assinaturas, templates]
    FIN[svc-finance-mgmt :8089<br/>contabilidade]
    INT[svc-intelligence :8091]
    PLT[svc-platform :8092]
    CAM[svc-cambio :8093]
    CAR[svc-cards :8094]
    CMP[svc-compliance :8205]
    AI[svc-ai :8206]
    FRD[svc-fraud :8207<br/>consumer Kafka]
    SH[aurix-shared<br/>JAR - entidades, EventHub, outbox, crypto, tenant]
  end

  subgraph Dados[Dados e Mensageria]
    PG[(PostgreSQL 15 :5432<br/>banco único aurix_db)]
    RD[(Redis 7 :6379)]
    KF[Kafka :9092<br/>outbox + eventos de domínio]
  end

  subgraph Observabilidade
    PR[Prometheus :9090]
    GR[Grafana :3000]
    ES[(Elasticsearch :9200)]
  end

  subgraph Pipelines["Dados e Inteligência"]
    DP[ETL Spark/Flink + dbt/Airflow]
    ML[Serving ML - FastAPI]
  end

  Canais --> G
  G --> GW
  BNK & PAY & CRD & CUS & PRD & CTR & FIN & INT & PLT & CAM & CAR & CMP & AI --> SH
  GW --> BNK & PAY & CRD & CUS & PRD & CTR & FIN & INT & PLT & CAM & CAR & CMP & AI
  BNK & PAY & CRD & CUS & PRD & CTR & FIN & INT & PLT & CAM & CAR & CMP & AI --> PG
  BNK & PAY & CRD --> RD
  BNK & PAY --> KF
  FRD --> KF
  KF --> DP
  DP --> ES
  DP --> ML
  SVC --> PR
  PR --> GR
```

## 3. Component (L2) – exemplo `svc-banking`

O `svc-banking` segue o padrão **Controller → Service → Repository** e contém os sub-domínios `poupanca/`, `salario/`, `pricing/`, `settlement/` e `payment/`.

```mermaid
flowchart TB
  G[Gateway / Traefik] --> C[Controller REST<br/>/api/v1/contas, /transacoes...]

  subgraph svc-banking
    C --> S[Services<br/>ContaService, TransacaoService,<br/>PoupancaService, PricingService...]
    S --> R[Repositories JPA]
    S --> OUT[OutboxEventPublisher]
    S --> EH[EventHub - aurix-shared]
  end

  R --> PG[(PostgreSQL - aurix_db)]
  OUT --> KF[Kafka<br/>outbox-relay -> tópicos de domínio]
  EH --> KF
  KF --> FRD[svc-fraud - consumer]

  S --> EXT[Providers/Stubs<br/>BACEN, Bureau, ClearSale, Quod]
```

---

**Última atualização**: Agosto 2026  
**Fonte**: [visao-geral.md](visao-geral.md), [docker-compose.yml](../../05-infrastructure/infrastructure/index.md)
