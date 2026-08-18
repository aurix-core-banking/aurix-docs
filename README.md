# Aurix Docs

Documentação da plataforma Aurix — arquitetura, ADRs, runbooks, diagramas C4, domínio bancário brasileiro.

## Conteúdo

- **architecture/** — arquitetura do sistema, diagramas C4
- **guides/** — setup de desenvolvimento, runbooks operacionais
- **decisions/** — Architecture Decision Records (ADRs)
- **adr/** — decisões de arquitetura documentadas

## ADRs Principais

| ADR | Título |
|---|---|
| ADR-0001 | Naming convention para Kafka topics (`dominio.entidade.evento.versao`) |
| ADR-0002 | Single database pattern (compartilhar `aurix_db`) |
| ADR-0003 | Outbox pattern para event publishing |
| ADR-0004 | Provider/Stub pattern para integrações externas |

## Domínio Bancário Brasileiro

Documentação de conceitos:
- **PIX** — Pagamento instantâneo 24/7 (SPI/STR)
- **TED/DOC** — Transferências interbancárias
- **Consignado** — Crédito com desconto em folha
- **Open Finance** — Ecossistema de dados abertos (FAPI-Brazil)
- **LGPD** — Proteção de dados pessoais
- **BACEN** — Banco Central do Brasil (regulamentação)

## Relacionados

- [aurix-backend](https://github.com/aurix-core-banking/aurix-backend)
- [aurix-infrastructure](https://github.com/aurix-core-banking/aurix-infrastructure)
