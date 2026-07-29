# AURIX - Big Picture

A AURIX é uma plataforma de Core Banking completa e moderna, desenvolvida para o mercado brasileiro. Este documento fornece uma visão de alto nível sobre o propósito da plataforma, sua arquitetura e diferenciais competitivos.

---

## 🏛️ Identidade e Missão

O nome **AURIX** remete à moeda de ouro do Império Romano, simbolizando os pilares da nossa plataforma: **Estabilidade, Valor, Confiança e Padrão**.

**Slogan**: *"O padrão de excelência financeira."*

Nossa missão é fornecer uma infraestrutura financeira de classe mundial que combina a robustez dos sistemas legados com a agilidade e inovação das tecnologias cloud-native.

## 🚀 Diferenciais Competitivos

A AURIX foi desenhada para superar as limitações das soluções tradicionais:
- **Performance Superior**: Arquitetura otimizada para processamento de transações em massa com baixa latência.
- **APIs Modernas**: Abordagem *API-First*, facilitando a integração e reduzindo o *time-to-market*.
- **Conformidade Nativa**: Módulos dedicados para atender a todas as exigências do BACEN e normas regulatórias brasileiras desde o primeiro dia.
- **Evolução Contínua**: Preparada para o futuro com uma stack de dados de última geração (Data Lakehouse).

---

## 👷 Para Quem

- **Instituições Financeiras**: Que buscam modernizar seu core banking com segurança e conformidade (BACEN, LGPD).
- **Equipes de Tecnologia**: Que precisam de uma base sólida e modular, com backend, frontends, dados e ML integrados e bem documentados.

---

## Visão da Arquitetura

```mermaid
flowchart TB
  subgraph Canais["Canais / UI"]
    Web[aurix-web]
    Admin[aurix-admin]
    Mobile[aurix-mobile]
  end

  subgraph Gateway["API Gateway"]
    GW[Traefik :8080]
  end

  subgraph Backend["Backend - 13 Domínios"]
    PAY[payments :8081]
    CRD[credit :8082]
    CUS[customer :8083]
    PRD[products :8084]
    FIN[finance-mgmt :8089]
    INT[intelligence :8091]
    PLT[platform :8092]
    CAM[cambio :8093]
    CRD2[cards :8094]
    BNK[banking :8095]
    FRD[fraud :8204]
    CMP[compliance :8205]
    AI[ai :8206]
  end

  subgraph DataLayer["Camada de Dados"]
    PG[(PostgreSQL)]
    Redis[(Redis)]
    Kafka[Kafka]
    CH[(ClickHouse)]
    TS[(TimescaleDB)]
    ES[Elasticsearch]
  end

  subgraph Pipelines["Dados e Inteligência"]
    DP[Pipelines Spark/Flink]
    ML[Modelos de ML & MLOps]
  end

  Canais --> Gateway
  Gateway --> Backend
  Backend --> DataLayer
  DataLayer --> Pipelines
```

### Componentes Chave:
- **Canais**: Interfaces modernas em React (Web/Admin) e React Native (Mobile).
- **Gateway**: Traefik v3 — roteamento, rate limiting (100 req/min), CORS, JWT auth.
- **Backend**: 13 domínios consolidados, cada um com Application.java, controllers, services e repositórios próprios.
- **Dados**: PostgreSQL (transacional), Redis (cache/sessões), Kafka (eventos), ClickHouse (analytics), TimescaleDB (séries temporais).
- **Inteligência**: Pipelines Spark/Flink + modelos ML para fraude e análise de risco.

---

## 🛠️ Stack Tecnológica

- **Backend**: Java 25, Spring Boot 4.1, Spring Cloud Gateway, JPA.
- **Frontend**: React 18, Material-UI, React Native.
- **Dados**: PostgreSQL, Redis, Kafka, ClickHouse, Elasticsearch.
- **Operações**: Docker, Maven, Git, Apache Spark, Airflow.
- **Inteligência**: Python, FastAPI, MLflow.

---

## 🧭 Onde Encontrar Mais

| Tema | Documento |
|------|-----------|
| Guia Geral | [../README.md](../README.md) |
| Roadmap e Visão | [./roadmap.md](./roadmap.md) |
| Detalhes da Arquitetura | [../02-technical/arquitetura/visao-geral.md](../02-technical/arquitetura/visao-geral.md) |
| Operação e Infra | [../05-infrastructure/infrastructure/index.md](../05-infrastructure/infrastructure/index.md) |
| APIs e Integração | [../03-development/portal-desenvolvedor/README.md](../03-development/portal-desenvolvedor/README.md) |

---

**AURIX Core Banking Platform** - *O padrão de excelência financeira*
