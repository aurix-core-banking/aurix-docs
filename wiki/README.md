# Wiki AUREUS – Documentação completa

Documentação em formato de livro: o que cada área precisa saber, guias de execução e referências para **setup**, **execução**, **manutenção** e **suporte** da plataforma AUREUS Core Banking.

---

## Como usar esta wiki

- **Por audiência**: Parte I – Negócio, SRE/Infra, Engenharia, Arquitetura, Áreas do banco.
- **Por fase do ciclo de vida**: Parte II – Setup, Run, Manutenção, Suporte.
- **Passo a passo**: Parte III – Guias de execução e checklists.
- **Aprofundamento**: Parte IV – Referências (links para documentação técnica).

---

# PARTE I – Por audiência (o que cada um precisa saber)

Índice: [04-audiences/README.md](04-audiences/README.md)

| # | Audiência | Documento |
|---|-----------|-----------|
| 1 | Negócio | [04-audiences/negocio.md](04-audiences/negocio.md) |
| 2 | SRE e Infraestrutura | [04-audiences/sre-infra.md](04-audiences/sre-infra.md) |
| 3 | Engenharia | [04-audiences/engenharia.md](04-audiences/engenharia.md) |
| 4 | Arquitetura | [04-audiences/arquitetura.md](04-audiences/arquitetura.md) |
| 5 | Áreas do banco | [04-audiences/areas-banco.md](04-audiences/areas-banco.md) |

---

# PARTE II – Setup, Run, Manutenção e Suporte

Índice: [02-lifecycle/README.md](02-lifecycle/README.md)

| Fase | Documento |
|------|-----------|
| Setup | [02-lifecycle/setup.md](02-lifecycle/setup.md) – Checklist: [01-guides-checklists/checklists/plataforma-de-pe.md](01-guides-checklists/checklists/plataforma-de-pe.md) |
| Run | [02-lifecycle/run.md](02-lifecycle/run.md) |
| Manutenção | [02-lifecycle/manutencao.md](02-lifecycle/manutencao.md) |
| Suporte | [02-lifecycle/suporte.md](02-lifecycle/suporte.md) |

---

# PARTE III – Guias de execução e checklists

**Índice de guias**: [01-guides-checklists/guias/README.md](01-guides-checklists/guias/README.md) | **Kit de implementação (go live)**: [01-guides-checklists/kit-implementacao/README.md](01-guides-checklists/kit-implementacao/README.md)

| Guia | Descrição | Link |
|------|-----------|------|
| Do zero ao primeiro PIX | Passo a passo para novo cliente colocar PIX no ar | [01-guides-checklists/kit-implementacao/guia-zero-primeiro-pix.md](01-guides-checklists/kit-implementacao/guia-zero-primeiro-pix.md) |
| Primeira conta | Guia desenvolvedor – primeira conta via API | [primeira-conta.md](../03-development/portal-desenvolvedor/guias/primeira-conta.md) |
| Primeiro PIX | Guia desenvolvedor – primeiro PIX via API | [primeiro-pix.md](../03-development/portal-desenvolvedor/guias/primeiro-pix.md) |
| Connections Airflow e dbt | Configuração do Data Lakehouse (connections, variáveis, buckets) | [AIRFLOW-DBT-CONNECTIONS.md](../04-data-ai/data-pipelines/AIRFLOW-DBT-CONNECTIONS.md) |

## Checklists

| Checklist | Uso | Link |
|-----------|-----|------|
| Plataforma de pé | Primeira vez: infra, backend, frontend, dados | [01-guides-checklists/checklists/plataforma-de-pe.md](01-guides-checklists/checklists/plataforma-de-pe.md) |
| Go live (novo cliente) | Antes do go live: BACEN, KYC, domínios, provisioning, RegTech, segurança | [01-guides-checklists/checklists/go-live.md](01-guides-checklists/checklists/go-live.md) |
| Atualização regulatória | Manter relatórios e layouts (BACEN, Receita) em dia | [01-guides-checklists/checklists/regulatorio.md](01-guides-checklists/checklists/regulatorio.md) |
| Data Lakehouse | Verificação e passos Airflow, Trino, MinIO, dbt, DAGs | [01-guides-checklists/checklists/data-lakehouse.md](01-guides-checklists/checklists/data-lakehouse.md) |

---

# PARTE IV – Referências

**Índice de referências**: [05-references/README.md](05-references/README.md)

## Documentação técnica (docs/)

| Tema | Documento |
|------|-----------|
| Índice da documentação | [docs/README.md](../README.md) |
| Big picture | [big-picture.md](../01-business/big-picture.md) |
| Plataforma (módulos, tech) | [plataforma.md](../01-business/big-picture.md) |
| Roadmap e status | [roadmap.md](../01-business/roadmap.md) |
| Changelog | [CHANGELOG.md](../CHANGELOG.md) |
| Arquitetura | [arquitetura/](../02-technical/arquitetura/) |
| Banco de dados | [banco-dados/](../banco-dados/) |
| Infraestrutura (14.1–14.7) | [infrastructure/index.md](../05-infrastructure/infrastructure/index.md) |
| Operação (runbook, backup, DR) | [infrastructure/](../05-infrastructure/infrastructure/index.md) |
| Conformidade | [03-compliance/](03-compliance/) |
| Portal desenvolvedor e APIs | [portal-desenvolvedor/](../03-development/portal-desenvolvedor/README.md) |
| Kit implementação (go live, seeds, templates, suporte) | [01-guides-checklists/kit-implementacao/README.md](01-guides-checklists/kit-implementacao/README.md) |
| Data pipelines e lakehouse | [data-pipelines/](../04-data-ai/data-pipelines/) |
| Testes E2E | [testes/e2e.md](../03-development/testes/e2e.md) |

## Mapa rápido por necessidade

| Preciso... | Onde ir |
|------------|---------|
| Subir a plataforma pela primeira vez | [02-lifecycle/setup.md](02-lifecycle/setup.md), [01-guides-checklists/checklists/plataforma-de-pe.md](01-guides-checklists/checklists/plataforma-de-pe.md) |
| Preparar go live de um cliente | [01-guides-checklists/kit-implementacao/README.md](01-guides-checklists/kit-implementacao/README.md), [01-guides-checklists/checklists/go-live.md](01-guides-checklists/checklists/go-live.md), [04-audiences/negocio.md](04-audiences/negocio.md) |
| Operar no dia a dia (deploy, alertas, incidente) | [02-lifecycle/run.md](02-lifecycle/run.md), [04-audiences/sre-infra.md](04-audiences/sre-infra.md), [Runbook](../05-infrastructure/infrastructure/aureus-cloud-runbook.md) |
| Atualizar relatórios regulatórios | [01-guides-checklists/checklists/regulatorio.md](01-guides-checklists/checklists/regulatorio.md), [02-lifecycle/manutencao.md](02-lifecycle/manutencao.md) |
| Desenvolver ou integrar via API | [04-audiences/engenharia.md](04-audiences/engenharia.md), [Portal desenvolvedor](../03-development/portal-desenvolvedor/README.md) |
| Entender decisões e evolução | [04-audiences/arquitetura.md](04-audiences/arquitetura.md), [roadmap.md](../01-business/roadmap.md) |
| Abrir ou tratar suporte | [02-lifecycle/suporte.md](02-lifecycle/suporte.md), [01-guides-checklists/kit-implementacao/suporte-tecnico.md](01-guides-checklists/kit-implementacao/suporte-tecnico.md) |

---

**Última atualização**: Fevereiro 2026 | **Versão wiki**: 1.0
