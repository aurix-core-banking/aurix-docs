# Guia: Primeira conta em poucos passos

Objetivo: criar um cliente e uma conta corrente via API, prontos para transacoes. Tempo estimado: 5 minutos.

## Pre-requisitos

- Gateway rodando em `http://localhost:8080` (ou core em `http://localhost:8081/api/core`).
- Header `X-Tenant-Id: default` em todas as requisicoes.

## Passo 1: Criar cliente

**Request**:
```http
POST http://localhost:8080/api/core/clientes
Content-Type: application/json
X-Tenant-Id: default

{
  "cpf": "12345678901",
  "nome": "Maria Silva",
  "email": "maria.silva@email.com",
  "telefone": "11987654321",
  "dataNascimento": "1985-06-20"
}
```

**Resposta esperada** `201 Created`: corpo com `id` (ex.: 1), cpf, nome, email, status ATIVO. Anote o `id` do cliente.

**Curl**:
```bash
curl -X POST http://localhost:8080/api/core/clientes \
  -H "Content-Type: application/json" \
  -H "X-Tenant-Id: default" \
  -d "{\"cpf\":\"12345678901\",\"nome\":\"Maria Silva\",\"email\":\"maria.silva@email.com\",\"telefone\":\"11987654321\",\"dataNascimento\":\"1985-06-20\"}"
```

## Passo 2: Criar conta para o cliente

Substitua `{clienteId}` pelo id retornado no passo 1 (ex.: 1).

**Request**:
```http
POST http://localhost:8080/api/core/contas
Content-Type: application/json
X-Tenant-Id: default

{
  "clienteId": 1,
  "tipoConta": "CORRENTE",
  "saldo": 0,
  "limiteCredito": 0
}
```

**Resposta esperada** `201 Created`: corpo com `id`, `numeroConta` (formato 12345-6), clienteId, status ATIVA, saldo 0. Anote o `id` e o `numeroConta` da conta.

**Curl**:
```bash
curl -X POST http://localhost:8080/api/core/contas \
  -H "Content-Type: application/json" \
  -H "X-Tenant-Id: default" \
  -d "{\"clienteId\":1,\"tipoConta\":\"CORRENTE\",\"saldo\":0,\"limiteCredito\":0}"
```

## Passo 3: Consultar conta

**Request**:
```http
GET http://localhost:8080/api/core/contas/1
X-Tenant-Id: default
```

**Resposta esperada** `200 OK`: dados da conta (numero, saldo, status, etc.).

## Pronto

Voce tem um cliente e uma conta ativa. Proximos passos: cadastrar chave PIX (ver [Primeiro PIX](./primeiro-pix.md)), criar transacoes ou usar o fluxo de onboarding para abertura de conta com KYC.

## Alternativa: via Onboarding

Para fluxo com documentos e aprovacao:

1. `POST /api/onboarding/solicitacoes` com dados do titular.
2. Adicionar documentos: `POST /api/onboarding/solicitacoes/{id}/documentos`.
3. (Opcional) Enviar para KYC: `POST /api/onboarding/solicitacoes/{id}/kyc`.
4. Back office aprova: `POST /api/onboarding/solicitacoes/{id}/aprovar` (cria cliente e conta no core).

Ver [API Onboarding](../apis/onboarding.md).
