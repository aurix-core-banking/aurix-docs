# Setup – Instalação e primeira execução

Guia para colocar a plataforma AURIX de pé pela primeira vez.

---

## Resumo

1. Pré-requisitos (Docker, Java, Maven, Node, Python, Git).
2. Stack de dados: `cd infrastructure/data-stack && docker-compose up -d`.
3. Backend: build e config (Postgres, Kafka); subir serviços.
4. Frontend: install e start (aurix-admin, aurix-web).
5. (Opcional) Airflow/Data Lakehouse: buckets MinIO, connections, DAGs.
6. Validação: health checks, testes.

**Checklist completo**: [01-guides-checklists/checklists/plataforma-de-pe.md](../../01-guides-checklists/checklists/plataforma-de-pe.md). Para go live de um novo cliente (PIX, seeds, templates, suporte): [01-guides-checklists/kit-implementacao/README.md](../../01-guides-checklists/kit-implementacao/README.md).

---

## Referências

- [Banco de dados](../../../02-technical/arquitetura/evolucao-arquitetura-dados.md)
- [Plataforma](../../01-business/big-picture.md)
- [Data Lakehouse – checklist](../../01-guides-checklists/checklists/data-lakehouse.md)
- [Connections Airflow e dbt](../../04-data-ai/data-pipelines/AIRFLOW-DBT-CONNECTIONS.md)
- [Instalação Aurix Admin](../../01-guides-checklists/guias/instalacao-admin.md)

[Voltar ao ciclo de vida](README.md) | [Índice da wiki](../../README.md)
