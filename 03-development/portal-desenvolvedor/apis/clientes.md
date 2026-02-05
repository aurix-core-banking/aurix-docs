# API Clientes

Gestao de clientes (pessoas fisicas). Base URL do Core: `/api/core`.

## Endpoints

### Criar cliente

`POST /api/core/clientes`

**Headers**: `Content-Type: application/json`, `X-Tenant-Id: <tenant>`

**Request**:
```json
{
  "cpf": "12345678901",
  "nome": "Fulano da Silva",
  "email": "fulano@email.com",
  "telefone": "11999999999",
  "dataNascimento": "1990-01-15",
  "endereco": "{\"logradouro\":\"Rua X\",\"numero\":\"100\",\"cep\":\"01310100\"}"
}
```

**Response** `201 Created`:
```json
{
  "id": 1,
  "cpf": "12345678901",
  "nome": "Fulano da Silva",
  "email": "fulano@email.com",
  "telefone": "11999999999",
  "dataNascimento": "1990-01-15",
  "endereco": "{...}",
  "status": "ATIVO",
  "dataCriacao": "2025-02-03T10:00:00",
  "dataAtualizacao": "2025-02-03T10:00:00"
}
```

**Erros**: 400 (CPF/email ja existe ou invalido), 400 (validacao de campos).

---

### Buscar cliente por ID

`GET /api/core/clientes/{id}`

**Response** `200 OK`: mesmo formato do objeto cliente acima.

**Erros**: 404 se id nao existir no tenant.

---

### Buscar cliente por CPF

`GET /api/core/clientes/cpf/{cpf}`

**Response** `200 OK`: objeto cliente.

**Erros**: 404 se CPF nao existir no tenant.

---

### Listar clientes

`GET /api/core/clientes`

**Response** `200 OK`: array de clientes.

---

### Listar clientes ativos

`GET /api/core/clientes/ativos`

**Response** `200 OK`: array de clientes com status ATIVO.

---

### Atualizar cliente

`PUT /api/core/clientes/{id}`

**Request**: mesmos campos editaveis (nome, email, telefone, dataNascimento, endereco).

**Response** `200 OK`: cliente atualizado.

**Erros**: 404 (id), 400 (validacao).
