# Plano de Implementação: `aurix-seguros` — Seguros

> Base: [Spec `2026-06-24-products-design.md`](../specs/2026-06-24-products-design.md) §6
> Porta: **8117** | Gateway: `/api/seguros/**` | Context-path: `/api/seguros`

---

## 1. Tarefas

### 1.1 Scaffold do módulo

- [ ] **Criar diretórios**
  ```
  apps/backend/aurix-seguros/
  ├── pom.xml
  ├── docker/
  ├── src/main/java/com/aurix/platform/seguros/
  │   ├── AurixSegurosApplication.java
  │   ├── package-info.java
  │   ├── entity/
  │   │   ├── package-info.java
  │   │   ├── ProdutoSeguro.java
  │   │   ├── CotacaoSeguro.java
  │   │   ├── Apolice.java
  │   │   ├── Tomador.java
  │   │   ├── ParcelaPremio.java
  │   │   ├── Sinistro.java
  │   │   ├── Comissao.java
  │   │   ├── Corretor.java
  │   │   └── TipoSeguro.java (enum avulso ou inner)
  │   ├── repository/
  │   │   ├── package-info.java
  │   │   ├── ProdutoSeguroRepository.java
  │   │   ├── CotacaoSeguroRepository.java
  │   │   ├── ApoliceRepository.java
  │   │   ├── TomadorRepository.java
  │   │   ├── ParcelaPremioRepository.java
  │   │   ├── SinistroRepository.java
  │   │   ├── ComissaoRepository.java
  │   │   └── CorretorRepository.java
  │   ├── dto/
  │   │   ├── package-info.java
  │   │   ├── request/
  │   │   │   ├── package-info.java
  │   │   │   ├── CotacaoRequest.java
  │   │   │   ├── EmitirApoliceRequest.java
  │   │   │   ├── RegistrarSinistroRequest.java
  │   │   │   ├── AnalisarSinistroRequest.java
  │   │   │   ├── CriarProdutoRequest.java
  │   │   │   ├── AtualizarProdutoRequest.java
  │   │   │   ├── CadastrarCorretorRequest.java
  │   │   │   └── PagarParcelaRequest.java
  │   │   └── response/
  │   │       ├── package-info.java
  │   │       ├── CotacaoResponse.java
  │   │       ├── ApoliceResponse.java
  │   │       ├── SinistroResponse.java
  │   │       ├── ParcelaPremioResponse.java
  │   │       ├── ProdutoSeguroResponse.java
  │   │       ├── CorretorResponse.java
  │   │       └── ComissaoResponse.java
  │   ├── event/
  │   │   ├── package-info.java
  │   │   ├── ApoliceEmitidaEvent.java
  │   │   ├── PremioPagoEvent.java
  │   │   ├── SinistroAbertoEvent.java
  │   │   └── SinistroLiquidadoEvent.java
  │   ├── client/
  │   │   ├── package-info.java
  │   │   ├── SusepClient.java
  │   │   ├── ResseguradoraClient.java
  │   │   ├── ContaCorrenteClient.java
  │   │   ├── CreditClient.java
  │   │   └── ConsignadoClient.java
  │   ├── config/
  │   │   ├── package-info.java
  │   │   ├── SegurosHttpConfig.java
  │   │   ├── SegurosKafkaConfig.java
  │   │   └── SegurosSecurityConfig.java
  │   ├── service/
  │   │   ├── package-info.java
  │   │   ├── CotacaoSeguroService.java
  │   │   ├── EmissaoService.java
  │   │   ├── SinistroService.java
  │   │   ├── ParcelaService.java
  │   │   ├── ComissaoService.java
  │   │   └── ProdutoSeguroService.java
  │   └── controller/
  │       ├── package-info.java
  │       ├── CotacaoController.java
  │       ├── ApoliceController.java
  │       ├── SinistroController.java
  │       ├── ProdutoController.java
  │       └── ParceiroController.java
  ├── src/main/resources/
  │   ├── application.yml
  │   └── application-prod.yml
  └── src/test/java/com/aurix/platform/seguros/
      ├── controller/
      │   ├── package-info.java
      │   ├── CotacaoControllerTest.java
      │   ├── ApoliceControllerTest.java
      │   ├── SinistroControllerTest.java
      │   └── ProdutoControllerTest.java
      └── AurixSegurosApplicationTest.java
  ```

- [ ] **`pom.xml`**
  Copiar estrutura de `aurix-poupanca/pom.xml`, alterar:
  - `artifactId` → `aurix-seguros`
  - `name` → `AURIX Seguros`
  - `description` → `Modulo de seguros do AURIX`
  Dependências idênticas: `aurix-shared`, `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `spring-boot-starter-validation`, `spring-boot-starter-security`, `spring-boot-starter-actuator`, `spring-kafka`, `spring-boot-starter-data-redis`, `postgresql` (runtime), `h2` (test), `springdoc-openapi-starter-webmvc-ui` (2.2.0), `spring-boot-starter-test`, `spring-security-test`, `spring-kafka-test`, `testcontainers-junit-jupiter`, `testcontainers-postgresql`.

- [ ] **`AurixSegurosApplication.java`** — `@SpringBootApplication`, `@EnableScheduling`

- [ ] **`package-info.java` (root + cada subpacote)** — `@NullMarked` com `import org.jspecify.annotations.NullMarked`

- [ ] **`application.yml`**
  - `server.port: 8117`
  - `server.servlet.context-path: /api/seguros`
  - Datasource PostgreSQL, JPA/Hibernate ddl-auto update, Redis, Kafka (`aurix-seguros-group`), Actuator, logging
  - Props específicas: `aurix.seguros.prazo-validade-cotacao: 30` (dias)

- [ ] **`application-prod.yml`** — ddl-auto: validate, pool maior, logging INFO

### 1.2 Entidades

- [ ] **`entity/ProdutoSeguro.java`**
  `@Entity @Table(name = "produtos_seguro")`
  `Long id`, `String tenantId`, `String nome`, `String tipo` (VIDA, PRESTAMISTA, RESIDENCIAL, AUTO, EMPRESARIAL), `String coberturasBase` (JSON column), `String coberturasAdicionais` (JSON column), `BigDecimal premioMinimo`, `BigDecimal comissaoCorretor`, `Long tabelaAtuarialId`, `boolean ativo`, `LocalDateTime dataCriacao`, `LocalDateTime dataAtualizacao`
  `@PrePersist` / `@PreUpdate`
  Getters/setters manuais (sem Lombok)

- [ ] **`entity/CotacaoSeguro.java`**
  `@Entity @Table(name = "cotacoes_seguro")`
  `Long id`, `String tenantId`, `Long clienteId`, `Long produtoId`, `String coberturas` (JSON), `BigDecimal premioCalculado`, `LocalDate dataValidade`, `String status` (ATIVA, EXPIRADA, CONVERTIDA), `LocalDateTime dataCriacao`, `LocalDateTime dataAtualizacao`
  `@ManyToOne(fetch = LAZY) @JoinColumn(name = "produto_id", insertable = false, updatable = false) ProdutoSeguro produto` (opcional)

- [ ] **`entity/Apolice.java`**
  `@Entity @Table(name = "apolices")`
  `Long id`, `String tenantId`, `String numero`, `Long clienteId`, `Long cotacaoId`, `Long produtoId`, `String coberturasContratadas` (JSON), `BigDecimal premioTotal`, `String formaPagamento`, `LocalDate dataInicio`, `LocalDate dataFim`, `String status` (ATIVA, CANCELADA, VENCIDA), `String endosso`, `LocalDateTime dataCriacao`, `LocalDateTime dataAtualizacao`

- [ ] **`entity/Tomador.java`**
  `@Entity @Table(name = "tomadores")`
  `Long id`, `Long apoliceId`, `String nome`, `String documento`, `String tipoPessoa` (FISICA, JURIDICA), `String endereco`, `String contato`

- [ ] **`entity/ParcelaPremio.java`**
  `@Entity @Table(name = "parcelas_premio")`
  `Long id`, `Long apoliceId`, `int numero`, `BigDecimal valor`, `LocalDate dataVencimento`, `LocalDate dataPagamento`, `String status` (PENDENTE, PAGA, ATRASADA), `String tenantId`

- [ ] **`entity/Sinistro.java`**
  `@Entity @Table(name = "sinistros")`
  `Long id`, `Long apoliceId`, `Long clienteId`, `LocalDate dataOcorrencia`, `LocalDate dataComunicacao`, `String tipo`, `String descricao`, `BigDecimal valorSolicitado`, `BigDecimal valorAprovado`, `String status` (ABERTO, EM_ANALISE, APROVADO, NEGADO, LIQUIDADO), `String peritoResponsavel`, `String tenantId`, `LocalDateTime dataCriacao`, `LocalDateTime dataAtualizacao`

- [ ] **`entity/Comissao.java`**
  `@Entity @Table(name = "comissoes")`
  `Long id`, `Long apoliceId`, `Long corretorId`, `BigDecimal valor`, `BigDecimal percentual`, `LocalDate dataPagamento`, `String tenantId`

- [ ] **`entity/Corretor.java`**
  `@Entity @Table(name = "corretores")`
  `Long id`, `String tenantId`, `String nome`, `String documento`, `String susepCode`, `BigDecimal comissaoPadrao`, `boolean ativo`

### 1.3 Repositories

- [ ] **`repository/ProdutoSeguroRepository.java`** — `findByAtivoTrue()`, `findByTipoAndAtivoTrue(String tipo)`, `findByTenantId(String)`
- [ ] **`repository/CotacaoSeguroRepository.java`** — `findByClienteId(Long)`, `findByStatus(String)`, `findByDataValidadeBeforeAndStatus(LocalDate, String)`
- [ ] **`repository/ApoliceRepository.java`** — `findByClienteId(Long)`, `findByNumero(String)`, `findByStatus(String)`
- [ ] **`repository/TomadorRepository.java`** — `findByApoliceId(Long)`
- [ ] **`repository/ParcelaPremioRepository.java`** — `findByApoliceIdOrderByNumeroAsc(Long)`, `findByStatusAndDataVencimentoBefore(String, LocalDate)`
- [ ] **`repository/SinistroRepository.java`** — `findByApoliceId(Long)`, `findByClienteId(Long)`, `findByStatus(String)`
- [ ] **`repository/ComissaoRepository.java`** — `findByApoliceId(Long)`, `findByCorretorId(Long)`, `findByDataPagamentoBetween(LocalDate, LocalDate)`
- [ ] **`repository/CorretorRepository.java`** — `findByAtivoTrue()`, `findByDocumento(String)`, `findBySusepCode(String)`

### 1.4 DTOs — Requests

- [ ] **`dto/request/CotacaoRequest.java`**
  `@NotNull Long clienteId`, `@NotNull Long produtoId`, `String coberturas` (JSON), `@NotNull BigDecimal valorSegurado`, `int prazoMeses`

- [ ] **`dto/request/EmitirApoliceRequest.java`**
  `@NotNull Long cotacaoId`, `String formaPagamento`, `TomadorRequest tomador`

- [ ] **`dto/request/TomadorRequest.java`** (inner record ou classe separada)
  `@NotBlank String nome`, `@NotBlank String documento`, `String tipoPessoa`, `String endereco`, `String contato`

- [ ] **`dto/request/RegistrarSinistroRequest.java`**
  `@NotNull Long apoliceId`, `@NotNull Long clienteId`, `@NotNull LocalDate dataOcorrencia`, `String tipo`, `@NotBlank String descricao`, `@NotNull BigDecimal valorSolicitado`

- [ ] **`dto/request/AnalisarSinistroRequest.java`**
  `@NotNull String decisao` (APROVADO, NEGADO), `BigDecimal valorAprovado`, `String parecer`

- [ ] **`dto/request/CriarProdutoRequest.java`**
  `@NotBlank String nome`, `@NotBlank String tipo`, `String coberturasBase`, `String coberturasAdicionais`, `BigDecimal premioMinimo`, `BigDecimal comissaoCorretor`, `Long tabelaAtuarialId`

- [ ] **`dto/request/AtualizarProdutoRequest.java`** — mesmos campos opcionais

- [ ] **`dto/request/CadastrarCorretorRequest.java`**
  `@NotBlank String nome`, `@NotBlank String documento`, `@NotBlank String susepCode`, `BigDecimal comissaoPadrao`

- [ ] **`dto/request/PagarParcelaRequest.java`**
  `@NotNull Long parcelaId`, `String formaPagamento`

### 1.5 DTOs — Responses

- [ ] **`dto/response/CotacaoResponse.java`** — id, clienteId, produtoId, coberturas, premioCalculado, dataValidade, status, dataCriacao
- [ ] **`dto/response/ApoliceResponse.java`** — id, numero, clienteId, cotacaoId, produtoId, coberturasContratadas, premioTotal, formaPagamento, dataInicio, dataFim, status, endosso, dataCriacao
- [ ] **`dto/response/SinistroResponse.java`** — id, apoliceId, clienteId, dataOcorrencia, dataComunicacao, tipo, descricao, valorSolicitado, valorAprovado, status, peritoResponsavel
- [ ] **`dto/response/ParcelaPremioResponse.java`** — id, apoliceId, numero, valor, dataVencimento, dataPagamento, status
- [ ] **`dto/response/ProdutoSeguroResponse.java`** — id, nome, tipo, coberturasBase, coberturasAdicionais, premioMinimo, comissaoCorretor, tabelaAtuarialId, ativo
- [ ] **`dto/response/CorretorResponse.java`** — id, nome, documento, susepCode, comissaoPadrao, ativo
- [ ] **`dto/response/ComissaoResponse.java`** — id, apoliceId, corretorId, valor, percentual, dataPagamento

### 1.6 Events (Java records)

- [ ] **`event/ApoliceEmitidaEvent.java`**
  ```java
  public record ApoliceEmitidaEvent(
      Long apoliceId, String numero, Long clienteId, Long produtoId,
      BigDecimal premioTotal, LocalDate dataInicio, LocalDate dataFim,
      String tenantId
  ) {}
  ```

- [ ] **`event/PremioPagoEvent.java`**
  ```java
  public record PremioPagoEvent(
      Long parcelaId, Long apoliceId, int numero,
      BigDecimal valor, LocalDate dataPagamento, String tenantId
  ) {}
  ```

- [ ] **`event/SinistroAbertoEvent.java`**
  ```java
  public record SinistroAbertoEvent(
      Long sinistroId, Long apoliceId, Long clienteId,
      String tipo, BigDecimal valorSolicitado, LocalDate dataOcorrencia,
      String tenantId
  ) {}
  ```

- [ ] **`event/SinistroLiquidadoEvent.java`**
  ```java
  public record SinistroLiquidadoEvent(
      Long sinistroId, Long apoliceId, Long clienteId,
      BigDecimal valorAprovado, LocalDate dataLiquidacao, String tenantId
  ) {}
  ```

### 1.7 HTTP Clients (`@HttpExchange`)

- [ ] **`client/SusepClient.java`**
  ```java
  @HttpExchange("/api/externo/susep")
  public interface SusepClient {
      @PostExchange("/apolices/registrar")
      void registrarApolice(@RequestBody Object request);

      @PostExchange("/sinistros/comunicar")
      void comunicarSinistro(@RequestBody Object request);
  }
  ```

- [ ] **`client/ResseguradoraClient.java`**
  ```java
  @HttpExchange("/api/externo/resseguradora")
  public interface ResseguradoraClient {
      @PostExchange("/cessao")
      void cederRisco(@RequestBody Object request);
  }
  ```

- [ ] **`client/ContaCorrenteClient.java`**
  ```java
  @HttpExchange("/api/core/contas")
  public interface ContaCorrenteClient {
      @PostExchange("/{id}/debitar")
      void debitar(@PathVariable Long id, @RequestBody DebitoRequest request);

      @PostExchange("/{id}/creditar")
      void creditar(@PathVariable Long id, @RequestBody CreditoRequest request);

      record DebitoRequest(BigDecimal valor, String descricao) {}
      record CreditoRequest(BigDecimal valor, String descricao) {}
  }
  ```

- [ ] **`client/CreditClient.java`**
  ```java
  @HttpExchange("/api/credit")
  public interface CreditClient {
      @GetExchange("/contratos/{id}")
      Object obterContrato(@PathVariable Long id);
  }
  ```

- [ ] **`client/ConsignadoClient.java`**
  ```java
  @HttpExchange("/api/consignado")
  public interface ConsignadoClient {
      @GetExchange("/contratos/{id}")
      Object obterContrato(@PathVariable Long id);
  }
  ```

### 1.8 Configurações

- [ ] **`config/SegurosHttpConfig.java`** — `@Configuration`, `@EnableResilientMethods`, `@ImportHttpServices({SusepClient.class, ResseguradoraClient.class, ContaCorrenteClient.class, CreditClient.class, ConsignadoClient.class})`

- [ ] **`config/SegurosKafkaConfig.java`** — tópicos:
  - `seguros-apolice-emitida` (3 partições)
  - `seguros-premio-pago` (3 partições)
  - `seguros-sinistro-aberto` (3 partições)
  - `seguros-sinistro-liquidado` (3 partições)

- [ ] **`config/SegurosSecurityConfig.java`** — `@Configuration @EnableWebSecurity @Profile("!test")`
  SecurityFilterChain: CSRF disable, STATELESS sessions. Rotas:
  - `/actuator/**`, `/swagger-ui/**`, `/v3/api-docs/**` → permitAll
  - `/cotacoes/**`, `/apolices/**`, `/sinistros/**` → authenticated
  - `/produtos/**` e `/corretores/**` → authenticated (role ADMIN em futura iteração)

### 1.9 Services

- [ ] **`service/ProdutoSeguroService.java`**
  - `listarAtivos()` → lista produtos `ativo=true` do tenant
  - `listarPorTipo(String tipo)` → filtro por tipo
  - `criar(CriarProdutoRequest)` → salva entidade
  - `atualizar(Long id, AtualizarProdutoRequest)` → merge campos
  - Métodos `private` de mapeamento request→entity, entity→response

- [ ] **`service/CotacaoSeguroService.java`**
  - `criar(CotacaoRequest)` → valida produto ativo, calcula prêmio baseado em tabela atuarial (simplificado: valorSegurado * taxaBase / 100), cria `CotacaoSeguro` com status `ATIVA` e `dataValidade = hoje + prazoConfig`, publica evento (se houver)
  - `buscarPorId(Long id)` → findById or throw
  - `buscarPorCliente(Long clienteId)` → lista
  - Se prestamista: consulta `CreditClient` ou `ConsignadoClient` para obter dados do financiamento

- [ ] **`service/EmissaoService.java`**
  - `emitir(EmitirApoliceRequest)` → busca cotação, valida se `ATIVA` e não expirada, cria `Apolice` com número gerado, gera `ParcelaPremio`s conforme formaPagamento (avista=1 parcela, parcelado=divisão), cria `Tomador`, atualiza cotação para `CONVERTIDA`
  - Se formaPagamento débito → `ContaCorrenteClient.debitar()`
  - Se aplicável → `SusepClient.registrarApolice()`
  - Se aplicável → `ResseguradoraClient.cederRisco()`
  - Publica `ApoliceEmitidaEvent`
  - `cancelar(Long id)` → seta status CANCELADA, estorna se necessário

- [ ] **`service/SinistroService.java`**
  - `registrar(RegistrarSinistroRequest)` → cria `Sinistro` status `ABERTO`, publica `SinistroAbertoEvent`, notifica SUSEP
  - `analisar(Long id, AnalisarSinistroRequest)` → seta `EM_ANALISE` → `APROVADO`/`NEGADO`; se `APROVADO` -> define `valorAprovado`
  - `liquidar(Long id)` → se `APROVADO`, seta `LIQUIDADO`, credita via `ContaCorrenteClient.creditar()`, publica `SinistroLiquidadoEvent`, comunica SUSEP
  - `buscarPorId`, `buscarPorApolice`

- [ ] **`service/ParcelaService.java`**
  - `listarPorApolice(Long apoliceId)` → parcelas ordenadas
  - `pagar(PagarParcelaRequest)` → seta `dataPagamento` e status `PAGA`, debita conta corrente se aplicável, publica `PremioPagoEvent`
  - Job opcional: `@Scheduled` diário para detectar parcelas vencidas → atualiza status para ATRASADA

- [ ] **`service/ComissaoService.java`**
  - `calcularEAgendar(Long apoliceId)` → calcula comissão do corretor vinculado, cria `Comissao`
  - `listarPorCorretor(Long corretorId)` → histórico
  - `listarPorPeriodo(LocalDate inicio, LocalDate fim)` → relatório

### 1.10 Controllers

- [ ] **`controller/CotacaoController.java`** — `@RestController @RequestMapping("/cotacoes") @Tag(name = "Cotação")`
  - `POST /` → `CotacaoSeguroService.criar()`, retorna 201
  - `GET /{id}` → `CotacaoSeguroService.buscarPorId()`
  - `POST /{id}/emitir` → `EmissaoService.emitir()`, retorna a apólice criada

- [ ] **`controller/ApoliceController.java`** — `@RestController @RequestMapping("/apolices") @Tag(name = "Apólice")`
  - `GET /{id}` → buscar
  - `GET /cliente/{clienteId}` → listar do cliente
  - `PATCH /{id}/cancelar` → cancelar
  - `GET /{id}/parcelas` → listar parcelas
  - `POST /{id}/parcelas/pagar` → pagar parcela

- [ ] **`controller/SinistroController.java`** — `@RestController @RequestMapping("/sinistros") @Tag(name = "Sinistro")`
  - `POST /` → registrar
  - `GET /{id}` → buscar
  - `PATCH /{id}/analisar` → analisar (aprovar/negar)
  - `PATCH /{id}/liquidar` → liquidar
  - `GET /apolice/{apoliceId}` → sinistros da apólice

- [ ] **`controller/ProdutoController.java`** — `@RestController @RequestMapping("/produtos") @Tag(name = "Produto Seguro")`
  - `GET /` → listar ativos
  - `POST /` → criar (admin)
  - `PUT /{id}` → atualizar (admin)
  - `GET /{id}/tabelas-atuariais` → (placeholder, retorna info se aplicável)

- [ ] **`controller/ParceiroController.java`** — `@RestController @RequestMapping("/corretores") @Tag(name = "Corretores")`
  - `GET /` → listar corretores
  - `POST /` → cadastrar

### 1.11 Gateway Route + Parent POM

- [ ] **`apps/backend/aurix-gateway/src/main/resources/application.yml`** — adicionar rota:
  ```yaml
  - id: aurix-seguros
    uri: http://localhost:8117
    predicates:
      - Path=/api/seguros/**
    filters:
      - StripPrefix=0
  ```

- [ ] **`apps/backend/pom.xml`** — adicionar `<module>aurix-seguros</module>` em `<modules>`

### 1.12 Testes

- [ ] **`controller/CotacaoControllerTest.java`**
  - `@SpringBootTest(webEnvironment = RANDOM_PORT) @ActiveProfiles("test") @Import(CotacaoControllerTest.TestConfig.class)`
  - TestConfig: `SecurityFilterChain` permitAll, `KafkaTemplate` mock, HTTP clients mock
  - `deveCriarCotacao()` → POST /cotacoes, assert 201 + premioCalculado
  - `deveBuscarCotacaoPorId()` → POST + GET, assert 200
  - `deveRejeitarCotacaoSemProduto()` → POST com produtoId inválido, assert 4xx

- [ ] **`controller/ApoliceControllerTest.java`**
  - Fluxo completo: criar cotação → emitir → buscar apólice → parcelas
  - `deveEmitirApolice()` → POST cotacoes, POST emitir, assert 201 + status ATIVA
  - `deveCancelarApolice()` → emitir → cancelar, assert 204
  - `deveListarParcelas()` → emitir → GET parcelas, assert lista não vazia

- [ ] **`controller/SinistroControllerTest.java`**
  - `deveRegistrarSinistro()` → POST sinistros, assert 201 + status ABERTO
  - `deveAnalisarELiquidar()` → registrar → analisar (APROVADO) → liquidar, assert status LIQUIDADO
  - `deveNegarSinistro()` → registrar → analisar (NEGADO), assert status NEGADO

- [ ] **`controller/ProdutoControllerTest.java`**
  - `deveCriarEListarProdutos()` → POST + GET, assert 200 com lista
  - `deveAtualizarProduto()` → criar → atualizar, assert campos alterados

- [ ] **`AurixSegurosApplicationTest.java`** — smoke test: carrega contexto, verifica `applicationContext` não nulo

## 2. Ordem de Execução Recomendada

| Passo | O quê | Justificativa |
|-------|-------|---------------|
| 1 | Scaffold + POM + configs | Base para tudo |
| 2 | Entidades + Repositories | Dependência zero |
| 3 | DTOs + Events | Dependem de entidades |
| 4 | HTTP Clients + Config | Dependência externa |
| 5 | Services (Produto → Cotação → Emissão → Sinistro → Parcela → Comissão) | Ordem de dependência |
| 6 | Controllers | Consomem services |
| 7 | Gateway + Parent POM | Integração |
| 8 | Testes por controller | Verificação |
