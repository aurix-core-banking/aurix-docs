# API Credito

Solicitacoes de credito, produtos, simulador, decisao e workflow. Base URL: `http://localhost:8083/api/credit`. Via gateway: `http://localhost:8080/api/credit`.

## Headers

- `X-Tenant-Id`: tenant
- `Content-Type: application/json`

## Produtos de credito

| Metodo | Endpoint | Descricao |
|--------|----------|-----------|
| GET | /produtos | Listar produtos ativos |
| GET | /produtos/{id} | Buscar por ID |
| GET | /produtos/codigo/{codigo} | Buscar por codigo |
| GET | /produtos/tipo/{tipo} | Listar por tipo (PESSOAL, CONSIGNADO, CDC, VEICULOS, etc.) |

## Solicitacoes

| Metodo | Endpoint | Descricao |
|--------|----------|-----------|
| POST | /solicitacoes | Criar solicitacao (clienteId, valorSolicitado, prazoMeses, taxaJuros, produtoCreditoId opc) |
| GET | /solicitacoes/{id} | Buscar por ID |
| GET | /solicitacoes/cliente/{clienteId} | Listar por cliente |
| GET | /solicitacoes/status/{status} | Listar por status |
| GET | /solicitacoes/pendentes | Listar pendentes |
| GET | /solicitacoes/refer | Listar refer (analise manual) |
| PUT | /solicitacoes/{id}/aprovar | Aprovar (valorAprovado, prazoAprovado, taxaAprovada) |
| PUT | /solicitacoes/{id}/rejeitar | Rejeitar (observacoes) |
| PUT | /solicitacoes/{id}/emitir-oferta | Emitir oferta (apos aprovacao) |
| PUT | /solicitacoes/{id}/aceitar-oferta | Cliente aceita oferta |
| PUT | /solicitacoes/{id}/registrar-contrato | Registra contrato (contratoUrl) |
| PUT | /solicitacoes/{id}/liberar | Liberar credito |

## Simulador

| Metodo | Endpoint | Descricao |
|--------|----------|-----------|
| GET | /simulador?valor=&prazoMeses=&taxaJurosAoMes= | Simular parcelas, valor total, CET |

## Decisao

| Metodo | Endpoint | Descricao |
|--------|----------|-----------|
| POST | /decisao/{solicitacaoId} | Retorna APPROVE / DECLINE / REFER, score e motivo. Consulta bureau (stub) e regras (aurix.credit.regras). |

Regras (config): score-min-aprovar (600), score-max-rejeitar (400). Entre os dois = REFER para analista.
