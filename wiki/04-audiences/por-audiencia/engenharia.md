# Engenharia

O que a engenharia precisa saber: ambiente de desenvolvimento, estrutura do código, APIs, testes, CI/CD, integrações e como contribuir.

---

## Pré-requisitos e setup

- **Java**, **Maven**, **Node**, **Docker**: conforme [plataforma.md](../../01-business/big-picture.md).
- **Setup completo**: [02-lifecycle/setup.md](../../02-lifecycle/ciclo-de-vida/setup.md).
- **Checklist plataforma de pé**: [plataforma-de-pe.md](../../01-guides-checklists/checklists/plataforma-de-pe.md).

---

## Estrutura do código

- **Backend**: módulos em `backend/` (gateway, core, credit, openfinance, etc.).
- **Frontend**: `frontend/aurix-admin/`, `frontend/aurix-web/`.
- **Data pipelines**: [data-pipelines/](../../04-data-ai/data-pipelines/), Airflow, dbt, Trino.

---

## APIs e portal do desenvolvedor

- **Portal desenvolvedor**: [portal-desenvolvedor/README.md](../../03-development/portal-desenvolvedor/README.md).
- **APIs**: [portal-desenvolvedor/apis/](../../03-development/portal-desenvolvedor/apis/).
- **Guias**: primeira conta, primeiro PIX em [portal-desenvolvedor/guias/](../../03-development/portal-desenvolvedor/guias/).

---

## Testes e qualidade

- **Testes E2E**: [testes/e2e.md](../../03-development/testes/e2e.md).
- **Swagger**: disponível nos serviços (ex.: gateway, credit).

---

## Integrações

- **Webhooks**, **sandbox**: documentados no portal do desenvolvedor.
- **Kit de implementação** (go live, guia PIX, seeds, templates): [kit-implementacao/README.md](../../01-guides-checklists/kit-implementacao/README.md).

[Voltar à audiência](README.md) | [Índice da wiki](../../README.md)
