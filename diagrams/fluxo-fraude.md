# Fluxo de Detecção de Fraude

Diagrama de sequência do fluxo de análise antifraude e AML (Anti-Money Laundering) em tempo real, desde a transação até a decisão de bloqueio ou aprovação.

```mermaid
sequenceDiagram
    autonumber
    participant SP as svc-payments
    participant K as Kafka
    participant FR as svc-fraud
    participant ML as Motor ML
    participant CO as svc-compliance
    participant BK as svc-banking
    participant NO as Notificações
    participant C as Cliente

    SP->>K: Publicar TransacaoAutorizadaEvent
    Note over SP,K: Topic: aurix.banking.transacao-autorizada

    K->>FR: Consumir evento
    Note over FR: Consumer: fraud-analysis-consumer

    FR->>FR: Regras determinísticas (AML)

    Note over FR: Regra 1: Valor > R$ 10.000 → alerta
    Note over FR: Regra 2: País de alto risco → bloqueio
    Note over FR: Regra 3: Frequência anômala → alerta

    alt Regra AML violada
        FR->>CO: Enviar para análise compliance
        CO->>CO: Verificar listas PEP/OFAC
        CO-->>FR: Restrição confirmada
        FR->>K: Publicar OcorrenciaFraudEvent
        Note over FR,K: Topic: aurix.fraud.ocorrencia-fraud
        FR->>BK: Bloquear conta temporariamente
        BK->>BK: Conta → status BLOQUEADA
        BK->>K: Publicar ContaEvent(CONTA_BLOQUEADA)
        FR->>NO: Notificar cliente (SMS + e-mail)
        NO-->>C: Conta bloqueada — entre em contato
    end

    FR->>ML: Solicitar predição de fraude
    Note over FR,ML: Features: valor, hora, geolocalização,<br/>histórico, dispositivo

    ML->>ML: XGBoost inference
    ML->>ML: SHAP: fatores contribuintes
    ML-->>FR: Score: 0.87 (87% probabilidade de fraude)

    alt Score > 0.8 (alto risco)
        FR->>FR: Decisão: BLOQUEAR
        FR->>K: Publicar OcorrenciaFraudEvent
        FR->>BK: Bloquear transação
        BK->>BK: Estornar débito
        BK-->>FR: Estorno confirmado
        FR->>NO: Notificar cliente
        NO-->>C: Transação bloqueada — confirme sua identidade
    else Score 0.3–0.8 (risco médio)
        FR->>FR: Decisão: REVISÃO_MANUAL
        FR->>K: Publicar OcorrenciaFraudEvent
        FR->>NO: Enviar para fila de análise
    else Score < 0.3 (baixo risco)
        FR->>FR: Decisão: APROVAR
        Note over FR: Transação liberada sem intervenção
    end

    FR->>K: Publicar ScoreAlteradoEvent
    Note over FR,K: Topic: aurix.fraud.score-alterado
```

## Atividades

| Ator | Descrição |
|------|-----------|
| **svc-payments** | Origem: transação autorizada |
| **Kafka** | Transporte de eventos entre serviços |
| **svc-fraud** | Orquestração: regras AML + integração com ML |
| **Motor ML** | Predição XGBoost com explicabilidade (SHAP) |
| **svc-compliance** | Listas PEP, OFAC, Sanções — AML regulatório |
| **svc-banking** | Execução: bloqueio, estorno, desbloqueio |
| **Notificações** | Alertas ao cliente (SMS, e-mail, push) |
| **Cliente** | Confirmação de identidade quando solicitado |

## Regras AML Implementadas

| Regra | Limite | Ação |
|-------|--------|------|
| Valor elevado | > R$ 10.000 | Alerta + revisão |
| País alto risco | Lista OFAC | Bloqueio imediato |
| Frequência anômala | > 10 transações/hora | Alerta |
| Horário atípico | 02:00–05:00 + valor alto | Alerta |
| Dispositivo novo | Primeira vez | Verificação 2FA |

## Decisões ML

| Score | Decisão | Ação |
|-------|---------|------|
| 0.0 – 0.3 | APROVAR | Sem intervenção |
| 0.3 – 0.8 | REVISÃO_MANUAL | Fila de análise |
| 0.8 – 1.0 | BLOQUEAR | Estorno + notificação |
