# Portal do Desenvolvedor AUREUS

Documentacao de APIs e guias para integracao com a plataforma AUREUS Core Banking.

## Base URL

- **Gateway (recomendado)**: `http://localhost:8080`
- **Core**: `http://localhost:8081/api/core`
- **PIX**: `http://localhost:8082/api/pix`
- **Onboarding**: `http://localhost:8095/api/onboarding`

## Autenticacao

- **Tenant**: Enviar header `X-Tenant-Id` em todas as requisicoes (ex.: `default` ou identificador da instituicao).
- **API Key** (quando habilitado): Header `X-Api-Key` com chave do parceiro.
- **OAuth2**: Para integracoes Open Finance; ver modulo aureus-openfinance.

## Documentacao por dominio

| Dominio | Documento | Principais endpoints |
|---------|-----------|----------------------|
| [Clientes](./apis/clientes.md) | clientes.md | POST/GET clientes |
| [Contas](./apis/contas.md) | contas.md | POST/GET contas, listar por cliente |
| [Transacoes](./apis/transacoes.md) | transacoes.md | Criar, listar por conta |
| [Onboarding](./apis/onboarding.md) | onboarding.md | Solicitar conta, KYC, aprovar |
| [PIX](./apis/pix.md) | pix.md | Chaves, transferencias |
| [Credito](./apis/credito.md) | credito.md | Solicitacoes, analise |
| [Boletos e debito automatico](./apis/boletos.md) | boletos.md | Emitir boleto, registrar pagamento, agendar debito |

## Guias passo a passo

- [Primeira conta](./guias/primeira-conta.md) - Do zero ate uma conta ativa
- [Primeiro PIX](./guias/primeiro-pix.md) - Cadastrar chave e fazer uma transferencia PIX

## Sandbox

- [Sandbox e credenciais de teste](./sandbox.md) - Ambiente com dados de teste e como obter API Key para sandbox

## Codigos de erro comuns

| HTTP | Significado |
|------|-------------|
| 400 | Requisicao invalida (validacao de campo, formato) |
| 401 | Nao autorizado (token ou API Key invalido) |
| 403 | Sem permissao para o recurso |
| 404 | Recurso nao encontrado (id ou numero inexistente) |
| 409 | Conflito (ex.: CPF ou numero de conta ja existente) |
| 422 | Regra de negocio (ex.: saldo insuficiente) |
| 500 | Erro interno do servidor |

Respostas de erro padrao incluem `message` e opcionalmente `errors` (lista de validacao).
