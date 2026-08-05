# Changelog - AURIX Core Banking Platform

Todas as mudanças notáveis neste projeto serão documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.0.0/),
e este projeto adere ao [Versionamento Semântico](https://semver.org/lang/pt-BR/).

## [2.0.0] - 2026-07

### Adicionado
- Runbooks operacionais (queda de serviço, lentidão no banco, falha no Kafka, incidente de segurança, rollback de deploy): `05-infrastructure/infrastructure/runbooks/`
- Diagramas C4 (Context, Container, Component) em Mermaid: `02-technical/arquitetura/c4-diagramas.md`
- Documentação dos 13 domínios `svc-*` com portas, responsabilidades e dependências

### Corrigido
- Portas da arquitetura atualizadas para o mapeamento real (svc-banking :8200, svc-payments :8201, svc-fraud :8207, etc.)
- Credenciais do PostgreSQL nos procedimentos de backup/restore (aurix_user / aurix_db)

### Alterado
- Arquitetura consolidada: módulos `aurix-*` migrados para 13 serviços `svc-*`
- `visao-geral.md` reescrito com a arquitetura real

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

**Última Atualização**: Julho 2026
