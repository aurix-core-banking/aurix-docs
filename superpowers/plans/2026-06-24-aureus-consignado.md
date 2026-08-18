# Plano de Implementação: `aurix-consignado` — Crédito Consignado

> Base: [Spec `2026-06-24-products-design.md`](../specs/2026-06-24-products-design.md) §3
> Porta: **8114** | Gateway: `/api/consignado/**` | Context-path: `/api/consignado`

---

## 1. Tarefas

### 1.1 Scaffold do módulo

- [ ] **Criar diretórios**
  ```
  apps/backend/aurix-consignado/
  ├── pom.xml
  ├── src/main/java/com/aurix/platform/consignado/
  │   ├── AurixConsignadoApplication.java
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
  └── src/test/java/com/aurix/platform/consignado/
      └── controller/
  ```
- [ ] **`pom.xml`** — copiar estrutura de `aurix-poupanca/pom.xml`, alterar `artifactId` para `aurix-consignado`, `name`/`description`. Dependências: `aurix-shared`, `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `spring-boot-starter-validation`, `spring-boot-starter-security`, `spring-boot-starter-actuator`, `spring-kafka`, `spring-boot-starter-data-redis`, `postgresql` (runtime), `h2` (test), `springdoc-openapi-starter-webmvc-ui` (2.2.0), `spring-boot-starter-test`, `spring-security-test`, `spring-kafka-test`, `testcontainers-junit-jupiter`, `testcontainers-postgresql`.
- [ ] **`AurixConsignadoApplication.java`** — `@SpringBootApplication`, `@EnableScheduling`, `@EnableResilientMethods` (ou colocar no config).
- [ ] **`package-info.java`** (root + cada subpacote) — `@NullMarked` com `import org.jspecify.annotations.NullMarked`.
- [ ] **`application.yml`** — `server.port: 8114`, `server.servlet.context-path: /api/consignado`, datasource PostgreSQL, JPA/Hibernate ddl-auto update, Redis, Kafka (`aurix-consignado-group`), Actuator, logging.
- [ ] **`application-prod.yml`** — ddl-auto: validate, pool maior, logging INFO.

### 1.2 Entidades

- [ ] **`entity/ContratoConsignado.java`**
  - `@Entity`, `@Table(name = "contratos_consignados")`
  - `Long id` (auto), `Long clienteId`, `Long contaSalarioId`, `BigDecimal valorTotal`, `BigDecimal taxaJuros`, `int prazoMeses`, `BigDecimal valorParcela`, `BigDecimal margemUtilizada`, `String fonteMargem` (INSS, SIAFI, EMPRESA), `String status` (PROPOSTA, ASSINADO, ATIVO, LIQUIDADO, INADIMPLENTE), `LocalDate dataContratacao`, `String tenantId`, `LocalDateTime dataCriacao`, `LocalDateTime dataAtualizacao`
  - `@PrePersist` / `@PreUpdate`
  - Getters/setters manuais (sem Lombok)

- [ ] **`entity/Parcela.java`**
  - `@Entity`, `@Table(name = "parcelas_consignadas")`
  - `Long id`, `Long contratoId`, `int numero`, `BigDecimal valor`, `LocalDate dataVencimento`, `LocalDate dataPagamento`, `String status` (PENDENTE, PAGA, ATRASADA, CANCELADA), `String tenantId`

- [ ] **`entity/MargemConsignavel.java`**
  - `@Entity`, `@Table(name = "margens_consignaveis")`
  - `Long id`, `Long clienteId`, `String fonteMargem`, `BigDecimal margemTotal`, `BigDecimal margemDisponivel`, `BigDecimal margemUtilizada`, `LocalDateTime dataAtualizacao`, `String tenantId`

- [ ] **`entity/ConvenioConsignado.java`**
  - `@Entity`, `@Table(name = "convenios_consignados")`
  - `Long id`, `String nome`, `String tipo` (INSS, SIAFI, EMPRESA), `String codigoFonte`, `boolean ativo`, `String tenantId`

- [ ] **`entity/ConsignadoSource.java`**
  - `@Entity`, `@Table(name = "consignado_sources")`
  - `Long id`, `String tipo`, `String endpoint`, `String credenciais`, `String config` (coluna TEXT/JSON), `String tenantId`

- [ ] **`entity/package-info.java`** — `@NullMarked`

### 1.3 Repositories

- [ ] **`repository/ContratoConsignadoRepository.java`** — `extends JpaRepository<ContratoConsignado, Long>`; `findByClienteId(Long)`, `findByStatus(String)`
- [ ] **`repository/ParcelaRepository.java`** — `findByContratoId(Long)`, `findByStatusAndDataVencimentoBefore(String, LocalDate)`
- [ ] **`repository/MargemConsignavelRepository.java`** — `findByClienteIdAndFonteMargem(Long, String)`, `findByClienteId(Long)`
- [ ] **`repository/ConvenioConsignadoRepository.java`** — `findByAtivoTrue()`
- [ ] **`repository/ConsignadoSourceRepository.java`** — `findByTipo(String)`
- [ ] **`repository/package-info.java`** — `@NullMarked`

### 1.4 DTOs (request/response)

- [ ] **`dto/CriarContratoRequest.java`** — `@NotNull Long clienteId`, `@NotNull Long contaSalarioId`, `@NotNull BigDecimal valorTotal`, `@NotNull BigDecimal taxaJuros`, `@NotNull int prazoMeses`, `@NotBlank String fonteMargem`
- [ ] **`dto/ContratoConsignadoResponse.java`** — todos os campos da entidade exceto audit
- [ ] **`dto/ParcelaResponse.java`** — id, contratoId, numero, valor, dataVencimento, dataPagamento, status
- [ ] **`dto/MargemResponse.java`** — clienteId, fonteMargem, margemTotal, margemDisponivel, margemUtilizada, dataAtualizacao
- [ ] **`dto/ConvenioRequest.java`** — nome, tipo, codigoFonte, ativo
- [ ] **`dto/ConvenioResponse.java`** — id, nome, tipo, codigoFonte, ativo
- [ ] **`dto/LiquidarRequest.java`** — (pode ser vazio ou conter `observacao`)
- [ ] **`dto/RenegociarRequest.java`** — `BigDecimal novoValor`, `int novoPrazoMeses`, `BigDecimal novaTaxaJuros`
- [ ] **`dto/package-info.java`** — `@NullMarked`

### 1.5 Eventos Kafka

- [ ] **`event/ContratoAssinadoEvent.java`** — `record(Long contratoId, Long clienteId, BigDecimal valorTotal, int prazoMeses, BigDecimal valorParcela, String fonteMargem, LocalDate dataContratacao, String tenantId)`
- [ ] **`event/ParcelaDebitadaEvent.java`** — `record(Long parcelaId, Long contratoId, int numero, BigDecimal valor, LocalDate dataPagamento, String tenantId)`
- [ ] **`event/MargemAtualizadaEvent.java`** — `record(Long clienteId, String fonteMargem, BigDecimal margemTotal, BigDecimal margemDisponivel, BigDecimal margemUtilizada, String tenantId)`
- [ ] **`event/ContratoLiquidadoEvent.java`** — `record(Long contratoId, Long clienteId, BigDecimal valorTotalPago, LocalDate dataLiquidacao, String tenantId)`
- [ ] **`event/package-info.java`** — `@NullMarked`

### 1.6 HTTP Clients (`@HttpExchange`)

- [ ] **`client/ContaSalarioClient.java`** — `@HttpExchange("/api/salario")`; `@PostExchange("/vincular/validar")` validarVinculo(...); `@PostExchange("/parcelas/debitar")` debitarParcela(...)
- [ ] **`client/SrccClient.java`** — `@HttpExchange("/api/srcc")`; `@GetExchange("/margem/{cpfCnpj}")` consultarMargem(@PathVariable String cpfCnpj); `@PostExchange("/contratos")` registrarContrato(...)
- [ ] **`client/DataprevClient.java`** — `@HttpExchange("/api/dataprev")`; `@GetExchange("/inss/margem/{cpf}")` consultarMargemInss(@PathVariable String cpf)
- [ ] **`client/SiafiClient.java`** — `@HttpExchange("/api/siafi")`; `@GetExchange("/servidor/margem/{cpf}")` consultarMargemServidor(@PathVariable String cpf)
- [ ] **`client/ESocialClient.java`** — `@HttpExchange("/api/esocial")`; `@GetExchange("/empresa/margem/{cpf}")` consultarMargemEmpresa(@PathVariable String cpf)
- [ ] **`client/package-info.java`** — `@NullMarked`
- [ ] **`client/*.java` records internos** — cada client declara `record Request/Response` conforme necessário (padrão `ContaCorrenteClient` / `TaxClient`)

### 1.7 Config

- [ ] **`config/ConsignadoHttpConfig.java`** — `@Configuration`, `@EnableResilientMethods`, `@ImportHttpServices({ContaSalarioClient.class, SrccClient.class, DataprevClient.class, SiafiClient.class, ESocialClient.class})`
- [ ] **`config/ConsignadoKafkaConfig.java`** — tópicos: `"consignado-contrato-assinado"` (3 partições), `"consignado-parcela-debitada"`, `"consignado-margem-atualizada"`, `"consignado-contrato-liquidado"`. Constantes `public static final String TOPICO_*`
- [ ] **`config/ConsignadoSecurityConfig.java`** — `@Configuration @EnableWebSecurity @Profile("!test")`, `SecurityFilterChain` — permitir `/actuator/**`, `/swagger-ui/**`, `/v3/api-docs/**`; autenticar `/consignados/**`, `/convenios/**`, `/margem/**`. Copiar padrão `PoupancaSecurityConfig`.
- [ ] **`config/package-info.java`** — `@NullMarked`

### 1.8 Services

- [ ] **`service/ContratoConsignadoService.java`**
  - `criarContrato(CriarContratoRequest)` — valida margem → cria `ContratoConsignado` + `Parcela`s → publica `ContratoAssinadoEvent` → se SRCC, registra na CIP
  - `buscarPorId(Long)`
  - `listarPorCliente(Long)`
  - `liquidar(Long)` — atualiza status `LIQUIDADO`, estorna margem, publica `ContratoLiquidadoEvent`
  - `renegociar(Long, RenegociarRequest)` — recalcula parcelas, atualiza margem
  - Kafka fire-and-forget com try-catch

- [ ] **`service/MargemService.java`**
  - `consultarMargem(Long clienteId)` — agrega todas as fontes (`ConvenioConsignado` ativos) + SRCC → retorna `MargemResponse`
  - `atualizarMargem(Long clienteId, String fonte, BigDecimal valorUtilizado)` — atualiza `MargemConsignavel`
  - `validarMargemDisponivel(Long clienteId, BigDecimal valorPretendido)` — verifica se cabe na margem

- [ ] **`service/ParcelaService.java`**
  - `gerarParcelas(ContratoConsignado)` — cria `Parcela`s com datas de vencimento
  - `listarParcelas(Long contratoId)`
  - `processarParcelasVencidas()` — chamado pelo job, debita via `ContaSalarioClient`, publica `ParcelaDebitadaEvent`

- [ ] **`service/ConvenioService.java`**
  - `listarConvenios()`
  - `criarConvenio(ConvenioRequest)`

- [ ] **`service/package-info.java`** — `@NullMarked`

### 1.9 Controllers

- [ ] **`controller/ConsignadoController.java`**
  - `@RestController @RequestMapping("/consignados") @Tag(name = "Credito Consignado")`
  - `POST /contratos` → `criarContrato` (201)
  - `GET /contratos/{id}` → `buscarPorId`
  - `GET /contratos/cliente/{clienteId}` → `listarPorCliente`
  - `GET /contratos/{id}/parcelas` → `listarParcelas`
  - `POST /contratos/{id}/liquidar` → `liquidar`
  - `PATCH /contratos/{id}/renegociar` → `renegociar`

- [ ] **`controller/ConvenioController.java`**
  - `@RestController @RequestMapping("/convenios") @Tag(name = "Convenios Consignado")`
  - `GET /` → `listarConvenios`
  - `POST /` → `criarConvenio` (201)

- [ ] **`controller/MargemController.java`** (ou dentro de `ConsignadoController` se preferir)
  - `GET /margem/{clienteId}` → `consultarMargem`

- [ ] **`controller/package-info.java`** — `@NullMarked`

### 1.10 Job

- [ ] **`job/ProcessamentoParcelasJob.java`**
  - `@Component`
  - `@Scheduled(cron = "0 0 3 * * *")` — diariamente 03:00
  - Busca parcelas `PENDENTE` com `dataVencimento.before(now)` ou `dataVencimento.equals(now)`
  - Para cada: tenta debitar via `ContaSalarioClient.debitarParcela()`, se sucesso → status `PAGA` + publica `ParcelaDebitadaEvent`, se falha → status `ATRASADA` + log de erro
  - Após processar, recalcula margem e publica `MargemAtualizadaEvent`
  - Segue padrão `ProcessamentoFolhaJob` (try-catch individual com log de erro)

- [ ] **`job/package-info.java`** — `@NullMarked`

### 1.11 Integração com Gateway

- [ ] **Gateway `application.yml`** — adicionar rota:
  ```yaml
  - id: aurix-consignado
    uri: http://localhost:8114
    predicates:
      - Path=/api/consignado/**
    filters:
      - StripPrefix=0
  ```

### 1.12 Parent POM

- [ ] **`apps/backend/pom.xml`** — adicionar `<module>aurix-consignado</module>` na lista de modules (ordenado alfabeticamente, após `aurix-compliance`).

### 1.13 OpenAPI Spec

- [ ] **OpenAPI spec** — adicionar tag `consignado`, paths do consignado e schemas correspondentes via springdoc-openapi.

### 1.14 Testes

- [ ] **`src/test/resources/application-test.yml`** — H2 in-memory, Kafka mock, Redis off (padrão poupanca)
- [ ] **`controller/ConsignadoControllerTest.java`**
  - `@SpringBootTest(webEnvironment = RANDOM_PORT)`, `@ActiveProfiles("test")`
  - `@Import(TestConfig.class)` com `SecurityFilterChain` permitAll + `@Primary KafkaTemplate mock`
  - `RestTemplate` com `@LocalServerPort`
  - Testes felizes: criar contrato, buscar por id, listar parcelas, liquidar
  - Testes validação: contrato sem clienteId → 4xx
- [ ] **`controller/ConvenioControllerTest.java`**
  - Teste feliz: criar convenio, listar
  - Teste validação: convenio sem nome → 4xx

---

## 2. Ordem de execução sugerida

1. Scaffold (pom.xml, Application, application.yml, package-info.java)
2. Entities + Repositories + package-info.java
3. DTOs
4. Events
5. HTTP Clients
6. Config (Http, Kafka, Security)
7. Services (ConvenioService → MargemService → ParcelaService → ContratoConsignadoService)
8. Controllers + MargemController
9. Job
10. Testes
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
| `application-test.yml` com H2 | `aurix-poupanca/src/test/resources/application-test.yml` |
| Endpoint `GET /{id}/parcelas` aninhado | Spec §3.3 |
| Validação de domínio na service (não no controller) | `ContaPoupancaService.java` |
