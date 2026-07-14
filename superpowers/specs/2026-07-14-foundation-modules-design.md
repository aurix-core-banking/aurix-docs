# Foundation Modules — Design Spec (Fase 1)

> **Objetivo:** Implementar 4 módulos de fundação para a plataforma AUREUS: Customer, KYC, Fraud, Notification.
> **Prioridade:** Valor de negócio (Adquirência, Cobrança, Garantias na Fase 2 dependem destes).
> **Arquitetura:** Microsserviços Spring Boot 4.1.0 + Kafka + PostgreSQL. Mesmo padrão dos 37 módulos existentes.

## Arquitetura Geral

```
[Customer] ──Kafka: cliente.criado──▶ [KYC] ──Kafka: kyc.aprovado──▶ [Notification]
    │                                        │                            │
    │                                        └──Kafka: kyc.aprovado──▶ [Fraud]
    │                                                                     │
    └──────────────────────Kafka: cliente.criado──────────────────────────▶ [Fraud]
                                                                           │
    [PIX] ──Kafka: pix.transferencia──▶ [Fraud]                            │
    [Cartões] ──Kafka: cartao.transacao──▶ [Fraud]                         │
                                                                            ▼
                                                                   Kafka: notificar
                                                                        │
                                                                        ▼
                                                                   [Notification]
```

- **Comunicação síncrona:** REST entre serviços (consultas pontuais)
- **Comunicação assíncrona:** Kafka para eventos de domínio
- **Cada módulo:** microsserviço independente com seu banco PostgreSQL (schema `aureus`)

## Módulos

### 1. aureus-customer

**Propósito:** Cadastro único de clientes (PF/PJ), perfis, segmentação, contatos, endereços.

**Porta:** 8123
**Context path:** `/api/customer`
**Kafka consumer group:** `aureus-customer-group`

#### Entidades

- `Cliente` — PF/PJ, nome, documento, segmento (PF/PJ/UNICO/PRIVATE/CORP), status (ATIVO/BLOQUEADO/INATIVO)
- `ClientePF` — CPF, RG, data_nascimento, estado_civil, nacionalidade, profissao, renda_mensal
- `ClientePJ` — CNPJ, razao_social, nome_fantasia, inscricao_estadual, capital_social, socios (JSON)
- `Endereco` — cliente_id, tipo (RESIDENCIAL/COMERCIAL/COBRANCA), logradouro, numero, complemento, bairro, cidade, uf, cep
- `Contato` — cliente_id, tipo (EMAIL/TELEFONE/CELULAR/WHATSAPP), valor, preferencial
- `ClienteRelacionamento` — cliente_id, gerente_id, carteira_id, data_abertura, score_relacionamento

#### REST API

| Método | Path | Descrição |
|--------|------|-----------|
| POST | `/api/clientes` | Criar cliente (PF ou PJ) |
| GET | `/api/clientes/{id}` | Buscar por ID |
| GET | `/api/clientes/documento/{doc}` | Buscar por CPF/CNPJ |
| PATCH | `/api/clientes/{id}` | Atualizar dados |
| GET | `/api/clientes` | Listar (filtros: segmento, status, nome) |
| POST | `/api/clientes/{id}/contatos` | Adicionar contato |
| GET | `/api/clientes/{id}/contatos` | Listar contatos |
| POST | `/api/clientes/{id}/enderecos` | Adicionar endereço |
| GET | `/api/clientes/{id}/enderecos` | Listar endereços |

#### Kafka Events (produz)

- `cliente.criado` — cliente cadastrado → KYC inicia validação
- `cliente.atualizado` — dados alterados
- `cliente.status.alterado` — ativo/bloqueado/inativo

### 2. aureus-kyc

**Propósito:** Validação documental, biometria, consulta PEP, score de risco do cliente.

**Porta:** 8124
**Context path:** `/api/kyc`
**Kafka consumer group:** `aureus-kyc-group`

#### Entidades

- `SolicitacaoKYC` — cliente_id, status (PENDENTE/EM_ANALISE/APROVADO/REJEITADO), score_risco, data_solicitacao, data_conclusao
- `DocumentoKYC` — solicitacao_id, tipo (RG/CNH/COMPROVANTE_ENDERECO/CONTRA_CHEQUE/CONT_SOCIAL), arquivo_ref, status, motivo_rejeicao
- `ScoreKYC` — cliente_id, score_geral, score_documento, score_biometria, score_pep, score_origem_fundos
- `PepsQuery` — cliente_id, nome_consulta, data_consulta, resultado_json, lista_origem

#### REST API

| Método | Path | Descrição |
|--------|------|-----------|
| POST | `/api/kyc/solicitacoes` | Iniciar KYC para cliente |
| GET | `/api/kyc/solicitacoes/{id}` | Status da solicitação |
| GET | `/api/kyc/solicitacoes/cliente/{id}` | Histórico KYC do cliente |
| POST | `/api/kyc/documentos` | Anexar documento |
| PATCH | `/api/kyc/documentos/{id}/status` | Atualizar status do documento |
| POST | `/api/kyc/solicitacoes/{id}/aprovar` | Aprovar manualmente |
| POST | `/api/kyc/solicitacoes/{id}/rejeitar` | Rejeitar (com motivo) |
| GET | `/api/kyc/score/{clienteId}` | Consultar score |
| GET | `/api/kyc/stats` | Estatísticas (pendentes, aprovados, tempo médio) |

#### Kafka Events

Consume: `cliente.criado` (inicia workflow)
Produz: `kyc.aprovado`, `kyc.rejeitado`, `kyc.documento.analise`

### 3. aureus-fraud

**Propósito:** Scoring de risco em tempo real, regras de fraude, monitoramento transacional.

**Porta:** 8125
**Context path:** `/api/fraud`
**Kafka consumer group:** `aureus-fraud-group`

#### Entidades

- `RegraFraude` — codigo, descricao, tipo (VELOCIDADE/VALOR/GEOGRAFIA/COMPORTAMENTO/SUSPEITO), parametros_json, prioridade, ativa
- `ScoreTransacao` — transacao_id, modulo_origem, cliente_id, score_geral, regras_disparadas (JSON), decisao (APROVAR/BLOQUEAR/REVISAR)
- `OcorrenciaFraude` — cliente_id, transacao_id, tipo, data, status (INVESTIGANDO/CONFIRMADA/FALSO_POSITIVO)
- `BloqueioPreventivo` — cliente_id, motivo, data_inicio, data_fim, responsavel, ativo

#### REST API

| Método | Path | Descrição |
|--------|------|-----------|
| POST | `/api/fraud/transacoes/avaliar` | Avaliar transação em tempo real (síncrono) |
| POST | `/api/fraud/regras` | Cadastrar regra |
| GET | `/api/fraud/regras` | Listar regras |
| PATCH | `/api/fraud/regras/{id}` | Ativar/desativar regra |
| GET | `/api/fraud/ocorrencias` | Consultar ocorrências (filtro: cliente, status) |
| POST | `/api/fraud/ocorrencias/{id}/confirmar` | Confirmar fraude |
| POST | `/api/fraud/ocorrencias/{id}/falso-positivo` | Marcar falso positivo |
| POST | `/api/fraud/bloqueios` | Bloquear cliente preventivamente |
| DELETE | `/api/fraud/bloqueios/{id}` | Remover bloqueio |
| GET | `/api/fraud/score/{clienteId}` | Score de risco do cliente |

#### Kafka Events

Consume: `cliente.criado`, `kyc.aprovado`, `pix.transferencia.criada`, `credito.solicitacao.criada`, `cartoes.transacao.autorizada`
Produz: `fraude.transacao.bloqueada`, `fraude.ocorrencia.criada`, `fraude.score.alterado`

### 4. aureus-notification

**Propósito:** Envio multicanal de notificações (Email, SMS, Push, WhatsApp).

**Porta:** 8126
**Context path:** `/api/notification`
**Kafka consumer group:** `aureus-notification-group`

#### Entidades

- `TemplateNotificacao` — codigo, nome, canal (EMAIL/SMS/PUSH/WHATSAPP), assunto, corpo_html, variaveis_json, ativo
- `FilaNotificacao` — destinatario, canal, template_codigo, variaveis_json, prioridade, status (PENDENTE/ENVIADO/FALHOU), data_criacao, data_envio, tentativas
- `ConfirmacaoRecebimento` — notificacao_id, canal, status (ENTREGUE/LIDA/CLICOU/BOUNCE), data_evento, metadata_json
- `PreferenciaCliente` — cliente_id, canal, ativo, horario_inicio, horario_fim, silencioso_noturno

#### REST API

| Método | Path | Descrição |
|--------|------|-----------|
| POST | `/api/notifications/enviar` | Enviar notificação imediata |
| POST | `/api/notifications/agendar` | Agendar para depois |
| GET | `/api/notifications/{id}` | Status da notificação |
| GET | `/api/notifications` | Consultar (filtro: cliente, status, canal) |
| POST | `/api/notifications/templates` | Criar template |
| GET | `/api/notifications/templates` | Listar templates |
| PATCH | `/api/notifications/templates/{id}` | Atualizar template |
| GET | `/api/notifications/preferencias/{clienteId}` | Preferências do cliente |
| PATCH | `/api/notifications/preferencias/{clienteId}` | Atualizar preferências |

#### Kafka Events

Consume: `cliente.criado`, `kyc.aprovado`, `kyc.rejeitado`, `fraude.transacao.bloqueada`
Produz: `notificacao.enviada`, `notificacao.falhou`

## Portas e Context Paths

| Módulo | Porta | Context Path | Grupo Kafka |
|--------|-------|-------------|-------------|
| aureus-customer | 8123 | `/api/customer` | aureus-customer-group |
| aureus-kyc | 8124 | `/api/kyc` | aureus-kyc-group |
| aureus-fraud | 8125 | `/api/fraud` | aureus-fraud-group |
| aureus-notification | 8126 | `/api/notification` | aureus-notification-group |

## Infra a adicionar

Por módulo:
- Dockerfile (mesmo padrão `eclipse-temurin:25-jdk-jammy`)
- SecurityConfig (`permitAll`)
- application.yml (porta, context-path, datasource, kafka, redis)
- Entrada no `docker-compose.yml` (imagem, porta, env vars, healthcheck, depends_on)
- Entrada no `infrastructure/traefik/dynamic.yml` (router + service)
- Entrada no `aureus-tests/e2e/config.py` (health endpoint)
- Módulo adicionado ao `backend/pom.xml` (`<modules>`)

## Dependências entre fases

```
Fase 1 (Foundation) ←── Fase 2 (Revenue) ←── Fase 3 (Post-trade)
     │                        │                        │
     ├── Customer             ├── Acquirer            ├── Trade
     ├── KYC                  ├── Collections         ├── Reconciliation
     ├── Fraud                ├── Guarantee           ├── Reporting
     └── Notification         └── (usa Customer,      └── Wealth
                                 KYC, Notification)      (usa Customer,
                                                          Notification)
```

Fase 2 depende de Customer + KYC (para ter clientes válidos) e Notification (para comunicação). Fase 3 depende de Customer + Notification.
