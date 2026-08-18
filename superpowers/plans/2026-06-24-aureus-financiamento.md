# Plano de Implementação: `aurix-financiamento`

> Baseado no design aprovado em `docs/superpowers/specs/2026-06-24-products-design.md` (Seção 7, p.16-18)
> Porta: **8118** | Gateway: `/api/financiamento/**` | Depende de `aurix-core` (contas)

---

## 1. Scaffold

### 1.1 `apps/backend/aurix-financiamento/pom.xml`

Archetype: copiar `aurix-poupanca/pom.xml`, alterar `artifactId` e `description`.

```xml
<parent>
    <groupId>com.aurix.platform</groupId>
    <artifactId>aurix-platform</artifactId>
    <version>1.0.0</version>
</parent>

<artifactId>aurix-financiamento</artifactId>
<name>AURIX Financiamento</name>
<description>Modulo de financiamento de bens (imobiliario, veicular, outros)</description>
```

Dependências: `aurix-shared`, `spring-boot-starter-web`, `data-jpa`, `validation`, `security`, `actuator`, `kafka`, `data-redis`, `postgresql` (runtime), `h2` (test), `springdoc-openapi`, `spring-boot-starter-test`, `spring-security-test`, `spring-kafka-test`, `testcontainers-junit-jupiter`.

### 1.2 `AurixFinanciamentoApplication.java`

`com.aurix.platform.financiamento`, anotado com `@SpringBootApplication`, `@EnableScheduling`.

### 1.3 `package-info.java`

Um por pacote: `com.aurix.platform.financiamento`, `entity`, `repository`, `dto`, `event`, `client`, `config`, `service`, `controller`, `job`. Todos com `@NullMarked`.

### 1.4 `application.yml`

```yaml
server:
  port: 8118
  servlet:
    context-path: /api/financiamento
spring:
  application:
    name: aurix-financiamento
  datasource: ... (mesmo padrao poupanca)
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    open-in-view: false
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: aurix-financiamento-group
      auto-offset-reset: earliest
    producer:
      acks: all
      retries: 3
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
aurix:
  financiamento:
    taxa-sac: 0.0099       # taxa padrao SAC (mensal)
    taxa-price: 0.0112     # taxa padrao Price (mensal)
    taxa-sacre: 0.0105     # taxa padrao SACRE (mensal)
    cet-taxa: 0.0025       # taxa adicional CET
    iof-diario: 0.000041
    iof-adicional: 0.0038
  security:
    jwt:
      secret: "aurix-jwt-secret-key-2024"
      expiration: 86400000
---
spring:
  config:
    activate:
      on-profile: dev
  jpa:
    hibernate:
      ddl-auto: create-drop
---
spring:
  config:
    activate:
      on-profile: prod
  jpa:
    hibernate:
      ddl-auto: validate
```

### 1.5 `application-prod.yml` (diferenciais de produção)

```yaml
spring:
  jpa:
    show-sql: false
  datasource:
    hikari:
      maximum-pool-size: 30
logging:
  level:
    com.aurix.platform: INFO
```

---

## 2. Config

### 2.1 `FinanciamentoHttpConfig.java`

`@Configuration`, `@EnableResilientMethods`, `@ImportHttpServices({ContaCorrenteClient.class, CartorioRgiClient.class, DetranClient.class, BacenClient.class})`.

### 2.2 `FinanciamentoKafkaConfig.java`

Tópicos (todos `NewTopic`, 3 partições, 1 réplica):

| Constante | Nome |
|-----------|------|
| `TOPICO_SIMULACAO_REALIZADA` | `financiamento-simulacao-realizada` |
| `TOPICO_CONTRATO_ASSINADO` | `financiamento-contrato-assinado` |
| `TOPICO_PARCELA_PAGA` | `financiamento-parcela-paga` |
| `TOPICO_CONTRATO_LIQUIDADO` | `financiamento-contrato-liquidado` |
| `TOPICO_GARANTIA_REGISTRADA` | `financiamento-garantia-registrada` |

### 2.3 `FinanciamentoSecurityConfig.java`

```java
@Configuration @EnableWebSecurity @Profile("!test")
public class FinanciamentoSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/**", "/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers(HttpMethod.POST, "/simulacoes/**").authenticated()
                .requestMatchers("/contratos/**").authenticated()
                .requestMatchers("/parcelas/**").authenticated()
                .requestMatchers("/bens/**").authenticated()
                .requestMatchers("/garantias/**").authenticated()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```

---

## 3. Entities (5 classes)

### 3.1 `ContratoFinanciamento`

`@Entity @Table(name = "contratos_financiamento")`

| Campo | Tipo | Anotações |
|-------|------|-----------|
| id | Long | `@Id @GeneratedValue(IDENTITY)` |
| tenantId | String | `@NotBlank @Column(length=50)` |
| clienteId | Long | `@NotNull` |
| tipo | `TipoFinanciamento` | `@Enumerated(STRING) @Column(length=20)` — IMOBILIARIO, VEICULAR, OUTROS_BENS |
| valorFinanciado | BigDecimal | `@NotNull @Column(precision=18, scale=2)` |
| valorEntrada | BigDecimal | `@Column(precision=18, scale=2)` default ZERO |
| taxaJuros | BigDecimal | `@NotNull @Column(precision=7, scale=5)` |
| prazoMeses | int | |
| sistemaAmortizacao | `SistemaAmortizacao` | `@Enumerated(STRING) @Column(length=10)` — SAC, PRICE, SACRE |
| valorParcela | BigDecimal | `@Column(precision=18, scale=2)` |
| saldoDevedor | BigDecimal | `@NotNull @Column(precision=18, scale=2)` |
| dataContratacao | LocalDate | `@NotNull` |
| dataPrimeiraParcela | LocalDate | `@NotNull` |
| dataVencimento | LocalDate | |
| status | `StatusContrato` | `@Enumerated(STRING) @Column(length=20)` — PROPOSTA, ASSINADO, ATIVO, LIQUIDADO, INADIMPLENTE |
| dataCriacao | LocalDateTime | `@Column(updatable=false)` + `@PrePersist` |
| dataAtualizacao | LocalDateTime | + `@PreUpdate` |

Enums internos: `TipoFinanciamento`, `SistemaAmortizacao`, `StatusContrato`.

### 3.2 `ParcelaFinanciamento`

`@Entity @Table(name = "parcelas_financiamento")`

| Campo | Tipo |
|-------|------|
| id | Long |
| contratoId | Long (`@NotNull`) |
| numero | int |
| dataVencimento | LocalDate |
| valorParcela | BigDecimal |
| valorAmortizacao | BigDecimal |
| valorJuros | BigDecimal |
| valorSaldoDevolver | BigDecimal |
| dataPagamento | LocalDate (nullable) |
| status | `StatusParcela` (PENDENTE, PAGA, ATRASADA, CANCELADA) |
| dataCriacao | LocalDateTime |

### 3.3 `BemFinanciado`

`@Entity @Table(name = "bens_financiados")`

| Campo | Tipo |
|-------|------|
| id | Long |
| contratoId | Long |
| tipo | `TipoBem` (IMOVEL, VEICULO, EQUIPAMENTO) |
| descricao | String |
| valorAvaliacao | BigDecimal |
| chassi | String (nullable) |
| placa | String (nullable) |
| matriculaRGI | String (nullable) |
| registroGarantia | String (nullable) |
| dataCriacao | LocalDateTime |

### 3.4 `Garantia`

`@Entity @Table(name = "garantias")`

| Campo | Tipo |
|-------|------|
| id | Long |
| contratoId | Long |
| bemId | Long |
| tipo | `TipoGarantia` (ALIENACAO_FIDUCIARIA, HIPOTECA, PENHOR) |
| valor | BigDecimal |
| dataRegistro | LocalDate |
| dataBaixa | LocalDate (nullable) |
| status | `StatusGarantia` (ATIVA, LIBERADA) |
| orgaoRegistro | String (RGI, DETRAN, CARTORIO) |
| dataCriacao | LocalDateTime |

### 3.5 `SimulacaoFinanciamento`

`@Entity @Table(name = "simulacoes_financiamento")`

| Campo | Tipo |
|-------|------|
| id | Long |
| tenantId | String |
| clienteId | Long |
| tipo | `TipoFinanciamento` |
| valorFinanciado | BigDecimal |
| prazoMeses | int |
| taxaJuros | BigDecimal |
| sistemaAmortizacao | `SistemaAmortizacao` |
| valorParcela | BigDecimal |
| tabelaSAC | `@Column(columnDefinition="TEXT")` — JSON |
| tabelaPrice | `@Column(columnDefinition="TEXT")` — JSON |
| dataSimulacao | LocalDateTime |

---

## 4. Repositories (5 interfaces)

Extendem `JpaRepository<Entity, Long>`.

- `ContratoFinanciamentoRepository` — `findByClienteId`, `findByTenantIdAndStatus`
- `ParcelaFinanciamentoRepository` — `findByContratoIdOrderByNumero`, `findByContratoIdAndStatus`, `findParcelasVencidas(@Param("data") LocalDate data)`
- `BemFinanciadoRepository` — `findByContratoId`
- `GarantiaRepository` — `findByContratoId`, `findByStatus`
- `SimulacaoFinanciamentoRepository` — `findByClienteIdOrderByDataSimulacaoDesc`

---

## 5. DTOs

### 5.1 Request DTOs

| Classe | Campos |
|--------|--------|
| `SimulacaoRequest` | tipo, valorFinanciado, prazoMeses, taxaJuros, sistemaAmortizacao |
| `CriarContratoRequest` | clienteId, contaCorrenteId, tipo, valorFinanciado, valorEntrada, prazoMeses, sistemaAmortizacao, bem (BemRequest), garantia (GarantiaRequest) |
| `BemRequest` | tipo, descricao, valorAvaliacao, chassi, placa, matriculaRGI |
| `GarantiaRequest` | tipo, valor, orgaoRegistro |
| `PagarParcelaRequest` | parcelaId (ou lista de ids), valor |
| `RenegociarRequest` | novosPrazos, novaTaxa |
| `AtualizarTaxaRequest` | sistemaAmortizacao, taxa |
| `LiberarGarantiaRequest` | dataBaixa |

### 5.2 Response DTOs

| Classe | Campos |
|--------|--------|
| `SimulacaoResponse` | id, tipo, valorFinanciado, prazoMeses, taxaJuros, sistemaAmortizacao, valorParcela, cet, tabelaSAC (List<LinhaTabela>), tabelaPrice (List<LinhaTabela>), dataSimulacao |
| `ContratoResponse` | id, clienteId, tipo, valorFinanciado, valorEntrada, taxaJuros, prazoMeses, sistemaAmortizacao, valorParcela, saldoDevedor, dataContratacao, dataPrimeiraParcela, status, bens (List<BemResponse>), garantias (List<GarantiaResponse>), dataCriacao |
| `ParcelaResponse` | id, contratoId, numero, dataVencimento, valorParcela, valorAmortizacao, valorJuros, valorSaldoDevolver, dataPagamento, status |
| `BemResponse` | id, tipo, descricao, valorAvaliacao, chassi, placa, matriculaRGI, registroGarantia |
| `GarantiaResponse` | id, tipo, valor, dataRegistro, dataBaixa, status, orgaoRegistro |
| `LinhaTabela` (record) | numero, valorParcela, amortizacao, juros, saldoDevedor |
| `ContratoResumoResponse` | id, tipo, valorFinanciado, saldoDevedor, status, dataContratacao (para listagem) |
| `TaxasResponse` | taxaSAC, taxaPrice, taxaSACRE, cetTaxa |

---

## 6. Events (5 records)

### 6.1 `SimulacaoRealizadaEvent`

```java
public record SimulacaoRealizadaEvent(
    Long id, Long clienteId, String tipo,
    BigDecimal valorFinanciado, int prazoMeses,
    String sistemaAmortizacao, LocalDateTime dataSimulacao, String tenantId
) {}
```

### 6.2 `ContratoFinanciamentoAssinadoEvent`

```java
public record ContratoFinanciamentoAssinadoEvent(
    Long id, Long clienteId, Long contaCorrenteId,
    String tipo, BigDecimal valorFinanciado,
    int prazoMeses, BigDecimal taxaJuros,
    LocalDate dataContratacao, String tenantId
) {}
```

### 6.3 `ParcelaPagaEvent`

```java
public record ParcelaPagaEvent(
    Long parcelaId, Long contratoId, int numero,
    BigDecimal valorPago, LocalDate dataPagamento, String tenantId
) {}
```

### 6.4 `ContratoLiquidadoEvent`

```java
public record ContratoLiquidadoEvent(
    Long contratoId, Long clienteId,
    BigDecimal valorQuitado, LocalDate dataLiquidacao, String tenantId
) {}
```

### 6.5 `GarantiaRegistradaEvent`

```java
public record GarantiaRegistradaEvent(
    Long garantiaId, Long contratoId, Long bemId,
    String tipo, String orgaoRegistro,
    LocalDate dataRegistro, String tenantId
) {}
```

---

## 7. HTTP Clients (4 interfaces)

### 7.1 `ContaCorrenteClient`

`@HttpExchange("/api/core/contas")` — mesmo padrao de `aurix-poupanca`:

```java
@PostExchange("/{id}/debitar")
void debitar(@PathVariable Long id, @RequestBody DebitoRequest request);
@PostExchange("/{id}/creditar")
void creditar(@PathVariable Long id, @RequestBody CreditoRequest request);

record DebitoRequest(BigDecimal valor, String descricao) {}
record CreditoRequest(BigDecimal valor, String descricao) {}
```

### 7.2 `CartorioRgiClient`

`@HttpExchange("/api/rgi")`

```java
@PostExchange("/registro")
RegistroResponse registrarGarantia(@RequestBody RegistroGarantiaRequest request);
@GetExchange("/consulta/{matricula}")
SituacaoRegistro consultar(@PathVariable String matricula);
```

### 7.3 `DetranClient`

`@HttpExchange("/api/detran")`

```java
@PostExchange("/garantias")
DetranResponse registrarGarantia(@RequestBody DetranGarantiaRequest request);
@GetExchange("/veiculos/{placa}")
DadosVeiculo consultarVeiculo(@PathVariable String placa);
```

### 7.4 `BacenClient`

`@HttpExchange("/api/bacen")`

```java
@GetExchange("/taxas/tr")
BigDecimal consultarTR();
@GetExchange("/taxas/selic")
BigDecimal consultarSelic();
```

---

## 8. Services

### 8.1 `SimulacaoService` (`AmortizacaoCalculator`)

Responsável pelo cálculo das tabelas de amortização. Injeção de `SimulacaoFinanciamentoRepository` + `KafkaTemplate`.

Métodos principais:

```java
SimulacaoResponse simular(SimulacaoRequest request)
```

Internamente delega para métodos de cálculo:

```java
List<LinhaTabela> calcularSAC(BigDecimal valor, int prazo, BigDecimal taxaMensal)
List<LinhaTabela> calcularPrice(BigDecimal valor, int prazo, BigDecimal taxaMensal)
List<LinhaTabela> calcularSACRE(BigDecimal valor, int prazo, BigDecimal taxaMensal)
```

**Fórmulas**:
- **SAC**: `amortizacao = valor / prazo` (constante); `juros = saldoDevedor * taxa`; `parcela = amortizacao + juros`
- **Price**: `parcela = valor * [taxa * (1+taxa)^prazo] / [(1+taxa)^prazo - 1]`; `juros = saldoDevedor * taxa`; `amortizacao = parcela - juros`
- **SACRE**: prestação recalculada a cada 12 meses (média SAC)

Calcula CET = valorParcela * prazo * (1 + taxaCet) — valorFinanciado.

Persiste `SimulacaoFinanciamento` com as tabelas em JSON. Publica `SimulacaoRealizadaEvent`.

### 8.2 `ContratoFinanciamentoService`

Métodos:

```java
ContratoResponse contratar(CriarContratoRequest request)
ContratoResponse buscarPorId(Long id)
List<ContratoResumoResponse> listarPorCliente(Long clienteId)
void liquidar(Long id)
ContratoResponse renegociar(Long id, RenegociarRequest request)
```

Fluxo do `contratar`:
1. Valida dados do cliente (`aurix-core` via `ContaCorrenteClient`?)
2. Cria `ContratoFinanciamento` com status `PROPOSTA`, `saldoDevedor = valorFinanciado`
3. Gera `ParcelaFinanciamento`s (parcelas futuras) chamando `AmortizacaoCalculator`
4. Cria `BemFinanciado` e `Garantia` com status `ATIVA`
5. Registra garantia via `CartorioRgiClient` ou `DetranClient` conforme tipo
6. Credita `valorFinanciado` na conta corrente via `ContaCorrenteClient.creditar()`
7. Altera status para `ASSINADO`, depois `ATIVO`
8. Publica `ContratoFinanciamentoAssinadoEvent`, `GarantiaRegistradaEvent`

Fluxo `liquidar`:
1. Calcula saldo devedor atual
2. Debita saldo devedor da conta corrente
3. Status → `LIQUIDADO`, atualiza `ParcelaFinanciamento` restantes → `CANCELADA`
4. Libera garantias
5. Publica `ContratoLiquidadoEvent`

### 8.3 `ParcelaService`

Métodos:

```java
List<ParcelaResponse> listarParcelas(Long contratoId)
void pagarParcela(Long contratoId, PagarParcelaRequest request)
```

Fluxo `pagarParcela`:
1. Calcula valor devido (pode incluir multa/juros de mora se atrasada)
2. Debita da conta corrente via `ContaCorrenteClient.debitar()`
3. Atualiza `ParcelaFinanciamento` → status `PAGA`, `dataPagamento`, `valorSaldoDevolver`
4. Atualiza `saldoDevedor` no `ContratoFinanciamento`
5. Publica `ParcelaPagaEvent`

### 8.4 `GarantiaService`

Métodos:

```java
GarantiaResponse registrar(GarantiaRequest request, Long contratoId, Long bemId)
void liberar(Long id, LiberarGarantiaRequest request)
List<GarantiaResponse> listarPorContrato(Long contratoId)
```

Registro: chama `CartorioRgiClient` (para imóveis) ou `DetranClient` (para veículos), depende do `orgaoRegistro`.

### 8.5 `AmortizacaoService` (cálculos puros)

Classe utilitária `@Service` ou package-private com métodos estáticos para SAC, Price, SACRE. Usada por `SimulacaoService` e `ContratoFinanciamentoService`.

```java
public static BigDecimal calcularParcelaPrice(BigDecimal valor, int prazo, BigDecimal taxa)
public static List<LinhaTabela> gerarTabelaSAC(BigDecimal valor, int prazo, BigDecimal taxa)
public static List<LinhaTabela> gerarTabelaPrice(BigDecimal valor, int prazo, BigDecimal taxa)
public static List<LinhaTabela> gerarTabelaSACRE(BigDecimal valor, int prazo, BigDecimal taxa)
public static BigDecimal calcularCet(BigDecimal valorParcela, int prazo, BigDecimal valorFinanciado, BigDecimal taxaCet)
```

---

## 9. Controllers

### 9.1 `SimulacaoController` — `@RequestMapping("/simulacoes")`

| Método | Endpoint | Response |
|--------|----------|----------|
| POST / | `simular(@Valid @RequestBody SimulacaoRequest)` | `201 SimulacaoResponse` |
| GET /{id} | `obterSimulacao(@PathVariable Long id)` | `200 SimulacaoResponse` |

### 9.2 `ContratoController` — `@RequestMapping("/contratos")`

| Método | Endpoint | Response |
|--------|----------|----------|
| POST / | `contratar(@Valid @RequestBody CriarContratoRequest)` | `201 ContratoResponse` |
| GET /{id} | `buscar(@PathVariable Long id)` | `200 ContratoResponse` |
| GET /cliente/{clienteId} | `listar(@PathVariable Long clienteId)` | `200 List<ContratoResumoResponse>` |
| PATCH /{id}/liquidar | `liquidar(@PathVariable Long id)` | `204` |
| PATCH /{id}/renegociar | `renegociar(@PathVariable Long id, @RequestBody RenegociarRequest)` | `200 ContratoResponse` |

### 9.3 `ParcelaController` — `@RequestMapping("/contratos/{contratoId}/parcelas")`

| Método | Endpoint | Response |
|--------|----------|----------|
| GET / | `listarParcelas(@PathVariable Long contratoId)` | `200 List<ParcelaResponse>` |
| POST /pagar | `pagarParcela(@PathVariable Long contratoId, @RequestBody PagarParcelaRequest)` | `200` |

### 9.4 `GarantiaController` — `@RequestMapping("/garantias")`

| Método | Endpoint | Response |
|--------|----------|----------|
| POST / | `registrar(@Valid @RequestBody GarantiaRequest)` | `201 GarantiaResponse` |
| PATCH /{id}/liberar | `liberar(@PathVariable Long id, @RequestBody LiberarGarantiaRequest)` | `204` |

Também `BemController` opcional em `/bens/{contratoId}`:

| GET /{contratoId} | `listarBens(@PathVariable Long contratoId)` | `200 List<BemResponse>` |

### 9.5 `AdminController` — `@RequestMapping("/admin")`

| Método | Endpoint | Response |
|--------|----------|----------|
| GET /taxas | `listarTaxas()` | `200 TaxasResponse` |
| PUT /taxas | `atualizarTaxas(@Valid @RequestBody AtualizarTaxaRequest)` | `200 TaxasResponse` |

---

## 10. Jobs

### 10.1 `ProcessamentoParcelasJob`

`@Scheduled(cron = "0 0 3 * * ?")` (diário 03:00)

- Busca `ContratoFinanciamento` com status `ATIVO` e `dataVencimento` <= hoje
- Para contratos com parcelas `PENDENTES` vencidas → status `ATRASADA`
- Se atraso > 90 dias → status `INADIMPLENTE`
- Gera novas parcelas futuras se ainda não geradas (para contratos novos)

### 10.2 `AtualizacaoGarantiasJob`

`@Scheduled(cron = "0 0 4 * * ?")` (diário 04:00)

- Consulta `CartorioRgiClient` / `DetranClient` para verificar status de garantias pendentes
- Atualiza `Garantia.status` se retornou confirmação de registro

---

## 11. Testes

### 11.1 `SimulacaoControllerTest`

```java
@SpringBootTest(classes = AurixFinanciamentoApplication.class, webEnvironment = RANDOM_PORT)
@ActiveProfiles("test")
@Import(SimulacaoControllerTest.TestConfig.class)
```

Testes:
- `deveSimularSAC()` — POST `/simulacoes` com `sistemaAmortizacao: SAC`, verifica `201`, tabela com amortização constante e juros decrescentes
- `deveSimularPrice()` — POST `/simulacoes` com `sistemaAmortizacao: PRICE`, verifica `201`, parcelas constantes
- `deveValidarCamposObrigatorios()` — POST sem `valorFinanciado` → `400`
- `deveBuscarSimulacao()` — POST + GET /{id} → `200`

### 11.2 `ContratoControllerTest`

- `deveContratarFinanciamento()` — POST `/contratos` → `201`, verifica contrato criado com parcelas
- `deveBuscarContrato()` — GET /{id} → `200`
- `deveLiquidarContrato()` — PATCH /{id}/liquidar → `204`, verifica status
- `deveListarPorCliente()` — GET /cliente/{id} → `200`

### 11.3 `ParcelaControllerTest`

- `deveListarParcelas()` — GET /contratos/{id}/parcelas → `200`
- `devePagarParcela()` — POST /contratos/{id}/parcelas/pagar → `200`

### 11.4 `GarantiaControllerTest`

- `deveRegistrarGarantia()` — POST /garantias → `201`
- `deveLiberarGarantia()` → `204`

### 11.5 `AdminControllerTest`

- `deveListarTaxas()` — GET /admin/taxas → `200`
- `deveAtualizarTaxas()` — PUT /admin/taxas → `200`

### 11.6 Teste de Cálculo SAC vs Price (unitário)

Teste para `AmortizacaoService` (sem Spring, JUnit puro):

```java
class AmortizacaoCalculatorTest {

    @Test
    void calcularSAC_amortizacaoConstante() {
        var tabela = AmortizacaoService.gerarTabelaSAC(
            BigDecimal.valueOf(100000), 12, new BigDecimal("0.01"));
        // amortizacao = 100000/12 = 8333.33... (constante em todas as linhas)
        assertThat(tabela.get(0).amortizacao()).isEqualByComparingTo(tabela.get(1).amortizacao());
        assertThat(tabela.get(0).juros()).isGreaterThan(tabela.get(1).juros()); // decrescente
        assertThat(tabela.get(tabela.size()-1).saldoDevedor()).isEqualByComparingTo(BigDecimal.ZERO);
    }

    @Test
    void calcularPrice_parcelaConstante() {
        var tabela = AmortizacaoService.gerarTabelaPrice(
            BigDecimal.valueOf(100000), 12, new BigDecimal("0.01"));
        assertThat(tabela.get(0).valorParcela()).isEqualByComparingTo(tabela.get(1).valorParcela());
        assertThat(tabela.get(0).amortizacao()).isLessThan(tabela.get(1).amortizacao()); // crescente
        assertThat(tabela.get(tabela.size()-1).saldoDevedor()).isEqualByComparingTo(BigDecimal.ZERO);
    }

    @Test
    void calcularPrice_valorParcelaConhecido() {
        // Para 100000, 12 meses, 1%: PMT = 100000 * [0.01*(1.01^12)] / [(1.01^12)-1]
        var tabela = AmortizacaoService.gerarTabelaPrice(
            BigDecimal.valueOf(100000), 12, new BigDecimal("0.01"));
        // valor esperado: ~8,884.88
        assertThat(tabela.get(0).valorParcela()).isEqualByComparingTo(
            new BigDecimal("8884.88").setScale(2, RoundingMode.HALF_EVEN));
    }
}
```

### 11.7 Config de Teste (`TestConfig` interna)

```java
@TestConfiguration
@EnableWebSecurity
static class TestConfig {
    @Bean
    public SecurityFilterChain testFilterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
        return http.build();
    }

    @Bean @Primary @SuppressWarnings("unchecked")
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return Mockito.mock(KafkaTemplate.class);
    }
}
```

---

## 12. Integração com a Plataforma

### 12.1 Gateway (`infra/gateway/application.yml`)

```yaml
- id: aurix-financiamento
  uri: http://localhost:8118
  predicates:
    - Path=/api/financiamento/**
  filters:
    - StripPrefix=0
```

### 12.2 Parent POM (`apps/backend/pom.xml`)

Adicionar `<module>aurix-financiamento</module>` na lista de módulos.

### 12.3 OpenAPI Spec

Após implementação, adicionar via springdoc-openapi:
- Tag `Financiamento`
- Paths `/api/financiamento/simulacoes`, `/api/financiamento/contratos`, `/api/financiamento/contratos/{contratoId}/parcelas`, `/api/financiamento/bens`, `/api/financiamento/garantias`, `/api/financiamento/taxas`
- Schemas: `SimulacaoRequest`, `SimulacaoResponse`, `CriarContratoRequest`, `ContratoResponse`, `ParcelaResponse`, `BemResponse`, `GarantiaResponse`, `LinhaTabela`, `TaxasResponse`

---

## 13. Ordem de Implementação Sugerida

| Fase | O quê |
|------|-------|
| 1 | Scaffold (pom.xml, application class, application.yml, package-info.java) |
| 2 | `AmortizacaoService` + testes unitários de cálculo SAC/Price/SACRE |
| 3 | Entities + Repositories |
| 4 | DTOs (request/response) |
| 5 | Events + Kafka config |
| 6 | HTTP Clients + HttpConfig |
| 7 | `SimulacaoService` + `SimulacaoController` + teste |
| 8 | `ContratoFinanciamentoService` + `ContratoController` + teste |
| 9 | `ParcelaService` + `ParcelaController` + teste |
| 10 | `GarantiaService` + `GarantiaController` + `BemController` + teste |
| 11 | `AdminController` + teste |
| 12 | Jobs (`ProcessamentoParcelasJob`, `AtualizacaoGarantiasJob`) |
| 13 | Security config |
| 14 | Gateway route + parent POM + OpenAPI update |
