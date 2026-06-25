# Plano de Implementação: `aureus-cambio` — Câmbio

> Base: [Spec `2026-06-24-products-design.md`](../specs/2026-06-24-products-design.md) §5
> Porta: **8116** | Gateway: `/api/cambio/**` | Context-path: `/api/cambio`

---

## 1. Tarefas

### 1.1 Scaffold do módulo

- [ ] **Criar diretórios**
  ```
  backend/aureus-cambio/
  ├── pom.xml
  ├── src/main/java/com/aureus/platform/cambio/
  │   ├── AureusCambioApplication.java
  │   ├── package-info.java
  │   ├── entity/
  │   ├── repository/
  │   ├── dto/
  │   ├── event/
  │   ├── client/
  │   ├── config/
  │   ├── service/
  │   ├── controller/
  │   └── job/
  ├── src/main/resources/
  │   ├── application.yml
  │   └── application-prod.yml
  └── src/test/java/com/aureus/platform/cambio/
      ├── controller/
      └── resources/
          └── application-test.yml
  ```
- [ ] **`pom.xml`** — copiar estrutura de `aureus-poupanca/pom.xml`, alterar `artifactId` para `aureus-cambio`, `name`/`description`. Dependências: `aureus-shared`, `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `spring-boot-starter-validation`, `spring-boot-starter-security`, `spring-boot-starter-actuator`, `spring-kafka`, `spring-boot-starter-data-redis`, `postgresql` (runtime), `h2` (test), `springdoc-openapi-starter-webmvc-ui` (2.2.0), `spring-boot-starter-test`, `spring-security-test`, `spring-kafka-test`, `testcontainers-junit-jupiter`, `testcontainers-postgresql`
- [ ] **`AureusCambioApplication.java`** — `@SpringBootApplication`, `@EnableScheduling`
- [ ] **`package-info.java`** (root + cada subpacote) — `@NullMarked` com `import org.jspecify.annotations.NullMarked`
- [ ] **`application.yml`** — `server.port: 8116`, `server.servlet.context-path: /api/cambio`, datasource PostgreSQL, JPA/Hibernate ddl-auto update, Redis, Kafka (`aureus-cambio-group`), Actuator, logging. Propriedades custom: `cambio.cotacao.fonte-bacen-api-url`, `cambio.limite-remessa-mensal-pf: 10000`, `cambio.limite-remessa-anual-pf: 30000`
- [ ] **`application-prod.yml`** — ddl-auto: validate, pool maior, logging INFO

### 1.2 Entidades

- [ ] **`entity/ContratoCambio.java`**
  - `@Entity`, `@Table(name = "contratos_cambio")`
  - `Long id`, `Long clienteId`, `String tipo` (COMPRA, VENDA), `String moedaOrigem`, `String moedaDestino`, `BigDecimal valorOrigem`, `BigDecimal valorDestino`, `BigDecimal taxaCambio`, `LocalDate dataContratacao`, `LocalDate dataLiquidacao`, `String finalidade`, `String status` (COTADO, CONTRATADO, LIQUIDADO, CANCELADO), `String registroBACEN`, `String tenantId`, `LocalDateTime dataCriacao`, `LocalDateTime dataAtualizacao`
  - `@PrePersist` / `@PreUpdate`
  - Getters/setters manuais

- [ ] **`entity/Cotacao.java`**
  - `@Entity`, `@Table(name = "cotacoes_cambio")`
  - `Long id`, `String moeda`, `BigDecimal taxaCompra`, `BigDecimal taxaVenda`, `LocalDateTime dataCotacao`, `String fonte` (BACEN, PARCEIRO, PROPRIO), `String tenantId`

- [ ] **`entity/OperacaoCambio.java`**
  - `@Entity`, `@Table(name = "operacoes_cambio")`
  - `Long id`, `Long contratoId`, `Long clienteId`, `String tipo`, `BigDecimal valorMoedaEstrangeira`, `BigDecimal valorMoedaNacional`, `BigDecimal taxa`, `LocalDateTime dataOperacao`, `String registroBACEN`, `String tenantId`

- [ ] **`entity/Remessa.java`**
  - `@Entity`, `@Table(name = "remessas_cambio")`
  - `Long id`, `Long contratoId`, `Long clienteId`, `BigDecimal valor`, `String moeda`, `String bancoDestino`, `String contaDestino`, `String codigoSwift`, `String finalidade`, `String status` (PENDENTE, ENVIADA, CONFIRMADA, FALHADA), `LocalDateTime dataSolicitacao`, `LocalDateTime dataConfirmacao`, `String tenantId`

- [ ] **`entity/ContaCambio.java`**
  - `@Entity`, `@Table(name = "contas_cambio")`
  - `Long id`, `Long clienteId`, `String saldoPorMoeda` (coluna TEXT/JSON), `LocalDateTime dataAtualizacao`, `String tenantId`

- [ ] **`entity/ClienteCambio.java`**
  - `@Entity`, `@Table(name = "clientes_cambio")`
  - `Long id`, `Long clienteId`, `String documentacao` (coluna TEXT/JSON), `BigDecimal limiteRemessaMensal`, `BigDecimal limiteRemessaAnual`, `String categoriasAutorizadas` (coluna TEXT), `String tenantId`

- [ ] **`entity/package-info.java`** — `@NullMarked`

### 1.3 Repositories

- [ ] **`repository/ContratoCambioRepository.java`** — `extends JpaRepository<ContratoCambio, Long>`; `findByClienteId(Long)`, `findByStatus(String)`
- [ ] **`repository/CotacaoRepository.java`** — `findFirstByMoedaOrderByDataCotacaoDesc(String)`, `findByMoedaAndDataCotacaoAfter(String, LocalDateTime)`
- [ ] **`repository/OperacaoCambioRepository.java`** — `findByClienteId(Long)`, `findByContratoId(Long)`
- [ ] **`repository/RemessaRepository.java`** — `findByClienteId(Long)`, `findByStatus(String)`, `findByStatusIn(List<String>)`
- [ ] **`repository/ContaCambioRepository.java`** — `findByClienteId(Long)`
- [ ] **`repository/ClienteCambioRepository.java`** — `findByClienteId(Long)`
- [ ] **`repository/package-info.java`** — `@NullMarked`

### 1.4 DTOs (request/response)

- [ ] **`dto/CotacaoRequest.java`** — `@NotBlank String moeda`, `@NotNull BigDecimal taxaCompra`, `@NotNull BigDecimal taxaVenda`, `String fonte`
- [ ] **`dto/CotacaoResponse.java`** — id, moeda, taxaCompra, taxaVenda, dataCotacao, fonte
- [ ] **`dto/FecharContratoRequest.java`** — `@NotNull Long clienteId`, `@NotNull String tipo`, `@NotBlank String moedaOrigem`, `@NotBlank String moedaDestino`, `@NotNull BigDecimal valorOrigem`, `@NotNull BigDecimal taxaCambio`, `String finalidade`
- [ ] **`dto/ContratoCambioResponse.java`** — id, clienteId, tipo, moedaOrigem, moedaDestino, valorOrigem, valorDestino, taxaCambio, dataContratacao, dataLiquidacao, finalidade, status, registroBACEN, dataCriacao
- [ ] **`dto/RemessaRequest.java`** — `@NotNull Long contratoId`, `@NotNull Long clienteId`, `@NotNull BigDecimal valor`, `@NotBlank String moeda`, `@NotBlank String bancoDestino`, `@NotBlank String contaDestino`, `@NotBlank String codigoSwift`, `@NotBlank String finalidade`
- [ ] **`dto/RemessaResponse.java`** — id, contratoId, clienteId, valor, moeda, bancoDestino, contaDestino, codigoSwift (truncado), finalidade, status, dataSolicitacao, dataConfirmacao
- [ ] **`dto/OperacaoCambioResponse.java`** — id, contratoId, clienteId, tipo, valorMoedaEstrangeira, valorMoedaNacional, taxa, dataOperacao, registroBACEN
- [ ] **`dto/ClienteCambioRequest.java`** — `@NotNull Long clienteId`, `@NotNull BigDecimal limiteRemessaMensal`, `@NotNull BigDecimal limiteRemessaAnual`, `String categoriasAutorizadas`
- [ ] **`dto/ClienteCambioResponse.java`** — id, clienteId, limiteRemessaMensal, limiteRemessaAnual, categoriasAutorizadas, documentacao
- [ ] **`dto/LimiteCambioResponse.java`** — clienteId, limiteRemessaMensal, limiteRemessaAnual, totalRemessasMes, totalRemessasAno, saldoDisponivelMensal, saldoDisponivelAnual
- [ ] **`dto/LiquidarContratoRequest.java`** — (pode ser vazio ou conter `observacao`)
- [ ] **`dto/AtualizarLimiteRequest.java`** — `@NotNull BigDecimal limiteRemessaMensal`, `@NotNull BigDecimal limiteRemessaAnual`
- [ ] **`dto/package-info.java`** — `@NullMarked`

### 1.5 Eventos Kafka

- [ ] **`event/CotacaoAtualizadaEvent.java`** — `record(Long cotacaoId, String moeda, BigDecimal taxaCompra, BigDecimal taxaVenda, LocalDateTime dataCotacao, String fonte, String tenantId)`
- [ ] **`event/ContratoFechadoEvent.java`** — `record(Long contratoId, Long clienteId, String tipo, String moedaOrigem, String moedaDestino, BigDecimal valorOrigem, BigDecimal valorDestino, BigDecimal taxaCambio, String registroBACEN, LocalDate dataContratacao, String tenantId)`
- [ ] **`event/ContratoLiquidadoEvent.java`** — `record(Long contratoId, Long clienteId, BigDecimal valorLiquidado, LocalDate dataLiquidacao, String tenantId)`
- [ ] **`event/RemessaProcessadaEvent.java`** — `record(Long remessaId, Long contratoId, Long clienteId, BigDecimal valor, String moeda, String status, String codigoSwift, LocalDateTime dataConfirmacao, String tenantId)`
- [ ] **`event/package-info.java`** — `@NullMarked`

### 1.6 HTTP Clients (`@HttpExchange`)

- [ ] **`client/BacenClient.java`**
  - `@HttpExchange("/api/bacen")`
  - `@GetExchange("/cambio/taxas/{moeda}")` consultarTaxa(@PathVariable String moeda)
  - `@PostExchange("/cambio/contratos")` registrarContrato(@RequestBody RegistrarContratoBacenRequest request)
  - Record `RegistrarContratoBacenRequest` (contratoId, valor, moeda, clienteDoc, tipo)
  - Record `TaxaBacenResponse` (moeda, taxaCompra, taxaVenda, dataReferencia)

- [ ] **`client/SwiftClient.java`**
  - `@HttpExchange("/api/swift")`
  - `@PostExchange("/remessas/enviar")` enviarRemessa(@RequestBody EnviarRemessaSwiftRequest request)
  - `@GetExchange("/remessas/{id}/status")` consultarStatus(@PathVariable String id)
  - Record `EnviarRemessaSwiftRequest` (valor, moeda, bancoDestino, contaDestino, codigoSwift, finalidade)
  - Record `SwiftStatusResponse` (idExterno, statusSwift, dataConfirmacao)

- [ ] **`client/ComplianceClient.java`**
  - `@HttpExchange("/api/compliance")`
  - `@GetExchange("/cambio/roe/{clienteId}")` consultarRoe(@PathVariable Long clienteId)
  - `@PostExchange("/cambio/validar")` validarOperacao(@RequestBody ValidarOperacaoRequest request)
  - `@PostExchange("/cambio/registrar")` registrarOperacao(@RequestBody RegistrarOperacaoRequest request)
  - Record `ValidarOperacaoRequest` (clienteId, tipo, valor, moeda, finalidade)
  - Record `ValidacaoResponse` (aprovada, motivo, protocolo)

- [ ] **`client/ParceiroCambioClient.java`**
  - `@HttpExchange("/api/parceiro-cambio")`
  - `@GetExchange("/cotacoes/{moeda}")` consultarCotacao(@PathVariable String moeda)
  - `@PostExchange("/contratos")` registrarContratoParceiro(@RequestBody Object request)
  - (Record interno conforme parceiro concreto)

- [ ] **`client/package-info.java`** — `@NullMarked`

### 1.7 Config

- [ ] **`config/CambioHttpConfig.java`** — `@Configuration`, `@EnableResilientMethods`, `@ImportHttpServices({BacenClient.class, SwiftClient.class, ComplianceClient.class, ParceiroCambioClient.class})`
- [ ] **`config/CambioKafkaConfig.java`** — tópicos: `"cambio-cotacao-atualizada"` (3 partições), `"cambio-contrato-fechado"`, `"cambio-contrato-liquidado"`, `"cambio-remessa-processada"`. Constantes `public static final String TOPICO_*`
- [ ] **`config/CambioSecurityConfig.java`** — `@Configuration @EnableWebSecurity @Profile("!test")`, `SecurityFilterChain` — permitir `/actuator/**`, `/swagger-ui/**`, `/v3/api-docs/**`; autenticar `/cotacoes/**`, `/contratos/**`, `/remessas/**`, `/operacoes/**`, `/clientes/**`
- [ ] **`config/package-info.java`** — `@NullMarked`

### 1.8 Services

- [ ] **`service/CotacaoService.java`**
  - `listarCotacoes()` — retorna cotação mais recente de cada moeda
  - `obterCotacao(String moeda)` — retorna cotação atual de uma moeda
  - `atualizarCotacao(CotacaoRequest)` — cria `Cotacao`, publica `CotacaoAtualizadaEvent`
  - `atualizarCotacoesExternas()` — chamado pelo job, busca fontes externas (BACEN, parceiro), atualiza tabela

- [ ] **`service/ContratoCambioService.java`**
  - `fecharContrato(FecharContratoRequest)` — cria `ContratoCambio` status `COTADO` → valida compliance → status `CONTRATADO` → registra SISBACEN → publica `ContratoFechadoEvent`
  - `buscarPorId(Long)`
  - `listarPorCliente(Long)`
  - `liquidar(Long, LiquidarContratoRequest)` — debita/credita conforme tipo, status `LIQUIDADO`, publica `ContratoLiquidadoEvent`
  - `cancelar(Long)` — status `CANCELADO`
  - Kafka fire-and-forget com try-catch

- [ ] **`service/RemessaService.java`**
  - `solicitarRemessa(RemessaRequest)` — valida compliance (limites, finalidade), cria `Remessa` status `PENDENTE`
  - `buscarPorId(Long)`
  - `listarPorCliente(Long)`
  - `cancelar(Long)` — apenas se `PENDENTE`
  - `processarRemessa(Long)` — chamado pelo job, envia via `SwiftClient`, atualiza status e publica `RemessaProcessadaEvent`
  - `processarRemessasPendentes()` — lote, chamado pelo job

- [ ] **`service/ComplianceCambialService.java`**
  - `validarLimitesCliente(Long clienteId, BigDecimal valor, String moeda)` — consulta `ClienteCambio`, verifica limites mensais/anuais gastos via `RemessaRepository` e `ContratoCambioRepository`
  - `validarFinalidade(String finalidade)` — verifica se finalidade é permitida
  - `registrarOperacaoBacen(ContratoCambio)` — chama `ComplianceClient.registrarOperacao()` + `BacenClient.registrarContrato()`
  - `consultarRoe(Long clienteId)` — chama `ComplianceClient.consultarRoe()`

- [ ] **`service/ClienteCambioService.java`**
  - `listarClientes()`
  - `habilitarCliente(ClienteCambioRequest)` — cria `ClienteCambio`
  - `ajustarLimites(Long id, AtualizarLimiteRequest)`
  - `consultarLimites(Long clienteId)` — retorna `LimiteCambioResponse` com gastos agregados

- [ ] **`service/package-info.java`** — `@NullMarked`

### 1.9 Controllers

- [ ] **`controller/CotacaoController.java`**
  - `@RestController @RequestMapping("/cotacoes") @Tag(name = "Cotacoes Cambio")`
  - `GET /` → listar cotações atuais (todas moedas)
  - `GET /{moeda}` → cotação atual de uma moeda
  - `POST /` → atualizar cotação (admin/job)

- [ ] **`controller/ContratoController.java`**
  - `@RestController @RequestMapping("/contratos") @Tag(name = "Contratos Cambio")`
  - `POST /` → fechar contrato (201)
  - `GET /{id}` → obter contrato
  - `GET /cliente/{clienteId}` → listar contratos do cliente
  - `PATCH /{id}/liquidar` → liquidar contrato
  - `PATCH /{id}/cancelar` → cancelar

- [ ] **`controller/RemessaController.java`**
  - `@RestController @RequestMapping("/remessas") @Tag(name = "Remessas Cambio")`
  - `POST /` → solicitar remessa (201)
  - `GET /{id}` → status da remessa
  - `GET /cliente/{clienteId}` → histórico de remessas
  - `PATCH /{id}/cancelar` → cancelar remessa

- [ ] **`controller/AdminController.java`**
  - `@RestController @RequestMapping("/clientes") @Tag(name = "Admin Cambio")`
  - `GET /cliente/{clienteId}/operacoes` → extrato cambial (ou via `GET /api/cambio/operacoes/{clienteId}`)
  - `GET /cliente/{clienteId}/limites` → limites disponíveis
  - `GET /` → listar clientes de câmbio
  - `POST /` → habilitar cliente (201)
  - `PUT /{id}/limites` → ajustar limites

- [ ] **`controller/package-info.java`** — `@NullMarked`

### 1.10 Jobs

- [ ] **`job/CotacaoJob.java`**
  - `@Component`
  - `@Scheduled(cron = "0 0 6,12,18 * * *")` — 3x ao dia (6h, 12h, 18h)
  - Chama `CotacaoService.atualizarCotacoesExternas()` — busca taxas BACEN e/ou parceiro, persiste, publica `CotacaoAtualizadaEvent`

- [ ] **`job/FechamentoDiarioJob.java`**
  - `@Component`
  - `@Scheduled(cron = "0 30 23 * * *")` — diariamente 23:30
  - Busca contratos `CONTRATADO` sem liquidação no dia, gera registros BACEN, atualiza status pendentes
  - Publica relatório interno (log estruturado)

- [ ] **`job/RemessaJob.java`**
  - `@Component`
  - `@Scheduled(fixedDelay = 60000)` — a cada 60s (ou `cron` mais espaçado em prod)
  - Chama `RemessaService.processarRemessasPendentes()` — busca remessas `PENDENTE`, envia via SWIFT, atualiza status
  - Cada remessa em try-catch individual com log de erro

- [ ] **`job/package-info.java`** — `@NullMarked`

### 1.11 Integração com Gateway

- [ ] **Gateway `application.yml`** — adicionar rota após a seção de AUREUS Credit:
  ```yaml
  - id: aureus-cambio
    uri: http://localhost:8116
    predicates:
      - Path=/api/cambio/**
    filters:
      - StripPrefix=0
  ```

### 1.12 Parent POM

- [ ] **`backend/pom.xml`** — adicionar `<module>aureus-cambio</module>` na lista de modules (entre `aureus-bacen` e `aureus-cartoes`, ordem alfabética).

### 1.13 OpenAPI Spec

- [ ] **`aureus-api-specs/aureus-core.yaml`** — adicionar tag `cambio`, paths do câmbio (`/api/cambio/cotacoes`, `/api/cambio/contratos`, `/api/cambio/remessas`, `/api/cambio/operacoes`, `/api/cambio/clientes`) e schemas correspondentes (`Cotacao`, `ContratoCambio`, `Remessa`, `OperacaoCambio`, `ClienteCambio`).

### 1.14 Testes

- [ ] **`src/test/resources/application-test.yml`** — H2 in-memory, Kafka mock (`spring.kafka.producer.bootstrap-servers: localhost:0`), Redis off (`spring.redis.host: localhost`, `spring.redis.port: 0` e `spring.autoconfigure.exclude: org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration`)
- [ ] **`controller/CotacaoControllerTest.java`**
  - `@SpringBootTest(webEnvironment = RANDOM_PORT)`, `@ActiveProfiles("test")`
  - `@Import(TestConfig.class)` com `SecurityFilterChain` permitAll + `@Primary KafkaTemplate mock`
  - Testes felizes: listar cotações, obter cotação por moeda, criar cotação
  - Testes validação: criar cotação sem moeda → 4xx
- [ ] **`controller/ContratoControllerTest.java`**
  - Testes felizes: fechar contrato, buscar por id, listar por cliente
  - Testes validação: contrato sem clienteId → 4xx
  - Teste fluxo: fechar → liquidar
- [ ] **`controller/RemessaControllerTest.java`**
  - Testes felizes: solicitar remessa, buscar status, listar histórico
  - Testes validação: remessa sem swift → 4xx
  - Teste cancelamento
- [ ] **`controller/AdminControllerTest.java`**
  - Teste feliz: habilitar cliente, consultar limites, listar clientes
  - Teste validação: habilitar sem clienteId → 4xx

---

## 2. Ordem de execução sugerida

1. Scaffold (pom.xml, Application, application.yml, application-prod.yml, package-info.java)
2. Entities + Repositories + package-info.java
3. DTOs
4. Events
5. HTTP Clients
6. Config (Http, Kafka, Security)
7. Services (CotacaoService → ClienteCambioService → ComplianceCambialService → ContratoCambioService → RemessaService)
8. Controllers (CotacaoController → ContratoController → RemessaController → AdminController)
9. Jobs (CotacaoJob → FechamentoDiarioJob → RemessaJob)
10. Testes (controller tests + application-test.yml)
11. Gateway route, Parent POM, OpenAPI

---

## 3. Padrões a seguir

| Padrão | Fonte |
|--------|-------|
| Getter/setter manuais (sem Lombok) | `ContaPoupanca.java`, `MovimentacaoPoupanca.java` |
| `@NullMarked` package-level | `poupanca/*/package-info.java` |
| `@HttpExchange` + `@ImportHttpServices` | `PoupancaHttpConfig.java`, `ContaCorrenteClient.java` |
| `@Retryable` (Spring Resilience) | `AniversarioService.processarConta()` |
| Kafka fire-and-forget com try-catch | `AniversarioService.java:92-97` |
| `@Scheduled` em `@Component` | `ProcessamentoFolhaJob.java` |
| Test: `@SpringBootTest(RANDOM_PORT)` + TestConfig | `ContaPoupancaControllerTest.java` |
| `application-test.yml` com H2 | `aureus-poupanca/src/test/resources/application-test.yml` |
| Validação de domínio na service (não no controller) | `ContaPoupancaService.java` |
| Service `@Transactional` | `ContaPoupancaService.java` |
