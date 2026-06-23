# AUREUS - Roadmap e Visão do Produto

Este documento descreve as capacidades atuais da plataforma AUREUS e a nossa visão para o futuro, com foco especial na evolução para um Data Lakehouse moderno e integrado.

---

## 1. Visão Geral da Plataforma

A AUREUS é uma solução de Core Banking completa, projetada para atender às demandas do mercado brasileiro com alta performance, conformidade nativa e uma arquitetura orientada a dados.

A plataforma já oferece suporte robusto para:
- **Ecossistema Financeiro**: PIX, Crédito, Tesouraria, Liquidação e Pricing.
- **Gestão e Controle**: Organização, Billing, Provisioning e Contabilidade (IFRS9).
- **Conformidade e Segurança**: Módulos dedicados ao BACEN, Compliance, Auditoria e Segurança Cibernética.
- **Experiência do Usuário**: Backoffice administrativo completo e interfaces voltadas para o cliente final (Web/Mobile).
- **Infraestrutura Moderna**: Stack de dados baseada em tecnologias open-source líderes de mercado, preparada para alta disponibilidade e escalabilidade.

---

## 2. Capacidades Atuais

### Core Banking e Módulos de Negócio
A plataforma possui um núcleo modular que abrange desde a abertura de contas (Onboarding) até operações complexas de BaaS (Banking as a Service). O desenvolvimento do módulo de **Cartões** está em estágio avançado, expandindo ainda mais o leque de serviços oferecidos.

### Stack de Dados e Observabilidade
Nossa base tecnológica utiliza PostgreSQL para transações de baixa latência e Redis para cache. A infraestrutura de dados é composta por:
- **Streaming**: Kafka para processamento de eventos em tempo real.
- **Armazenamento e Analíticos**: ClickHouse para analytics de alta performance e MinIO para armazenamento de objetos.
- **Monitoramento**: Stack completa com Prometheus, Grafana e Elasticsearch/Kibana para visibilidade total da operação.

### Infraestrutura e Confiabilidade
A arquitetura foi desenhada para suportar missões críticas:
- **Alta Disponibilidade (HA)** e planos de **Disaster Recovery (DR)**.
- **Deploy Sem Downtime** utilizando estratégias de Blue/Green e Canary.
- **Segurança**: Gestão de identidade e acessos (IAM) e conformidade com normas regulatórias.

### Conformidade Regulatória
Geração nativa de relatórios fundamentais para o BACEN e Receita Federal, como COSIF, E-Financeira, SCR/CCS e SPED.

---

## 3. Direcionamento e Evolução (Roadmap)

Nosso foco principal está na consolidação do **Data Lakehouse AUREUS**, transformando a plataforma em uma potência analítica.

### Integração de Dados Modernas
- **Orquestração Inteligente**: Expansão do uso do Apache Airflow para gerenciar pipelines complexos de ingestão e transformação.
- **Arquitetura de Medalhão**: Implementação formal das camadas Bronze, Silver e Gold para garantir governança e qualidade dos dados.
- **Motor de Consulta Unificado**: Uso do Trino para permitir consultas SQL de alta velocidade através de diferentes fontes de dados.

### Inteligência e Produto
- **Expansão de Canais**: Evolução dos canais digitais e novas integrações para ecossistemas de e-commerce.
- **Analytics Avançado**: Criação de novos dashboards de BI e modelos de ML integrados diretamente ao Lakehouse para análise de crédito e fraude.
- **Integração de Dados de Mercado Abertos**: Unificação de dados públicos (CVM, Tesouro Direto, ANBIMA, B3) inspirada no modelo do Open Brazil Market (OBM), possibilitando marcação a mercado real e suporte a advisory de investimentos automatizado.
- **Automação de Infraestrutura**: Evolução contínua dos scripts de Terraform e Kubernetes para suportar operações de qualquer escala com o mínimo esforço manual.


---

## 4. Referências e Guias

Para detalhes técnicos e operacionais, consulte os guias na pasta `docs/`:

| Tema | Documento |
|------|------------|
| Visão Arquitetural | [big-picture.md](big-picture.md) |
| Runbook de Infraestrutura | [../05-infrastructure/infrastructure/index.md](../05-infrastructure/infrastructure/index.md) |
| Estratégia de Dados | [../04-data-ai/data-pipelines/README.md](../04-data-ai/data-pipelines/README.md) |
| Guia de Implementação | [../wiki/README.md](../wiki/README.md) |

---

**Última atualização**: Fevereiro 2026  
**Versão**: 1.2.0
