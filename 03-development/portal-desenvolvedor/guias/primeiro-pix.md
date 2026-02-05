# Guia: Primeiro PIX em poucos passos

Objetivo: ter uma conta, cadastrar uma chave PIX e realizar uma transferencia PIX (ou simular). Tempo estimado: 10 minutos.

## Pre-requisitos

- Uma conta ativa (se nao tiver, siga o [Primeira conta](./primeira-conta.md)).
- Modulo PIX rodando e acessivel (ex.: `http://localhost:8080/api/pix` via gateway).
- Header `X-Tenant-Id: default`.

## Passo 1: Obter ID da conta

Se ainda nao tiver, crie cliente e conta pelo guia Primeira conta. Anote o `id` da conta (ex.: 1).

## Passo 2: Cadastrar chave PIX

Consulte a documentacao do modulo PIX (Swagger em `http://localhost:8082/api/pix/swagger-ui.html` ou equivalente) para o endpoint exato de cadastro de chave. Tipicamente:

**Request** (exemplo generico):
```http
POST http://localhost:8080/api/pix/chaves
Content-Type: application/json
X-Tenant-Id: default

{
  "contaId": 1,
  "tipoChave": "CPF",
  "valorChave": "12345678901"
}
```

Ou tipo EMAIL, TELEFONE, ALEATORIA conforme implementacao do modulo. Ajuste o body conforme o contrato do `PixChaveController`.

**Resposta esperada**: chave cadastrada com id e valor (chave exibida ou mascarada). Anote o identificador da chave para uso em transferencias.

## Passo 3: Realizar transferencia PIX

**Request** (exemplo generico):
```http
POST http://localhost:8080/api/pix/transferencias
Content-Type: application/json
X-Tenant-Id: default

{
  "contaOrigemId": 1,
  "chaveDestino": "destino@email.com",
  "tipoChaveDestino": "EMAIL",
  "valor": 10.00,
  "descricao": "Teste PIX"
}
```

Campos podem variar (contaId, chavePixDestino, valor, idExterno, etc.). Consulte o Swagger do PIX.

**Resposta esperada**: transferencia criada com status (ex.: PENDENTE, PROCESSADA), endToEndId quando disponivel.

## Passo 4: Consultar transferencia

**Request** (exemplo):
```http
GET http://localhost:8080/api/pix/transferencias/{id}
X-Tenant-Id: default
```

Ou consulta por end-to-end id, conforme API do modulo.

## Observacoes

- Em ambiente local, a transferencia pode ser processada de forma sincrona ou assincrona; o modulo PIX pode integrar com SPI em homologacao/producao.
- Para testes sem debitar de verdade, use contas e valores de teste ou ambiente sandbox (ver [Sandbox](../sandbox.md)).
