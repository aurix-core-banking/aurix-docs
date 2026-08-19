# Fluxo de Pagamento PIX

Diagrama de sequência do fluxo completo de uma transferência PIX, desde a criação até a liquidação via SPI do BACEN.

```mermaid
sequenceDiagram
    autonumber
    participant C as Cliente
    participant GW as API Gateway
    participant SP as svc-payments
    participant BK as svc-banking
    participant K as Kafka
    participant SPI as BACEN SPI

    C->>GW: POST /api/pix/transferencias
    Note over C,GW: Chave PIX, valor, idempotência

    GW->>SP: Criar transferência PIX
    SP->>SP: Validar dados (chave, saldo, limites)

    alt Saldo insuficiente
        SP-->>GW: 400 - Saldo insuficiente
        GW-->>C: Erro: saldo insuficiente
    end

    SP->>BK: Debitar conta origem
    BK->>BK: Débito com lock otimista
    BK-->>SP: Débito confirmado

    SP->>K: Publicar TransacaoAutorizadaEvent
    Note over SP,K: Topic: aurix.payments.pix-autorizado

    SP->>SPI: POST /pix/spi (liquidação)
    Note over SP,SPI: endToEndId, valor, chave, ISPB

    SPI-->>SP: 201 - Líquidação processada
    Note over SPI: endToEndId gerado pelo BACEN

    SP->>BK: Creditar conta destino
    BK->>BK: Crédito na conta

    BK-->>SP: Crédito confirmado

    SP->>K: Publicar PixLiquidadoEvent
    Note over SP,K: Topic: aurix.payments.pix-liquidado

    SP-->>GW: 201 - Transferência criada
    GW-->>C: PIX processado com sucesso
    Note over C: ID, endToEndId, status
```

## Atividades

| Ator | Descrição |
|------|-----------|
| **Cliente** | Usuário final que inicia a transferência PIX |
| **API Gateway** | Traefik — roteamento, autenticação, rate limiting |
| **svc-payments** | Serviço de pagamentos — criação, validação e orquestração |
| **svc-banking** | Serviço core — débito/crédito em contas |
| **Kafka** | Broker de mensagens — eventos assíncronos |
| **BACEN SPI** | Sistema de Pagamentos Instantâneos do Banco Central |

## Cenários de Erro

1. **Saldo insuficiente** → Rejeição antes da chamada SPI
2. **Timeout SPI** → Retry com idempotência, estado PENDENTE
3. **Erro BACEN** → Estorno automático do débito, evento de falha
4. **Chave PIX inválida** → Validação via `/pix/chave/validar` antes do envio
