# Fluxo de Empréstimo

Diagrama de sequência do fluxo completo de concessão de crédito, desde a solicitação até o desembolso.

```mermaid
sequenceDiagram
    autonumber
    participant C as Cliente
    participant GW as API Gateway
    participant CR as svc-credit
    participant FR as svc-fraud
    participant ML as Motor ML
    participant BK as svc-banking
    participant K as Kafka

    C->>GW: POST /api/credit/solicitacoes
    Note over C,GW: Tipo, valor, prazo, renda comprovada

    GW->>CR: Criar solicitação de crédito
    CR->>CR: Validar elegibilidade do produto

    CR->>FR: Consultar score de fraude
    FR->>K: Consumir evento TransacaoAutorizadaEvent
    FR->>ML: Solicitar predição de fraude
    ML-->>FR: Score: 0.05 (baixo risco)
    FR-->>CR: Aprovado (score OK)

    alt Fraude detectada
        FR-->>CR: Rejeitado (score alto)
        CR-->>GW: 403 - Crédito negado por fraude
        GW-->>C: Solicitação rejeitada
    end

    CR->>ML: Solicitar scoring de crédito
    ML->>ML: Avaliar renda, histórico, inadimplência
    ML-->>CR: Aprovado — limite R$ 15.000, taxa 1,99% a.m.

    CR->>CR: Gerar contrato
    CR->>K: Publicar SolicitacaoCreditoCriadaEvent
    Note over CR,K: Topic: aurix.credit.solicitacao-criada

    CR-->>GW: 201 - Contrato gerado
    GW-->>C: Contrato disponível para assinatura

    C->>GW: POST /api/credit/contratos/{id}/assinar
    GW->>CR: Assinar contrato digitalmente
    CR->>CR: Validar assinatura (ICP-Brasil)

    CR->>K: Publicar ContratoAssinadoEvent
    Note over CR,K: Topic: aurix.credit.contrato-assinado

    CR->>BK: Solicitar desembolso
    BK->>BK: Creditar conta do cliente
    BK->>K: Publicar ContaEvent(CONTA_ATUALIZADA)
    BK-->>CR: Desembolso confirmado

    CR-->>GW: 200 - Empréstimo contratado e desembolsado
    GW-->>C: Valor creditado na conta
```

## Atividades

| Ator | Descrição |
|------|-----------|
| **Cliente** | Solicita crédito, assina contrato digital |
| **svc-credit** | Orquestração: elegibilidade, contrato, desembolso |
| **svc-fraud** | Análise de fraude anti-lavagem (AML) |
| **Motor ML** | Scoring de crédito e predição de default |
| **svc-banking** | Movimentação financeira — desembolso na conta |
| **Kafka** | Eventos assíncronos entre serviços |

## Cenários

1. **Renda insuficiente** → Rejeição antes do scoring de fraude
2. **Score ML baixo** → Proposta com condições diferenciadas (taxa maior)
3. **Garantia necessária** → Fluxo auxiliar: registro de garantia em cartório
4. **Consigado** → Margem verificada junto ao empregador antes da aprovação
