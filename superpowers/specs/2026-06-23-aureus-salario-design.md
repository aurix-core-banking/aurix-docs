# AUREUS Conta Salário — Design Specification

## 1. Visão Geral

Módulo dedicado `aureus-salario` para gestão de contas salário, convênios com empresas,
portabilidade de salário, e processamento de folha de pagamento via CNAB 240.

**Porta**: 8112
**Gateway route**: `/api/salario/**` (StripPrefix=0)

## 2. Decisões de Design

| Decisão | Escolha |
|---------|---------|
| Arquitetura | Módulo dedicado (`aureus-salario`), mesma abordagem de `aureus-poupanca` |
| Folha de pagamento | Interna com CNAB 240 + API REST complementar |
| Portabilidade | Sob demanda via app |
| Depósito não-salário | Redirecionar para conta corrente vinculada |
| Validação | Jakarta Validation (`@NotNull`, `@DecimalMin`, etc.) |
| Null safety | `@NullMarked` em todos os packages |
| Lombok | Não usar (convenção do código-base) |
| Feign | Não usar — `@HttpExchange` + `@ImportHttpServices` |
| Resilience | `@Retryable` (Spring native), `@ConcurrencyLimit(1)` |
| Eventos | Kafka, fire-and-forget com try-catch |
| Testes | `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `RestTemplate` |
| Gateway | StripPrefix=0 (padrão do código-base) |

## 3. Estrutura do Módulo

```
aureus-salario/
├── src/main/java/com/aureus/platform/salario/
│   ├── AureusSalarioApplication.java           (@SpringBootApplication, @EnableScheduling)
│   ├── entity/
│   │   ├── ContaSalario.java                   (FK lógica → Conta do core)
│   │   ├── ConvenioEmpresa.java                (empresa conveniada)
│   │   ├── FolhaPagamento.java                 (lote CNAB)
│   │   ├── ItemFolhaPagamento.java             (crédito individual)
│   │   └── SolicitacaoPortabilidade.java       (portabilidade)
│   ├── repository/
│   │   ├── ContaSalarioRepository.java
│   │   ├── ConvenioEmpresaRepository.java
│   │   ├── FolhaPagamentoRepository.java
│   │   ├── ItemFolhaPagamentoRepository.java
│   │   └── SolicitacaoPortabilidadeRepository.java
│   ├── dto/
│   │   ├── ContaSalarioRequest.java
│   │   ├── ContaSalarioResponse.java
│   │   ├── ConvenioRequest.java
│   │   ├── ConvenioResponse.java
│   │   ├── PortabilidadeRequest.java
│   │   ├── PortabilidadeResponse.java
│   │   ├── FolhaResponse.java
│   │   ├── ItemFolhaResponse.java
│   │   └── CreditoDiretoRequest.java
│   ├── event/
│   │   ├── ContaSalarioCriadaEvent.java
│   │   ├── SalarioCreditadoEvent.java
│   │   └── PortabilidadeSolicitadaEvent.java
│   ├── client/
│   │   ├── ContaCorrenteClient.java            (@HttpExchange → /api/core/contas)
│   │   └── CnabParser.java                     (parser caseiro, sem lib externa)
│   ├── service/
│   │   ├── ContaSalarioService.java
│   │   ├── ConvenioService.java
│   │   ├── PortabilidadeService.java
│   │   ├── CnabService.java
│   │   └── FolhaService.java
│   ├── controller/
│   │   ├── ContaSalarioController.java         (/api/salario/contas)
│   │   ├── ConvenioController.java             (/api/salario/convenios)
│   │   ├── PortabilidadeController.java        (/api/salario/portabilidade)
│   │   └── FolhaController.java                (/api/salario/folhas)
│   ├── config/
│   │   ├── SalarioKafkaConfig.java
│   │   ├── SalarioSecurityConfig.java
│   │   ├── SalarioHttpConfig.java
│   │   └── CnabConfig.java
│   └── job/
│       └── ProcessamentoFolhaJob.java
├── src/main/resources/
│   ├── application.yml
│   └── application-prod.yml
├── src/test/java/.../
│   ├── controller/
│   │   └── ContaSalarioControllerTest.java
│   └── service/
│       ├── CnabParserTest.java
│       └── FolhaServiceTest.java
├── src/test/resources/
│   ├── cnab/folha-valida.txt
│   └── application-test.yml
└── pom.xml
```

## 4. Modelo de Dados

### ContaSalario (`contas_salario`)

| Campo | Tipo | Restrições |
|-------|------|------------|
| id | Long | PK auto |
| tenantId | String | NOT NULL |
| contaCorrenteId | Long | FK lógica para Conta (core), NOT NULL |
| empresaId | Long | FK lógica para ConvenioEmpresa, NOT NULL |
| matriculaFuncionario | String(50) | NOT NULL |
| dataAdmissao | LocalDate | NOT NULL |
| dataRescisao | LocalDate | NULL se ativo |
| valorSalarioBruto | BigDecimal(15,2) | NOT NULL |
| valorSalarioLiquido | BigDecimal(15,2) | NOT NULL |
| diaPagamento | Integer(1-31) | NOT NULL |
| portabilidadeAtiva | Boolean | DEFAULT false |
| status | Enum(SALARIO) | ATIVA, RESCINDIDA, BLOQUEADA |

### ConvenioEmpresa (`convenios_empresa`)

| Campo | Tipo | Restrições |
|-------|------|------------|
| id | Long | PK auto |
| tenantId | String | NOT NULL |
| cnpj | String(14) | UNIQUE + tenantId, NOT NULL |
| razaoSocial | String(200) | NOT NULL |
| contaCorrenteId | Long | FK para Conta (core), NOT NULL |
| ativo | Boolean | DEFAULT true |

### FolhaPagamento (`folhas_pagamento`)

| Campo | Tipo | Restrições |
|-------|------|------------|
| id | Long | PK auto |
| empresaId | Long | FK ConvenioEmpresa, NOT NULL |
| arquivoNome | String(255) | NOT NULL |
| totalFuncionarios | Integer | NOT NULL |
| valorTotal | BigDecimal(15,2) | NOT NULL |
| dataReferencia | Date(YYYY-MM) | NOT NULL |
| dataProcessamento | LocalDateTime | NOT NULL |
| status | Enum | RECEBIDO, VALIDADO, PROCESSADO, ERRO_ESTRUTURA |

### ItemFolhaPagamento (`itens_folha_pagamento`)

| Campo | Tipo | Restrições |
|-------|------|------------|
| id | Long | PK auto |
| folhaId | Long | FK FolhaPagamento, NOT NULL |
| contaSalarioId | Long | FK ContaSalario, NOT NULL |
| cpfFuncionario | String(11) | NOT NULL |
| valorLiquido | BigDecimal(15,2) | NOT NULL |
| descontos | JSON | INSS, IRRF, planoSaude |
| status | Enum | PENDENTE, CREDITADO, PORTADO, ERRO |

### SolicitacaoPortabilidade (`solicitacoes_portabilidade`)

| Campo | Tipo | Restrições |
|-------|------|------------|
| id | Long | PK auto |
| contaSalarioId | Long | FK ContaSalario, NOT NULL |
| codigoBancoDestino | String(3) | Compensação, NOT NULL |
| agenciaDestino | String(10) | NOT NULL |
| contaDestino | String(20) | NOT NULL |
| valorPercentual | BigDecimal(5,2) | DEFAULT 100.00 |
| dataSolicitacao | LocalDateTime | NOT NULL |
| status | Enum | PENDENTE, ATIVA, CANCELADA |

## 5. Regras de Negócio

1. **Abertura**: só pode abrir ContaSalario se o cliente já tiver ContaCorrente ativa no core
2. **Movimentação**: só recebe crédito via CNAB da empresa conveniada ou via API de crédito direto
3. **Depósito avulso**: qualquer tentativa de depósito externo (PIX/TED não-CNAB) deve ser redirecionada para a conta corrente vinculada
4. **Cheque especial**: não existe (limiteCredito = 0, não alterável)
5. **Tarifas**: isento por lei (não cobrar manutenção)
6. **Portabilidade**: se ativa, todo crédito salário é automaticamente transferido para a conta corrente vinculada
7. **Rescisão**: empresa envia CNAB com indicador de rescisão → status RESCINDIDA → conta não recebe mais créditos
8. **Limite de 1 conta salário por empresa por CPF**: unique constraint (empresaId + cpf)

## 6. Fluxo CNAB 240

```
Upload arquivo (multipart)
  → CnabService.validaEstrutura(arquivo)
    → CnabParser.parse(linhas): retorna header + lotes + detalhes + trailers
    → valida: banco, header/trailer consistente, totalFuncionarios ok
    → salva FolhaPagamento(status=VALIDADO) + N ItemFolhaPagamento(status=PENDENTE)
  → erro de estrutura → salva FolhaPagamento(status=ERRO_ESTRUTURA)

Job noturno (ProcessamentoFolhaJob @Scheduled cron="0 3 * * *")
  → busca FolhaPagamento(status=VALIDADO)
  → para cada ItemFolhaPagamento:
    → busca ContaSalario por cpfFuncionario + empresaId
    → se ContaSalario:
      → se portabilidadeAtiva → ContaCorrenteClient.creditar(contaCorrenteId, valor)
      → senão → ContaSalario.saldo += valor, repository.save()
      → item.status = CREDITADO ou PORTADO
      → Kafka: salario-creditado
    → se não encontrar ContaSalario:
      → item.status = ERRO
  → FolhaPagamento.status = PROCESSADO
```

## 7. API Endpoints

### Contas Salário (`/api/salario/contas`)
- `POST /contas` — abrir conta salário (request: clienteId, empresaId, matricula, salarioBruto, salarioLiquido, diaPagamento)
- `GET /contas/{id}` — buscar por ID
- `GET /contas/cliente/{clienteId}` — listar por cliente
- `GET /contas/empresa/{empresaId}` — listar por empresa
- `PATCH /contas/{id}/bloquear` — bloquear
- `PATCH /contas/{id}/rescindir` — rescindir vínculo

### Convênios (`/api/salario/convenios`)
- `POST /convenios` — cadastrar empresa conveniada
- `GET /convenios/{id}` — buscar
- `PUT /convenios/{id}` — atualizar dados
- `GET /convenios` — listar ativos

### Portabilidade (`/api/salario/portabilidade`)
- `POST /portabilidade` — solicitar
- `GET /portabilidade/{id}` — consultar
- `DELETE /portabilidade/{id}` — cancelar
- `GET /portabilidade/conta/{contaSalarioId}` — listar solicitações

### Folha de Pagamento (`/api/salario/folhas`)
- `POST /folhas/upload` — upload CNAB 240 (multipart/form-data, campo "arquivo")
- `POST /folhas/credito-direto` — crédito manual (request: empresaId, cpfFuncionario, valor, descontos)
- `GET /folhas/{id}` — status do lote
- `GET /folhas` — listar lotes
- `GET /folhas/{id}/itens` — itens do lote com status

## 8. Kafka Events

| Tópico | Partições | Evento | Payload |
|--------|-----------|--------|---------|
| `salario-conta-criada` | 3 | `ContaSalarioCriadaEvent` | contaSalarioId, clienteId, empresaId, dataAdmissao |
| `salario-creditado` | 3 | `SalarioCreditadoEvent` | contaSalarioId, valor, tipo(CNAB/DIRETO), dataReferencia |
| `salario-portabilidade-solicitada` | 3 | `PortabilidadeSolicitadaEvent` | portabilidadeId, contaSalarioId, bancoDestino, percentual |

## 9. Dependências (pom.xml)

Mesmas do `aureus-poupanca` (Spring Boot Starters JPA, Validation, Web, Security, Kafka, Test)
+ `spring-boot-starter-batch` (processamento em lote do CNAB — opcional, pode ser @Scheduled simples)

## 10. Plano de Implementação Sugerido

1. **Scaffold**: pom.xml, AureusSalarioApplication, application.yml, gateway route, parent pom
2. **Domínio**: 5 entidades, 5 repos, DTOs, events, package-info
3. **HTTP Clients**: ContaCorrenteClient, SalarioHttpConfig
4. **Config**: Kafka, Security, CnabConfig
5. **ContaSalarioService + Controller + Tests**
6. **ConvenioService + Controller + Tests**
7. **PortabilidadeService + Controller + Tests**
8. **CnabParser + CnabService (upload/validação)**
9. **FolhaService + ProcessamentoFolhaJob + Tests**
10. **Full build verification**
