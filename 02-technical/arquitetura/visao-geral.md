# AURIX - Arquitetura

## Visão Geral

**Objetivo**: Plataforma de core banking para o mercado brasileiro, com 13 domínios consolidados e conformidade regulatória nativa.

**Abordagem**: Maven multi-module monorepo com serviços Spring Boot independentes, comunicação via REST síncrono + Kafka event-driven (outbox transacional). Todos os serviços compartilham um único PostgreSQL (`aurix_db`).

## Princípios Arquiteturais

### 1. Domínios Consolidados
- **13 serviços** `svc-*` cada um responsável por um domínio de negócio
- **aurix-shared** — biblioteca compartilhada (entidades JPA, DTOs, eventos, EventHub, cache, crypto, tenant)
- **aurix-gateway** — gateway fino com segurança por API key

### 2. API-First
- APIs RESTful com OpenAPI 3.0 (`aurix-api-specs`)
- Documentação automática via Swagger UI
- Gateway roteamento `/api/*` para todos os serviços

### 3. Comunicação
- **Síncrona**: REST (client gerado de OpenAPI)
- **Assíncrona**: Kafka com outbox transacional (`OutboxEvent` → `OutboxRelay` no `svc-banking`)
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
│        Traefik / aurix-gateway (:8080)              │
│         Rate Limit · CORS · JWT Auth                │
└──────────────────────┬──────────────────────────────┘
                       │
    ┌──────────────────┼──────────────────┐
    │                  │                  │
┌───▼──────┐  ┌────────▼──────┐  ┌────────▼────────┐
│svc-banking│  │svc-payments  │  │  svc-credit     │
│  :8200   │  │  :8201       │  │   :8082         │
│contas/PIX│  │PIX/SPI/STR   │  │Consig/Financ/   │
│poupança  │  │              │  │Garantias        │
└───┬──────┘  └──────┬───────┘  └────────┬────────┘
    │                │                   │
┌───▼──────┐  ┌──────▼───────┐  ┌────────▼────────┐
│svc-customer│ │svc-products │  │svc-finance-mgmt │
│  :8083    │  │  :8084      │  │     :8089       │
│Onboard/KYC│  │Catálogo     │  │Contabilidade    │
└───┬──────┘  └──────┬───────┘  └────────┬────────┘
    │                │                   │
┌───▼──────┐  ┌──────▼───────┐  ┌────────▼────────┐
│svc-cambio│  │svc-cards     │  │  svc-platform   │
│  :8093   │  │  :8094       │  │     :8092       │
│Câmbio/   │  │Crédito/Débito│  │Plataforma/BaaS  │
│BACEN     │  │Faturas       │  │Webhooks         │
└───┬──────┘  └──────┬───────┘  └────────┬────────┘
    │                │                   │
┌───▼──────┐  ┌──────▼───────┐  ┌────────▼────────┐
│svc-compliance│ │ svc-ai    │  │  svc-fraud      │
│  :8205   │  │  :8206       │  │     :8207       │
│Regulação │  │LLM/RAG       │  │Fraude (Kafka)   │
│AML/KYC   │  │Integração ML │  │consumer         │
└──────────┘  └──────────────┘  └─────────────────┘
```

> Diagramas C4 (Context, Container, Component) em Mermaid: [c4-diagramas.md](c4-diagramas.md).

## Serviços

| Serviço | Porta | Domínio | Responsabilidades | Dependências |
|---------|-------|---------|-------------------|--------------|
| `aurix-gateway` | 8080 | Gateway | Roteamento, API key, rate limit | Traefik |
| `svc-banking` | 8200 | Banking | Contas, transações, poupança, salário, pricing, liquidação | PostgreSQL, Kafka, Redis |
| `svc-payments` | 8201 | Pagamentos | PIX, SPI/STR | PostgreSQL, Kafka, Redis |
| `svc-credit` | 8082 | Crédito | Empréstimos, consignado, financiamento, garantias | PostgreSQL, Kafka, Redis |
| `svc-customer` | 8083 | Clientes | Onboarding PF/PJ, KYC, auth, JWT, MFA | PostgreSQL, Keycloak |
| `svc-products` | 8084 | Produtos | Catálogo de produtos, elegibilidade, tarifas (poupança, salário, investimentos, seguros) | PostgreSQL |
| `svc-contracts` | 8085 | Contratos | Gestão de contratos, assinaturas digitais, templates | PostgreSQL |
| `svc-finance-mgmt` | 8089 | Financeiro | Contabilidade, orçamento, custos, IFRS9, impostos | PostgreSQL |
| `svc-intelligence` | 8091 | Inteligência | Analytics, BI, chatbot, ML fraude | PostgreSQL |
| `svc-platform` | 8092 | Plataforma | Open Finance, BaaS, webhooks, notificações, audit | PostgreSQL |
| `svc-cambio` | 8093 | Câmbio | Câmbio, BACEN, remessas | PostgreSQL |
| `svc-cards` | 8094 | Cartões | Cartões crédito/débito, faturas, adquirentes | PostgreSQL |
| `svc-compliance` | 8205 | Compliance | Regulação, AML/KYC, auditoria | PostgreSQL |
| `svc-ai` | 8206 | IA | LLM, RAG, agentes, integração ML (gRPC) | PostgreSQL, Redis, ML (gRPC) |
| `svc-fraud` | 8207 | Fraude | Detecção de fraude, scoring (consumer Kafka) | Kafka, PostgreSQL |

> Observações:
> - `svc-contracts` (gestão de contratos com assinaturas e templates) foi implementado na porta 8085, está no docker-compose, é roteado no gateway (`/api/contracts/**`) e tem health check E2E (ver [ADR-0004](adr/0004-produtos-e-contratos.md)).
> - Todos os serviços usam `aurix-shared` e compartilham o mesmo PostgreSQL `aurix_db`.

## Fluxos Principais

### Fluxo PIX
```
Cliente → Gateway → svc-payments → Validação → PostgreSQL
         svc-banking → Kafka (outbox) → Evento de domínio
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
- **AES-256**: Criptografia de dados sensíveis (`aurix-shared/crypto`)
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
| Build | Maven multi-module (`mvnw`) |
| Database | PostgreSQL 15 (banco único) |
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
