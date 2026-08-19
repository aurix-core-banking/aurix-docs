# Fluxo de TED

Diagrama de sequência do fluxo completo de uma transferência bancária (TED), desde a criação até a compensação via SPI/STR.

```mermaid
sequenceDiagram
    autonumber
    participant C as Cliente
    participant GW as API Gateway
    participant BK as svc-banking
    participant SP as svc-payments
    participant K as Kafka
    participant BACEN as BACEN (SPI/STR)
    participant BD as Banco Destino

    C->>GW: POST /api/banking/ted
    Note over C,GW: Banco destino, agência, conta, valor, CPF/CNPJ

    GW->>BK: Criar TED
    BK->>BK: Validar dados (banco, agência, conta, titular)

    BK->>BK: Debitar conta origem
    Note over BK: Lock otimista + saldo verificado

    alt Saldo insuficiente
        BK-->>GW: 400 - Saldo insuficiente
        GW-->>C: Erro: saldo insuficiente
    end

    BK->>K: Publicar TransacaoAutorizadaEvent
    Note over BK,K: Topic: aurix.banking.transacao-autorizada

    BK->>SP: Encaminhar TED para processamento

    alt Horário comercial SPB (09:00–17:00)
        SP->>SP: Processar imediatamente via SPI
        SP->>BACEN: POST /spi/ted (liquidação)
        Note over SP,BACEN: ISPB origem, ISPB destino, valor

        BACEN-->>SP: 201 - TED liquidado
        Note over BACEN: Compensação instantânea

        SP->>BD: Notificar crédito
        BD-->>SP: Confirmação de crédito
    else Fora do horário SPB
        SP->>SP: Enfileirar para processamento
        SP->>BACEN: POST /str/ted (compensação D+1)
        Note over SP,BACEN: Agendamento para próximo dia útil

        BACEN-->>SP: 201 - TED agendado
    end

    SP->>K: Publicar TransacaoLiquidadaEvent
    Note over SP,K: Topic: aurix.banking.transacao-liquidada

    SP-->>BK: TED processado
    BK-->>GW: 200 - TED confirmado
    GW-->>C: Transferência realizada com sucesso
    Note over C: ID, data de compensação
```

## Atividades

| Ator | Descrição |
|------|-----------|
| **Cliente** | Solicita a transferência TED |
| **API Gateway** | Traefik — autenticação e roteamento |
| **svc-banking** | Débito na conta, validação de dados bancários |
| **svc-payments** | Orquestração do envio ao BACEN |
| **Kafka** | Eventos de autorização e liquidação |
| **BACEN (SPI/STR)** | Sistema de Pagamentos Instantâneos ou STR (D+1) |
| **Banco Destino** | Instituição que recebe a transferência |

## Compensação

| Horário | Canal | Compensação |
|---------|-------|-------------|
| 09:00–17:00 (dia útil) | SPI | Instantânea (segundos) |
| Fora do horário | STR | D+1 (próximo dia útil) |
| Sábado/Domingo/feriado | STR | D+1 (próximo dia útil) |

## Cenários de Erro

1. **Conta destino inexistente** → BACEN rejeita, estorno automático
2. **ISPB inválido** → Validação prévia antes do envio
3. **Timeout SPI** → Retry com idempotência (max 3 tentativas)
4. **Valor acima do limite** → Exige autorização adicional (segundo fator)
