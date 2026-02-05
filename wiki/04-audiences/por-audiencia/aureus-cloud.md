# AUREUS Cloud – Acesso, limites e SLA

Documento voltado ao cliente que utiliza o AUREUS em modo SaaS (AUREUS Cloud). Item 8.5 do roadmap.

---

## O que é o AUREUS Cloud

O AUREUS Cloud é a oferta em que a plataforma AUREUS (core banking, PIX, crédito, regulatório, etc.) é operada por nós. Sua instituição é um tenant com ambiente dedicado: banco de dados separado e configuração própria (branding, limites, produtos habilitados), sem compartilhar dados com outras instituições.

---

## Como acessar

- **Portal e APIs**: após a ativação do seu tenant, você receberá:
  - URL base das APIs (ex.: `https://api.aureuscloud.example.com`).
  - Identificador do tenant (`tenant_id`) e instruções para enviar o header `X-Tenant-Id` em todas as requisições.
  - Credenciais de acesso (API Key e/ou OAuth2) conforme o plano contratado. Consulte o [Portal do desenvolvedor](../../03-development/portal-desenvolvedor/README.md) para documentação das APIs, exemplos e guias (primeira conta, primeiro PIX, etc.).
- **Internet Banking e painel**: se o plano incluir front-end white-label, serão fornecidos URL de acesso e fluxo de login (usuário/senha ou SSO).
- **Suporte**: canal de suporte técnico (e-mail ou portal de tickets), com SLA de resposta conforme o plano (ver abaixo). Use esse canal para dúvidas de acesso, integração e incidentes.

---

## Limites

Os limites da sua instituição são definidos pelo plano (Starter, Growth, Enterprise) e podem ser ajustados na configuração do tenant. Típicos:

- **Contas ativas**: número máximo de contas ativas permitidas no período.
- **Transações por mês**: volume de transações (ex.: PIX, TED, liquidações) no ciclo de billing.
- **Chamadas de API por mês**: volume de requisições às APIs do AUREUS Cloud no ciclo.

O uso atual e o histórico podem ser consultados no painel do cliente ou via API de billing (quando disponível). Aproximação dos limites gera notificação; o excesso pode ser bloqueado ou faturado conforme contrato.

---

## SLA (disponibilidade)

- **Meta de disponibilidade**: 99,9% de uptime mensal (tempo em que as APIs e serviços essenciais estão acessíveis), exceto janelas de manutenção comunicadas com antecedência.
- **Manutenção**: manutenções planejadas serão comunicadas por e-mail ou no painel, com data e janela estimada. Esforço para realização em horário de menor uso.
- **Crédito por indisponibilidade**: se a disponibilidade mensal ficar abaixo do acordado, crédito proporcional pode ser aplicado na fatura conforme termos do contrato.
- **Exclusões**: indisponibilidade causada por falhas de rede ou sistemas do cliente, uso indevido ou força maior não entram no cálculo do SLA.

---

## Segurança e dados

- **Isolamento**: seus dados ficam em banco de dados exclusivo do seu tenant; não há compartilhamento de dados com outros clientes.
- **Criptografia**: dados em trânsito (TLS) e em repouso conforme política do AUREUS Cloud e do cloud provider.
- **Conformidade**: operação alinhada à LGPD e exigências regulatórias aplicáveis (BACEN). Contrato e políticas de privacidade regem o tratamento de dados.

---

## Atualizações e tecnologia regulatória

- Atualizações da plataforma (correções, novas funcionalidades e adequações regulatórias) são aplicadas pelo operador. Cliente é notificado sobre mudanças relevantes (APIs, comportamento) conforme política de comunicação.
- Pacotes de conformidade regulatória (relatórios BACEN, SPED, etc.) são mantidos e atualizados pelo AUREUS Cloud dentro do escopo do plano; ofertas adicionais (consultoria, pacotes específicos) podem ser contratadas em separado.

---

## Referências

- [Portal do desenvolvedor](../../03-development/portal-desenvolvedor/README.md): documentação de APIs e guias.
- [Roadmap e status](../../roadmap.md): visão do produto e evolução.
- [Suporte técnico](../01-guides-checklists/kit-implementacao/suporte-tecnico.md)

[Voltar à audiência](README.md) | [Índice da wiki](../README.md)
