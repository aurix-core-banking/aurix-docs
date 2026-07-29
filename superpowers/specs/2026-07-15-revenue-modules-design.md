# Revenue Modules — Design Spec (Fase 2)

> **Objetivo:** Implementar 3 módulos de receita para a plataforma AURIX: Acquirer (Adquirência), Collections (Cobrança), Guarantee (Garantias).
> **Prioridade:** Fase 2 — depende de Customer + Notification (Foundation, Fase 1).
> **Arquitetura:** Microsserviços Spring Boot 4.1.0 + Kafka + PostgreSQL. Mesmo padrão dos 4 módulos Foundation.

## Arquitetura Geral

```
[Customer] ──▶ [Acquirer] ──Kafka: acquirer.transacao.autorizada.v1──▶ [Notification]
    │                          │
    │                          └──Kafka: acquirer.transacao.liquidada.v1──▶ [Settlement]
    │
    ├──▶ [Collections] ──Kafka: collections.boleto.emitido.v1──▶ [Notification]
    │       │
    │       └──Kafka: collections.cobranca.paga.v1──▶ [Bacen]
    │
    └──▶ [Guarantee] ──Kafka: guarantee.garantia.registrada.v1──▶ [Audit]
```

### Padrão comum a todos os módulos

- **Framework:** Spring Boot 4.1.0, Java 25
- **Banco:** PostgreSQL, schema `aurix`, gerenciado por Flyway
- **Kafka:** `JsonSerializer`/`JsonDeserializer` (shared KafkaConfig), eventos tipados `BaseEvent`
- **Infra:** Dockerfile (`eclipse-temurin:25-jdk-jammy`), entrypoint no `docker-compose.yml`, rota no `traefik/dynamic.yml`
- **Portas:** 8127 (acquirer), 8128 (collections), 8129 (guarantee)
- **Context paths:** `/api/acquirer`, `/api/collections`, `/api/guarantee`
- **Security:** `permitAll` nas rotas (padrão dev), autenticação via gateway
- **Package naming:** `com.aurix.platform.acquirer`, `.collections`, `.guarantee`

## 1. Módulo Acquirer (Porta 8127)

### Propósito

Captura, processamento e liquidação de transações de cartão de crédito/débito (presencial + online) e subadquirência. Gerencia estabelecimentos, terminais, taxas e ciclos de liquidação.

### Entidades

**Estabelecimento** (`acquirer.estabelecimentos`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| clienteId | Long | FK → customer.cliente |
| nomeFantasia | String | Nome comercial |
| cnpj | String (14) | CNPJ do estabelecimento |
| bandeirasHabilitadas | String (JSON array) | Bandeiras aceitas |
| taxaAdmin | BigDecimal | Taxa de administração % |
| prazoLiquidacao | Integer | Dias para liquidação |
| status | String (ATIVO/BLOQUEADO/INATIVO) | Status do estabelecimento |

**Terminal** (`acquirer.terminals`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| estabelecimentoId | Long | FK → estabelecimento |
| modelo | String | Modelo do terminal |
| tipo | String (POS/APP/WEB) | Tipo de terminal |
| codigo | String (unique) | Código identificador |
| status | String (ATIVO/INATIVO) | Status |

**TransacaoCaptura** (`acquirer.transacoes`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| terminalId | Long | FK → terminal |
| bandeira | String | Bandeira (VISA/MASTERCARD/ELO) |
| valor | BigDecimal | Valor da transação |
| parcelas | Integer | Número de parcelas |
| dadosCartao | String (tokenizado) | Cartão tokenizado |
| status | String | AUTORIZADA/CAPTURADA/LIQUIDADA/ESTORNADA |
| codigoAutorizacao | String | Código de autorização |
| nsu | String | NSU da transação |
| dataHora | LocalDateTime | Data/hora da transação |
| tenantId | String | Tenant ID |

**Liquidacao** (`acquirer.liquidacoes`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| transacaoId | Long | FK → transacao |
| valorLiquido | BigDecimal | Valor líquido (valor - taxas) |
| taxaAdmin | BigDecimal | Taxa de administração |
| valorRepasse | BigDecimal | Valor a repassar |
| dataLiquidacao | LocalDateTime | Data da liquidação |
| status | String | PENDENTE/PROCESADA/REALIZADA |

**TaxaAcquirer** (`acquirer.taxas`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| bandeira | String | Bandeira |
| tipoTransacao | String | DEBITO/CREDITO/PARCELADO |
| percentual | BigDecimal | Percentual da taxa |
| valorFixo | BigDecimal | Valor fixo por transação |
| produtoCredito | String | Produto de crédito associado |

### REST API

| Método | Path | Descrição |
|--------|------|-----------|
| POST | `/api/acquirer/transacoes/autorizar` | Autorizar transação |
| POST | `/api/acquirer/transacoes/capturar` | Capturar transação autorizada |
| POST | `/api/acquirer/transacoes/estornar` | Estornar transação |
| GET | `/api/acquirer/transacoes` | Consultar transações (filtro: terminal, data, status) |
| GET | `/api/acquirer/transacoes/{id}` | Detalhe da transação |
| POST | `/api/acquirer/estabelecimentos` | Cadastrar estabelecimento |
| GET | `/api/acquirer/estabelecimentos/{id}` | Consultar estabelecimento |
| GET | `/api/acquirer/estabelecimentos` | Listar estabelecimentos |
| GET | `/api/acquirer/liquidacoes` | Consultar liquidações pendentes |
| POST | `/api/acquirer/liquidacoes/processar` | Processar lote de liquidações |

### Kafka Events

- `acquirer.transacao.autorizada.v1` — quando transação é autorizada
- `acquirer.transacao.capturada.v1` — quando transação é capturada
- `acquirer.transacao.liquidada.v1` — quando liquidação é processada
- `acquirer.transacao.estornada.v1` — quando transação é estornada

Eventos estendem `BaseEvent`, em `com.aurix.platform.shared.event`:
- `TransacaoAutorizadaEvent` — transacaoId, terminalId, valor, bandeira, autorizacao, nsu
- `TransacaoCapturadaEvent` — transacaoId, valor, parcelas, bandeira
- `TransacaoLiquidadaEvent` — liquidacaoId, transacaoId, valorLiquido, valorRepasse
- `TransacaoEstornadaEvent` — transacaoId, valor, motivo

## 2. Módulo Collections (Porta 8128)

### Propósito

Gestão completa de cobrança: emissão de boletos registrados, carnês de parcelas, acordos, negativação em órgãos de crédito, recuperação.

### Entidades

**Cobranca** (`collections.cobrancas`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| clienteId | Long | FK → customer.cliente |
| tipo | String | BOLETO/CARNE/ACORDO |
| valor | BigDecimal | Valor da cobrança |
| valorPago | BigDecimal | Valor pago |
| dataVencimento | LocalDate | Data de vencimento |
| dataPagamento | LocalDate | Data de pagamento |
| status | String | EMITIDA/VENCIDA/PAGA/CANCELADA/NEGATIVADA |
| nossoNumero | String | Nosso número (unique) |
| linhaDigitavel | String | Linha digitável do boleto |
| codigoBarras | String | Código de barras |
| tenantId | String | Tenant ID |

**CarnetParcela** (`collections.carnet_parcelas`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| cobrancaId | Long | FK → cobranca |
| numeroParcela | Integer | Número da parcela |
| valor | BigDecimal | Valor da parcela |
| dataVencimento | LocalDate | Vencimento |
| status | String | PENDENTE/PAGA/ATRASADA |
| nossoNumero | String | Nosso número da parcela |

**Acordo** (`collections.acordos`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| cobrancaOrigemId | Long | FK → cobranca |
| clienteId | Long | FK → customer.cliente |
| valorTotal | BigDecimal | Valor total do acordo |
| numeroParcelas | Integer | Número de parcelas |
| valorParcela | BigDecimal | Valor de cada parcela |
| dataPrimeiroVencimento | LocalDate | Primeiro vencimento |
| status | String | ATIVO/QUITADO/RESCINDIDO |

**Negativacao** (`collections.negativacoes`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| cobrancaId | Long | FK → cobranca |
| orgao | String | SERASA/SPC/QUOD |
| dataEnvio | LocalDate | Data do envio |
| dataBaixa | LocalDate | Data da baixa |
| status | String | NEGATIVADA/BAIXADA |

**RegistroCobranca** (`collections.registros`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| cobrancaId | Long | FK → cobranca |
| camara | String | CIP/C6/BANCO_DO_BRASIL |
| dataRegistro | LocalDateTime | Data do registro |
| protocolo | String | Protocolo de registro |
| status | String | PENDENTE/REGISTRADO/REJEITADO |

### REST API

| Método | Path | Descrição |
|--------|------|-----------|
| POST | `/api/collections/boletos/emitir` | Emitir boleto registrado |
| GET | `/api/collections/boletos/{nossoNumero}` | Consultar boleto |
| POST | `/api/collections/carnes` | Criar carnê de parcelas |
| POST | `/api/collections/acordos` | Propor acordo |
| POST | `/api/collections/negativar` | Negativar cobrança |
| POST | `/api/collections/baixar` | Baixar negativação |
| GET | `/api/collections/cobrancas` | Consultar cobranças (filtro: cliente, status, data) |
| GET | `/api/collections/cobrancas/{id}` | Detalhe da cobrança |
| GET | `/api/collections/negativacoes` | Consultar negativações ativas |
| POST | `/api/collections/registrar` | Solicitar registro em câmara |

### Kafka Events

- `collections.boleto.emitido.v1` — boleto emitido com sucesso
- `collections.cobranca.paga.v1` — cobrança recebida
- `collections.cobranca.negativada.v1` — cobrança negativada
- `collections.cobranca.cancelada.v1` — cobrança cancelada

Eventos no shared:
- `BoletoEmitidoEvent` — cobrancaId, nossoNumero, valor, dataVencimento, clienteId
- `CobrancaPagaEvent` — cobrancaId, valorPago, dataPagamento
- `CobrancaNegativadaEvent` — cobrancaId, orgao, dataEnvio
- `CobrancaCanceladaEvent` — cobrancaId, motivo

## 3. Módulo Guarantee (Porta 8129)

### Propósito

Gestão centralizada de garantias: alienação fiduciária, penhor, hipoteca, fiança. Extraído do `aurix-financiamento` com expansão para suportar múltiplos tipos de contrato.

### Entidades

**Garantia** (`guarantee.garantias`) — extraída do finanziamento
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| contratoId | Long | FK → contrato (genérico, pode ser de qualquer módulo) |
| clienteId | Long | FK → customer.cliente |
| bemId | Long | FK → bem |
| tipo | String | ALIENACAO_FIDUCIARIA/PENHOR/HIPOTECA/FIANCA |
| valor | BigDecimal | Valor da garantia |
| dataRegistro | LocalDate | Data de registro |
| dataVencimento | LocalDate | Data de vencimento |
| status | String | ATIVA/LIBERADA/VENCIDA |
| dataBaixa | LocalDate | Data de baixa/liberação |
| tenantId | String | Tenant ID |

**Bem** (`guarantee.bens`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| tipo | String | IMOVEL/VEICULO/EQUIPAMENTO/TITULO |
| descricao | String | Descrição do bem |
| valorAvaliacao | BigDecimal | Valor de avaliação |
| registroCartorio | String | Registro em cartório |
| chassi | String | Chassi (veículos) |
| placa | String | Placa (veículos) |
| tenantId | String | Tenant ID |

**Avaliacao** (`guarantee.avaliacoes`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| bemId | Long | FK → bem |
| data | LocalDate | Data da avaliação |
| valor | BigDecimal | Valor avaliado |
| metodo | String | Método de avaliação |
| avaliadorId | Long | ID do avaliador |
| validadeAte | LocalDate | Validade da avaliação |

**RegistroGarantia** (`guarantee.registros`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK, auto) | Identificador |
| garantiaId | Long | FK → garantia |
| orgao | String | CARTORIO/DETRAN/JUCESP |
| dataRegistro | LocalDateTime | Data do registro |
| protocolo | String | Protocolo |
| status | String | PENDENTE/REGISTRADO/REJEITADO |

### REST API

| Método | Path | Descrição |
|--------|------|-----------|
| POST | `/api/guarantee/garantias` | Registrar garantia |
| PATCH | `/api/guarantee/garantias/{id}/liberar` | Liberar (baixar) garantia |
| GET | `/api/guarantee/garantias/{id}` | Consultar garantia |
| GET | `/api/guarantee/garantias` | Listar garantias (filtro: cliente, tipo, status) |
| POST | `/api/guarantee/bens` | Cadastrar bem |
| GET | `/api/guarantee/bens/{id}` | Consultar bem |
| POST | `/api/guarantee/avaliacoes` | Registrar avaliação |
| GET | `/api/guarantee/garantias/{id}/avaliacoes` | Histórico de avaliações |
| GET | `/api/guarantee/bens` | Listar bens |

### Kafka Events

- `guarantee.garantia.registrada.v1` — garantia registrada com sucesso
- `guarantee.garantia.liberada.v1` — garantia liberada/baixada

Eventos no shared:
- `GarantiaRegistradaEvent` — garantiaId, contratoId, clienteId, tipo, valor
- `GarantiaLiberadaEvent` — garantiaId, contratoId, dataBaixa

## 4. Extração do Guarantee do Financiamento

### O que extrair do `aurix-financiamento`

| Arquivo | Destino |
|---------|---------|
| `GarantiaService.java` | `aurix-guarantee/src/main/java/.../service/GarantiaService.java` |
| `GarantiaRepository.java` | `aurix-guarantee/src/main/java/.../repository/GarantiaRepository.java` |
| `Garantia.java` (entity) | `aurix-guarantee/src/main/java/.../entity/Garantia.java` |
| `GarantiaRequest.java` (DTO) | `aurix-guarantee/src/main/java/.../dto/GarantiaRequest.java` |
| `GarantiaControllerTest.java` | `aurix-guarantee/src/test/java/.../controller/GarantiaControllerTest.java` |
| `GarantiaResponse.java` | `aurix-guarantee/src/main/java/.../dto/GarantiaResponse.java` |
| `AtualizacaoGarantiasJob.java` | `aurix-guarantee/src/main/java/.../job/AtualizacaoGarantiasJob.java` |

### Ajustes pós-extração

1. `aurix-guarantee` adiciona um `GarantiaClient` (Feign) que expõe os mesmos endpoints do GarantiaService para consumo interno
2. `aurix-financiamento` substitui injeção direta de `GarantiaRepository` por `GarantiaClient`
3. Schema da tabela `garantias` permanece em `aurix` (mesmo schema dos demais módulos)
4. `aurix-financiamento/pom.xml` adiciona dependência `aurix-guarantee`
5. `aurix-guarantee` usa as mesmas entidades compartilhadas (`BaseEvent` do shared)
6. Testes do financiamento que usam `GarantiaRepository` passam a usar mock de `GarantiaClient`

### Estratégia de migração

1. Criar módulo `aurix-guarantee` com estrutura básica (pom.xml, application.yml)
2. Copiar arquivos, ajustar pacotes de `...financiamento...` → `...guarantee...`
3. Criar `GarantiaClient` (Feign) no `aurix-guarantee`
4. Atualizar `aurix-financiamento` para usar `GarantiaClient`
5. Remover classes originais do `aurix-financiamento`
6. Compilar ++ testar ambos os módulos

## 5. Infraestrutura

### Por módulo

- `Dockerfile` — `eclipse-temurin:25-jdk-jammy`, EXPOSE 8127/8128/8129
- `application.yml` — porta, context-path, datasource (`jdbc:postgresql://postgres:5432/aurix`), kafka (`localhost:9092`), redis
- `docker-compose.yml` — image, ports, environment, healthcheck, depends_on (postgres, kafka)
- `infra/traefik/dynamic.yml` — router + service para cada módulo
- `tests/e2e/config.py` — health endpoint para cada módulo
- `apps/backend/pom.xml` — adicionar `<module>aurix-acquirer</module>`, `<module>aurix-collections</module>`, `<module>aurix-guarantee</module>`

### Tabela de portas e context paths

| Módulo | Porta | Context Path | Grupo Kafka |
|--------|-------|-------------|-------------|
| aurix-acquirer | 8127 | `/api/acquirer` | aurix-acquirer-group |
| aurix-collections | 8128 | `/api/collections` | aurix-collections-group |
| aurix-guarantee | 8129 | `/api/guarantee` | aurix-guarantee-group |

## 6. Dependências

```
aurix-financiamento ──▶ aurix-guarantee (via GarantiaClient Feign)
aurix-acquirer ──▶ aurix-shared (eventos tipados)
aurix-collections ──▶ aurix-shared (eventos tipados)
aurix-guarantee ──▶ aurix-shared (eventos tipados)
```

Nenhum módulo Fase 2 depende de outro módulo Fase 2 — são independentes entre si.

## 7. Ordem de implementação

1. **Guarantee** — extrair primeiro (desbloqueia financiamento + outros contratos)
2. **Acquirer** — novo módulo, sem dependências de extração
3. **Collections** — novo módulo, sem dependências de extração

## 8. Testes

- **Unitários:** JUnit 5 + Mockito, service tests com entidades mockadas
- **Integração:** Testcontainers (PostgreSQL + Kafka) para fluxos completos
- **Contrato:** OpenAPI spec para cada módulo em `specs/`
- **E2E:** Health-check endpoints em `tests/e2e/`

## 9. Eventos no shared

Adicionar em `Topics.java`:

```java
// ===== acquirer =====
String ACQUIRER_TRANSACAO_AUTORIZADA = "acquirer.transacao.autorizada.v1";
String ACQUIRER_TRANSACAO_CAPTURADA = "acquirer.transacao.capturada.v1";
String ACQUIRER_TRANSACAO_LIQUIDADA = "acquirer.transacao.liquidada.v1";
String ACQUIRER_TRANSACAO_ESTORNADA = "acquirer.transacao.estornada.v1";

// ===== collections =====
String COLLECTIONS_BOLETO_EMITIDO = "collections.boleto.emitido.v1";
String COLLECTIONS_COBRANCA_PAGA = "collections.cobranca.paga.v1";
String COLLECTIONS_COBRANCA_NEGATIVADA = "collections.cobranca.negativada.v1";
String COLLECTIONS_COBRANCA_CANCELADA = "collections.cobranca.cancelada.v1";

// ===== guarantee =====
String GUARANTEE_GARANTIA_REGISTRADA = "guarantee.garantia.registrada.v1";
String GUARANTEE_GARANTIA_LIBERADA = "guarantee.garantia.liberada.v1";
```

Adicionar classes de evento tipado em `com.aurix.platform.shared.event`:
- `TransacaoAutorizadaEvent`, `TransacaoCapturadaEvent`, `TransacaoLiquidadaEvent`, `TransacaoEstornadaEvent`
- `BoletoEmitidoEvent`, `CobrancaPagaEvent`, `CobrancaNegativadaEvent`, `CobrancaCanceladaEvent`
- `GarantiaRegistradaEvent`, `GarantiaLiberadaEvent`
