# API Transacoes

Criacao e consulta de transacoes. Base URL do Core: `/api/core`. O contexto do core expoe transacoes em `/api/transacoes` (verificar context-path do modulo core).

## Base path

Se o core estiver com context-path `/api/core`, os endpoints de transacao podem estar em `/api/core/api/transacoes` ou em um controller com RequestMapping relativo. Consulte o Swagger do core: `http://localhost:8081/api/core/swagger-ui.html`.

## Endpoints (referencia)

### Criar transacao

`POST /api/core/api/transacoes` (ou path indicado no Swagger)

**Request**:
```json
{
  "contaOrigemId": 1,
  "contaDestinoId": 2,
  "tipoTransacao": "PIX",
  "valor": 100.50,
  "descricao": "Pagamento",
  "dadosPix": null,
  "dadosTed": null,
  "dataTransacao": "2025-02-03T10:00:00"
}
```

**Tipos**: PIX, TED, DOC, SAQUE, DEPOSITO, TRANSFERENCIA_INTERNA, PAGAMENTO_BOLETO, PAGAMENTO_CARTAO.

**Response** `200 OK`: objeto transacao com id, codigoTransacao, status (PENDENTE), etc.

**Erros**: 400 (conta inexistente), 422 (saldo insuficiente se aplicavel).

---

### Buscar transacao por ID

`GET /api/core/api/transacoes/{id}`

**Response** `200 OK`: transacao.

**Erros**: 404.

---

### Buscar por codigo

`GET /api/core/api/transacoes/codigo/{codigoTransacao}`

**Response** `200 OK**: transacao.

---

### Listar por conta

`GET /api/core/api/transacoes/conta/{contaId}`

**Response** `200 OK`: array de transacoes da conta (origem ou destino).

---

### Listar pendentes

`GET /api/core/api/transacoes/pendentes`

**Response** `200 OK`: array de transacoes com status PENDENTE.
