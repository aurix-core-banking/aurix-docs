# Sandbox e credenciais de teste

Ambiente com dados de teste para desenvolvimento e integracao sem afetar producao.

## Como subir o ambiente

1. Banco: PostgreSQL rodando (ex.: Docker ou local) com schema `aureus` e usuario configurado nos `application.yml` de cada modulo.
2. Subir os servicos na ordem: core (8081), pix (8082), onboarding (8095), gateway (8080). Ou use o Docker Compose da infraestrutura quando disponivel.
3. Tenant padrao: envie `X-Tenant-Id: default` em todas as requisicoes.

## Dados de teste

- **Tenant**: `default`
- **CPF de teste**: use CPFs validos mas nao reais (ex.: gerados por ferramentas de teste). O modulo core valida formato 11 digitos.
- **Cliente/Conta**: crie via API (ver [Primeira conta](./guias/primeira-conta.md)); nao ha seed obrigatorio. Opcional: scripts de seed em `docs/` ou em cada modulo para popular cliente e conta iniciais.

## API Key (sandbox)

O gateway suporta API Key e rate limit por plano. Para habilitar:

- **Header**: `X-Api-Key: <chave>` em todas as requisicoes (quando `api-key.required: true`).
- **Chave de sandbox** (pre-configurada no gateway): `sandbox-key-aureus-demo` — plano `sandbox`, tenant `default`, 30 req/min.

Configuracao no gateway (`aureus-gateway/src/main/resources/application.yml`):
```yaml
aureus:
  gateway:
    api-key:
      enabled: true
      required: false
      keys:
        sandbox-key-aureus-demo:
          plan: sandbox
          tenantId: default
      plan-limits:
        free: 10
        sandbox: 30
        starter: 60
        growth: 300
        enterprise: 1000
```

Com `enabled: true`, o gateway valida a chave (se enviada) e aplica o plano; com `required: true`, a ausencia de chave retorna 401. Rate limit usa Redis; sem Redis, o filtro de rate limit nao e aplicado.

## OAuth2 (sandbox)

Para fluxo Open Finance / OAuth2, use o modulo aureus-openfinance. Em sandbox, um cliente OAuth2 de teste pode ser configurado (client_id e client_secret) para obter token e chamar APIs em nome do usuario. Ver documentacao do modulo openfinance.

## Rate limit por plano

| Plano      | Requisicoes/minuto | Uso |
|------------|--------------------|-----|
| free       | 10                 | Sem API Key ou chave nao mapeada |
| sandbox    | 30                 | Chave de sandbox |
| starter    | 60                 | Desenvolvimento / pequeno volume |
| growth     | 300                | Producao medio |
| enterprise | 1000               | Producao alto (configuravel) |

O plano e derivado da API Key (mapeamento em `aureus.gateway.api-key.keys`). Rate limit e aplicado por plano + chave em janela de 1 minuto; exceder retorna 429.

## Resumo

- Use `X-Tenant-Id: default` e, quando disponivel, `X-Api-Key` com chave de sandbox.
- Crie cliente e conta pela API; opcionalmente use seeds.
- Consulte [Portal do Desenvolvedor](./README.md) e guias [Primeira conta](./guias/primeira-conta.md) e [Primeiro PIX](./guias/primeiro-pix.md).
