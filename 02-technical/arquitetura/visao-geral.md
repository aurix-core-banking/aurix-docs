# AURIX - Arquitetura

## Visão Geral

**Objetivo**: Plataforma de core banking para o mercado brasileiro, com 13 domínios consolidados e conformidade regulatória nativa.

**Abordagem**: Maven multi-module monorepo com serviços Spring Boot independentes, communicate via REST síncrono + Kafka event-driven.

## Princípios Arquiteturais

### 1. Domínios Consolidados
- **13 serviços** `svc-*` cada um responsável por um domínio de negócio
- **aurix-shared** — biblioteca compartilhada (DTOs, configs, utils)
- **aurix-gateway** — roteamento interno (sendo substituído por Traefik)

### 2. API-First
- APIs RESTful com OpenAPI 3.0
- Documentação automática via Swagger UI
- Gateway roteamento `/api/*` para todos os serviços

### 3. Comunicação
- **Síncrona**: REST (client gerado de OpenAPI)
- **Assíncrona**: Kafka com outbox transacional
- **Saga**: Coreografada para fluxos multi-domínio
- Detalhes em [ADR-0001](adr/0001-comunicacao-entre-servicos.md)

## Arquitetura de Alto Nível

```
┌─────────────────────────────────────────────────────┐
│                   Clientes                          │
│            (Web, Mobile, Admin)                     │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│              Traefik / Gateway (:8080)              │
│         Rate Limit · CORS · JWT Auth                │
└──────────────────────┬──────────────────────────────┘
                       │
    ┌──────────────────┼──────────────────┐
    │                  │                  │
┌───▼────┐  ┌─────────▼───┐  ┌──────────▼──────┐
│Payments│  │   Credit    │  │   Customer      │
│ :8081  │  │   :8082     │  │   :8083         │
│Core/PIX│  │Cred/Consig  │  │Identity/KYC     │
└───┬────┘  └──────┬──────┘  └────────┬────────┘
    │              │                   │
┌───▼────┐  ┌──────▼──────┐  ┌────────▼────────┐
│Products│  │Finance Mgmt │  │  Intelligence   │
│ :8084  │  │   :8089     │  │     :8091       │
│Poup/Inv│  │Contab/Impost│  │  Analytics/BI   │
└───┬────┘  └──────┬──────┘  └────────┬────────┘
    │              │                   │
┌───▼────┐  ┌──────▼──────┐  ┌────────▼────────┐
│Cambio  │  │   Cards     │  │   Banking       │
│ :8093  │  │   :8094     │  │   :8095         │
│BACEN   │  │Cred/Deb     │  │Org/Empresas     │
└────────┘  └─────────────┘  └─────────────────┘

┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Fraud      │  │  Compliance  │  │     AI       │
│   :8204      │  │   :8205      │  │    :8206     │
│  Detection   │  │  AML/KYC     │  │  LLM/RAG     │
└──────────────┘  └──────────────┘  └──────────────┘
```

## Serviços

| Serviço | Porta | Domínio | Responsabilidades |
|---------|-------|---------|-------------------|
| `svc-payments` | 8081 | Pagamentos | Core bancário, contas, transações, PIX, boletos, SPI/STR |
| `svc-credit` | 8082 | Crédito | Análise de crédito, consignado, financiamento, garantias |
| `svc-customer` | 8083 | Clientes | Identidade, KYC, onboarding, auth |
| `svc-products` | 8084 | Produtos | Poupança, salário, investimento, seguros, tesouraria |
| `svc-finance-mgmt` | 8089 | Financeiro | Contabilidade, orçamento, custos, IFRS9, impostos |
| `svc-intelligence` | 8091 | Inteligência | Analytics, BI, chatbot, ML fraud |
| `svc-platform` | 8092 | Plataforma | Open Finance, BaaS, webhooks, notificações, audit |
| `svc-cambio` | 8093 | Câmbio | Câmbio, BACEN, SPI/STR, remessas |
| `svc-cards` | 8094 | Cartões | Cartões crédito/débito, faturas, adquirentes |
| `svc-banking` | 8095 | Banking | Organização, empresas, relacionamento |
| `svc-fraud` | 8204 | Fraude | Detecção de fraude, scoring, bloqueio preventivo |
| `svc-compliance` | 8205 | Compliance | Regulação, AML/KYC, auditoria |
| `svc-ai` | 8206 | IA | LLM, RAG, agentes LangChain4j |

## Fluxos Principais

### Fluxo PIX
```
Cliente → Gateway → svc-payments → Validação → PostgreSQL
         svc-payments → Kafka (outbox) → Evento PIX
         svc-fraud → Scoring → Bloqueio se suspeito
         svc-platform → Notificação → Cliente
```

### Fluxo Crédito
```
Cliente → Gateway → svc-credit → Simulação → PostgreSQL
         svc-credit → svc-customer → Dados do cliente
         svc-credit → svc-intelligence → Análise ML
         svc-credit → Decisão → Kafka → svc-platform (notificação)
```

## Segurança

- **JWT**: Tokens para autenticação (Keycloak IAM)
- **RBAC**: Role-based access control
- **MFA**: Multi-factor authentication
- **TLS**: Transport layer security
- **AES-256**: Criptografia de dados sensíveis
- **Rate Limiting**: 100 req/min por IP (Traefik)

## Observabilidade

- **Logs**: ELK Stack (Elasticsearch, Logstash, Kibana)
- **Métricas**: Prometheus + Grafana
- **Traces**: Jaeger (distributed tracing)
- **Health**: Spring Actuator (`/actuator/health`)

## Stack Tecnológica

| Camada | Tecnologia |
|--------|-----------|
| Language | Java 25 |
| Framework | Spring Boot 4.1 |
| Build | Maven 3.9+ |
| Database | PostgreSQL 15 |
| Cache | Redis 7 |
| Events | Kafka (Confluent 7.4) |
| Gateway | Traefik v3 |
| IAM | Keycloak 23 |
| Search | Elasticsearch 8.11 |
| Analytics | ClickHouse |
| Time Series | TimescaleDB |

---

**Última atualização**: Julho 2026  
**Versão**: 2.0.0  
**Status**: 13 domínios consolidados
