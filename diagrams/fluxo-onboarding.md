# Fluxo de Onboarding

Diagrama de sequência do cadastro de novos clientes (PF/PJ), incluindo validação KYC e criação de conta.

```mermaid
sequenceDiagram
    autonumber
    participant C as Cliente
    participant GW as API Gateway
    participant CU as svc-customer
    participant RF as ReceitaFederal
    participant CS as ClearSale
    participant BK as svc-banking
    participant K as Kafka
    participant NO as Notificações

    C->>GW: POST /api/customer/clientes
    Note over C,GW: Dados pessoais, documento, endereço

    GW->>CU: Criar cadastro PF/PJ
    CU->>CU: Validar campos obrigatórios

    CU->>RF: Consultar CPF/CNPJ
    Note over CU,RF: GET /receita-federal/{documento}
    RF-->>CU: Dados confirmados (nome, situação, CNPJ)

    alt Documento inválido
        RF-->>CU: 404 - CPF/CNPJ não encontrado
        CU-->>GW: 400 - Documento inválido
        GW-->>C: Erro: documento não encontrado
    end

    CU->>CS: Consulta ClearSale (score antifraude)
    Note over CU,CS: CPF, nome, data nascimento
    CS-->>CS: Análise de identidade
    CS-->>CU: Score: 850 (baixo risco)

    alt Score alto (suspeita)
        CS-->>CU: Score: 200 (alto risco)
        CU->>CU: Marcar para análise manual
        CU-->>GW: 202 - Cadastro em análise
        GW-->>C: Aguardando aprovação manual
    end

    CU->>CU: Aprovar cadastro KYC

    CU->>K: Publicar ClienteCriadoEvent
    Note over CU,K: Topic: aurix.customer.cliente-criado

    CU->>BK: Solicitar criação de conta
    Note over CU,BK: clienteId, tipoConta, segmento

    BK->>BK: Gerar agência e número da conta
    BK->>BK: Criar conta com saldo zero
    BK->>K: Publicar ContaEvent(CONTA_CRIADA)
    Note over BK,K: Topic: aurix.banking.conta-criada

    BK-->>CU: Conta criada
    CU-->>GW: 201 - Cliente e conta criados
    GW-->>C: Cadastro completo

    CU->>NO: Enviar boas-vindas
    Note over CU,NO: E-mail + SMS com dados da conta
    NO-->>C: Notificação de boas-vindas
```

## Atividades

| Ator | Descrição |
|------|-----------|
| **Cliente** | Usuário que realiza o cadastro inicial |
| **API Gateway** | Traefik — roteamento e autenticação |
| **svc-customer** | Onboarding: cadastro, KYC, validação de documentos |
| **ReceitaFederal** | Validação de CPF/CNPJ junto à Receita |
| **ClearSale** | Score antifraude para validação de identidade |
| **svc-banking** | Criação da conta bancária |
| **Kafka** | Eventos de cliente criado e conta criada |
| **Notificações** | Envio de e-mail/SMS de boas-vindas |

## Cenários

1. **Pessoa Jurídica** → Fluxo adicional: consulta CNPJ, sócio, capital social
2. **Estrangeiro** → Validação de RNE/CRNM em vez de CPF
3. **Rejeição KYC** → Cadastro fica pendente; cliente pode reenviar documentos
4. **Regra LGPD** → Consentimento explícito antes do processamento dos dados
