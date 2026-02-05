# API Contas

Gestao de contas bancarias. Base URL do Core: `/api/core`.

## Endpoints

### Criar conta

`POST /api/core/contas`

**Headers**: `Content-Type: application/json`, `X-Tenant-Id: <tenant>`

**Request**:
```json
{
  "clienteId": 1,
  "tipoConta": "CORRENTE",
  "saldo": 0,
  "limiteCredito": 1000,
  "dadosExtras": null
}
```

**Tipos**: `CORRENTE`, `POUPANCA`, `SALARIO`.

**Response** `201 Created`:
```json
{
  "id": 1,
  "numeroConta": "12345-6",
  "clienteId": 1,
  "clienteNome": "Fulano da Silva",
  "tipoConta": "CORRENTE",
  "saldo": 0,
  "limiteCredito": 1000,
  "limiteUtilizado": 0,
  "limiteDisponivel": 1000,
  "status": "ATIVA",
  "dataAbertura": "2025-02-03T10:00:00",
  "dataFechamento": null,
  "dadosExtras": null,
  "dataCriacao": "...",
  "dataAtualizacao": "..."
}
```

**Erros**: 400 (clienteId inexistente), 404 (cliente nao encontrado no tenant).

---

### Buscar conta por ID

`GET /api/core/contas/{id}`

**Response** `200 OK`: objeto conta.

**Erros**: 404.

---

### Buscar conta por numero

`GET /api/core/contas/numero/{numeroConta}`

Formato do numero: `12345-6` (5 digitos, hifen, 1 digito).

**Response** `200 OK**: objeto conta.

**Erros**: 404, 400 (formato invalido).

---

### Listar contas por cliente

`GET /api/core/contas/cliente/{clienteId}`

**Response** `200 OK`: array de contas do cliente.

---

### Listar contas ativas por cliente

`GET /api/core/contas/cliente/{clienteId}/ativas`

**Response** `200 OK`: array de contas com status ATIVA.

---

### Listar todas as contas (tenant)

`GET /api/core/contas`

**Response** `200 OK`: array de contas do tenant.

---

### Atualizar conta

`PUT /api/core/contas/{id}`

**Request**: campos editaveis (limiteCredito, dadosExtras).

**Response** `200 OK`: conta atualizada.

**Erros**: 404.
