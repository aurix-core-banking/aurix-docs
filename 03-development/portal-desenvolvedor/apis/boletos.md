# API Boletos e Debito automatico

Dominio no **Core** (via Gateway: `POST/GET http://localhost:8080/api/core/...`).

## Boletos

| Metodo | Endpoint | Descricao |
|--------|----------|-----------|
| POST | /api/core/boletos/emitir | Emitir boleto (numero, linha digitavel, vencimento). Parametros: contaIdPagador (opc), beneficiarioNome, beneficiarioDocumento, pagadorNome, pagadorDocumento, valor, dataVencimento, descricao, usarProvedorExterno |
| POST | /api/core/boletos/{id}/registrar-pagamento | Registrar pagamento (baixa em conta + status PAGO). Parametro: contaIdPagador |
| GET | /api/core/boletos/{id} | Buscar boleto por ID |
| GET | /api/core/boletos/numero/{numeroBoleto} | Buscar por numero |
| GET | /api/core/boletos/conta/{contaIdPagador} | Listar boletos da conta |
| GET | /api/core/boletos/status/{status} | Listar por status (PENDENTE, PAGO, CANCELADO, VENCIDO) |

## Debito automatico

| Metodo | Endpoint | Descricao |
|--------|----------|-----------|
| POST | /api/core/agendamentos-debito | Agendar debito. Parametros: contaId, valor, dataDebito, descricao (opc), boletoId (opc), recorrente (opc), periodicidade (opc) |
| POST | /api/core/agendamentos-debito/{id}/cancelar | Cancelar agendamento |
| GET | /api/core/agendamentos-debito/conta/{contaId} | Listar agendamentos da conta |
| GET | /api/core/agendamentos-debito/pendentes | Listar agendamentos pendentes |

Quando `boletoId` e informado no agendamento, na data do debito o job executa o pagamento do boleto (baixa em conta). Caso contrario, e feito um debito generico na conta (tipo OUTROS). Job de execucao roda diariamente (6h).
