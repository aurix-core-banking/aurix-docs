# Changelog - AURIX Core Banking Platform

Todas as mudancas notaveis neste projeto serao documentadas neste arquivo.

O formato e baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.0.0/),
e este projeto adere ao [Versionamento Semantico](https://semver.org/lang/pt-BR/).

## [1.1.0] - 2025-01-26

### Adicionado
- Modulo AURIX Organization - Sistema completo de estrutura organizacional, controle de alcada, delegacao de poderes, APIs REST
- Integracoes: RH, eSocial, Receita Federal, LinkedIn, BI, Webhooks
- Novo modulo `aurix-organization` na porta 8086, 9 entidades JPA, 15+ repositorios, 8+ controllers
- 9 novas tabelas, indices, triggers, dados iniciais
- Documentacao do modulo Organization

## [1.0.0] - 2025-01-01

### Adicionado
- AURIX Core, PIX, Credit, Treasury, Security, Compliance, Analytics, Audit, Gateway, Shared
- Infraestrutura: Docker, PostgreSQL, Redis, Kafka, Elasticsearch, Prometheus, Grafana
- Java 17 + Spring Boot 3.2, Spring Security, Spring Data JPA, Maven, Swagger/OpenAPI

---

**Ultima Atualizacao**: Janeiro 2025
