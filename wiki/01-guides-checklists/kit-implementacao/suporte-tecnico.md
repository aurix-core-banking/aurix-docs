# Processo de suporte técnico

Canal de suporte, SLA de resposta e referência a runbooks. Item 13.5 do roadmap.

---

## Canal

- **Email**: suporte@aurix.example.com (substituir pelo domínio real).
- **Portal de tickets**: URL do sistema de tickets (ex.: Zendesk, Freshdesk, Jira Service Management) para abertura e acompanhamento de chamados.
- **Documentação**: antes de abrir ticket, consultar [Portal do desenvolvedor](../../03-development/portal-desenvolvedor/README.md), [webhooks](../../03-development/portal-desenvolvedor/webhooks.md), [wiki](../../README.md) e [runbook AURIX Cloud](../../05-infra/infra/aurix-cloud-runbook.md).

---

## Classificação e SLA de resposta

| Severidade | Descrição | Tempo de primeira resposta (exemplo) |
|------------|-----------|--------------------------------------|
| **Crítica** | Sistema indisponível ou perda de dados | 1 hora |
| **Alta** | Funcionalidade principal quebrada (ex.: PIX não processa) | 4 horas |
| **Média** | Problema com contorno ou impacto limitado | 1 dia útil |
| **Baixa** | Dúvida, melhoria, documentação | 3 dias úteis |

Os tempos acima são exemplos; o contrato com o cliente deve definir os SLAs efetivos.

---

## Informações a enviar no ticket

- **Tenant ID** (ou instituição).
- **Ambiente**: homologação ou produção.
- **Descrição**: o que estava tentando fazer e o que aconteceu.
- **Logs/evidências**: trecho de log, response HTTP, código de erro (sem dados sensíveis).
- **Horário aproximado** do ocorrido.

---

## Runbook e escalação

- **Runbook operacional**: [aurix-cloud-runbook.md](../../05-infra/infra/aurix-cloud-runbook.md) – procedimentos para operação (deploy, latência, banco, backup, incidente).
- **Escalação**: tickets críticos ou altos que não forem resolvidos no SLA devem ser escalados para o time de engenharia ou plantão conforme política interna.
- **Pós-incidente**: para incidentes críticos, realizar reunião de revisão e atualizar runbook e documentação quando aplicável.

[Voltar ao kit](README.md) | [Ciclo de vida – Suporte](../../02-lifecycle/ciclo-de-vida/suporte.md) | [Índice da wiki](../../README.md)
