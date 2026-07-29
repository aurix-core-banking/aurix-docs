# Scripts de seed – Dados de exemplo por tenant

Documentação dos scripts e mecanismos de seed para produtos de crédito, tarifas, parâmetros e dados de exemplo por tenant. Item 13.2 do roadmap.

---

## Onde estão os seeds

- **Planos (billing)**: módulo `aurix-billing` possui `PlanoDataLoader` que, na primeira subida (`aurix.billing.seed-planos: true`), insere os planos STARTER, GROWTH e ENTERPRISE. Desativar em produção se os planos forem criados manualmente ou por script próprio.
- **Produtos de crédito**: módulo `aurix-credit` utiliza entidade `ProdutoCredito`; não há seed padrão. Os scripts abaixo ou a API de administração devem criar os produtos por tenant.
- **Tarifas**: módulo `aurix-pricing` possui entidades de tarifa e pacotes; a carga inicial pode ser feita via API ou script SQL/import.

---

## Scripts e exemplos

### 1. Produtos de crédito (por tenant)

Criar produtos iniciais após o provisioning do tenant. Exemplo via API (ou adaptar para script):

```http
POST /api/credit/produtos
X-Tenant-Id: {tenant_id}
Content-Type: application/json

{
  "codigo": "EMPRESTIMO_PESSOAL",
  "nome": "Empréstimo Pessoal",
  "taxaMensal": 2.5,
  "prazoMinMeses": 6,
  "prazoMaxMeses": 48,
  "valorMin": 1000,
  "valorMax": 50000,
  "ativo": true
}
```

Repetir para outros produtos (capital de giro, CDC, etc.) conforme catálogo da instituição.

### 2. Tarifas e parâmetros

- **Tarifas**: usar a API do módulo `aurix-pricing` para cadastrar tarifas por operação (PIX, TED, saque, etc.) ou importar via script que insere em `aurix.pacote_tarifas` e tabelas relacionadas.
- **Parâmetros gerais**: se existir tabela de parâmetros por tenant, popular com valores default (limites, prazos, flags) no momento do provisioning ou via script pós-provisioning.

### 3. Dados de exemplo (sandbox/homolog)

Para ambiente de teste ou sandbox, pode-se executar:

- Inserção de clientes e contas de exemplo (via API do core ou script que chama as APIs).
- Inserção de uma instituição e config de tenant no provisioning; em seguida, produtos de crédito e tarifas como acima.

Os guias "primeira conta" e "primeiro PIX" no portal do desenvolvedor usam dados de exemplo; em sandbox, o seed pode criar esses dados automaticamente.

---

## Automação no provisioning

O fluxo de provisioning (item 8.3) pode, após criar o banco e config do tenant, chamar um **serviço de seed** ou **job** que:

1. Insere planos (se billing compartilhado) ou associa o tenant ao plano contratado.
2. Insere produtos de crédito padrão (quando definidos no catálogo do produto).
3. Insere tarifas padrão e parâmetros iniciais.
4. Opcionalmente, em ambiente não produtivo, insere clientes e contas de exemplo.

A implementação desse passo pode ser um endpoint interno (ex.: `POST /api/provisioning/tenants/{id}/seed`) ou um script executado pelo pipeline de deploy após o provisioning.

---

## Referências

- [Portal do desenvolvedor – sandbox](../../03-development/portal-desenvolvedor/sandbox.md)
- Módulo aurix-billing: `PlanoDataLoader` e README
- Módulo aurix-credit: API de produtos de crédito
- Módulo aurix-pricing: API de tarifas

[Voltar ao kit](README.md) | [Índice da wiki](../../README.md)
