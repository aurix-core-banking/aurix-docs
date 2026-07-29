# Suporte

Canal, classificação de tickets, SLA, informações para abertura de ticket, runbook e escalação.

---

## Canal

- **Email**: suporte@aurix.example.com (substituir pelo domínio real).
- **Portal de tickets**: URL do sistema de tickets (Zendesk, Freshdesk, Jira Service Management, etc.).

Antes de abrir ticket: consultar [Portal do desenvolvedor](../../03-development/portal-desenvolvedor/README.md), [webhooks](../../03-development/portal-desenvolvedor/webhooks.md), [wiki](../README.md) e [runbook](../../05-infrastructure/infrastructure/aurix-cloud-runbook.md).

---

## Classificação e SLA (exemplo)

| Severidade | Descrição | Primeira resposta (exemplo) |
|------------|-----------|----------------------------|
| Crítica | Sistema indisponível ou perda de dados | 1 hora |
| Alta | Funcionalidade principal quebrada (ex.: PIX não processa) | 4 horas |
| Média | Problema com contorno ou impacto limitado | 1 dia útil |
| Baixa | Dúvida, melhoria, documentação | 3 dias úteis |

Os tempos são exemplos; o contrato com o cliente define os SLAs efetivos.

---

## Informações no ticket

- Tenant ID (ou instituição).
- Ambiente: homologação ou produção.
- Descrição: o que estava tentando fazer e o que aconteceu.
- Logs/evidências: trecho de log, response HTTP, código de erro (sem dados sensíveis).
- Horário aproximado do ocorrido.

---

## Runbook e escalação

- **Runbook**: [aurix-cloud-runbook.md](../../05-infrastructure/infrastructure/aurix-cloud-runbook.md) – procedimentos para operação (deploy, latência, banco, backup, incidente).
- **Escalação**: tickets críticos ou altos não resolvidos no SLA devem ser escalados para engenharia ou plantão conforme política interna.
- **Pós-incidente**: para incidentes críticos, realizar revisão e atualizar runbook e documentação quando aplicável.

**Detalhes**: [suporte-tecnico.md](../../01-guides-checklists/kit-implementacao/suporte-tecnico.md).

[Voltar ao ciclo de vida](README.md) | [Índice da wiki](../../README.md)
