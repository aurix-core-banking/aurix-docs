# API Onboarding

Abertura de conta com fluxo de solicitacao, documentos, KYC e aprovacao. Base URL: `/api/onboarding`.

## Headers

- `Content-Type: application/json`
- `X-Tenant-Id`: obrigatorio (ex.: `default`)

## Endpoints

### Solicitar abertura de conta

`POST /api/onboarding/solicitacoes`

**Request**:
```json
{
  "cpf": "12345678901",
  "nome": "Fulano da Silva",
  "email": "fulano@email.com",
  "telefone": "11999999999",
  "dataNascimento": "1990-01-15",
  "ocupacao": "Empregado",
  "rendaDeclarada": 5000,
  "endereco": "{\"logradouro\":\"Rua X\",\"numero\":\"100\"}"
}
```

**Response** `200 OK`: objeto solicitacao com id, status (RECEBIDA), scoreBureau, pep, etc.

**Erros**: 400 (CPF ja com solicitacao em andamento), 400 (validacao).

---

### Consultar status

`GET /api/onboarding/solicitacoes/{id}`

**Response** `200 OK`: solicitacao com status, documentos, historico.

**Erros**: 404.

---

### Listar solicitacoes (back office)

`GET /api/onboarding/solicitacoes?status=RECEBIDA&status=EM_ANALISE_KYC`

**Query params**: `status` (opcional, multiplos) - RECEBIDA, DOCUMENTOS_PENDENTES, EM_ANALISE_KYC, KYC_APROVADO, KYC_REJEITADO, EM_ANALISE_MANUAL, APROVADA, REJEITADA, CONTA_CRIADA.

**Response** `200 OK`: array de solicitacoes.

---

### Aprovar solicitacao

`POST /api/onboarding/solicitacoes/{id}/aprovar?usuarioAnalista=analista1&observacao=Ok`

Cria cliente e conta no core e atualiza status para CONTA_CRIADA.

**Response** `200 OK`: solicitacao com clienteIdCriado, contaIdCriada.

**Erros**: 404, 500 (falha ao criar no core).

---

### Rejeitar solicitacao

`POST /api/onboarding/solicitacoes/{id}/rejeitar?usuarioAnalista=analista1&observacao=Documentos inconsistentes`

**Response** `200 OK`: solicitacao com status REJEITADA.

---

### Enviar para KYC

`POST /api/onboarding/solicitacoes/{id}/kyc`

**Request**:
```json
{
  "documentos": [
    { "tipo": "RG", "urlOuBase64": "base64...", "nomeArquivo": "rg.jpg" },
    { "tipo": "CPF", "urlOuBase64": "base64...", "nomeArquivo": "cpf.jpg" }
  ],
  "selfieBase64": "data:image/jpeg;base64,..."
}
```

**Response** `200 OK`: solicitacao com status KYC_APROVADO ou KYC_REJEITADO e resultadoKyc.

---

### Adicionar documento

`POST /api/onboarding/solicitacoes/{id}/documentos`

**Request**:
```json
{
  "tipoDocumento": "RG_FRENTE",
  "nomeArquivo": "rg_frente.pdf",
  "urlStorage": "https://bucket.s3.../doc123"
}
```

**Response** `204 No Content`.

---

### Consultar PEP

`GET /api/onboarding/pep/{cpf}`

**Response** `200 OK`: `{ "pep": true }` ou `{ "pep": false }`.

---

### Registrar PEP

`POST /api/onboarding/pep`

**Request**:
```json
{
  "cpf": "12345678901",
  "nome": "Fulano",
  "cargoOuVinculo": "Vereador"
}
```

**Response** `200 OK`: entidade PEP.
