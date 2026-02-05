# API PIX

Chaves PIX e transferencias. Base URL do modulo PIX: `http://localhost:8082/api/pix`. Via gateway: `http://localhost:8080/api/pix`.

## Headers

- `X-Tenant-Id`: tenant (opcional; padrao e `default`; obrigatorio apenas em deploy multi-tenant)
- `Content-Type: application/json`

## Endpoints (referencia)

Consulte o Swagger do modulo PIX: `http://localhost:8082/api/pix/swagger-ui.html` para a lista exata de paths.

### Chaves PIX

- **Cadastrar chave**: POST ao endpoint de chaves (chave tipo CPF, email, telefone ou aleatoria).
- **Listar chaves por conta**: GET com identificacao da conta.
- **Remover chave**: DELETE por id da chave.

### Transferencias PIX

- **Iniciar transferencia**: POST com conta origem, chave destino (ou conta destino), valor, identificador externo.
- **Consultar transferencia**: GET por id ou end-to-end id.

## Exemplo de fluxo

1. Obter conta (API Contas no core).
2. Cadastrar chave PIX para a conta (API PIX).
3. Realizar transferencia informando valor e chave destino (API PIX).

Ver [Primeiro PIX](../guias/primeiro-pix.md) para passo a passo completo.
