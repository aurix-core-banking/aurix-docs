# aurix-cartoes — Cartões de Crédito/Débito — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to dispatch each task below as an independent subagent. Tasks 1–13 are independent in dependency groups as annotated. Run `mvn clean install -DskipTests -pl aurix-cartoes -am` after all tasks. Run `mvn test -pl aurix-cartoes` to verify. Gate: all `mvn test` green, `mvn pmd:check spotbugs:check checkstyle:check` pass.

**Goal:** Expand the existing `aurix-cartoes` module with partner-engine configuration (bandeira/adquirente), complete issuer authorization flow, monthly billing job, Kafka events, HTTP clients, full API surface (21 endpoints), and controller tests.

**Architecture:** Motor de cartões configurável por bandeira e adquirente. Suporta bandeiras parceiras (Visa, Mastercard, Elo) e private label (processamento interno). Adquirentes parceiros (Rede, Stone, GetNet) e own-acquirer. Antifraude integrado com `aurix-ml`. Faturamento mensal via `@Scheduled` job. Eventos Kafka para cada fluxo. Conta corrente integrada via `@HttpExchange` `ContaCorrenteClient` para débito de faturas.

**Tech Stack:** Spring Boot 4.1.0, Java 25, Spring Data JPA, PostgreSQL, H2 (test), Kafka, Redis, Spring Security, Springdoc OpenAPI 2.2.0, JSpecify `@NullMarked`, `@HttpExchange` + `@ImportHttpServices`, `@Retryable` (spring-resilience), Testcontainers.

## Global Constraints

1. No Lombok — manual getters/setters/constructors (`@SuppressWarnings("all")` delombok style as existing Cartao.java)
2. `@HttpExchange` + `@ImportHttpServices` for HTTP clients (not Feign)
3. `@Retryable` from `org.springframework.resilience.annotation` (not spring-retry)
4. `@NullMarked` package-level null safety via JSpecify in all packages
5. `jakarta.*` packages
6. `@EnableResilientMethods` for retry support on HTTP config
7. Test: `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `RestTemplate` with `@LocalServerPort`
8. H2 for repo tests, Testcontainers for integration
9. Kafka events fire-and-forget with try-catch
10. Entity base: extends `com.aurix.platform.shared.entity.BaseEntity`
11. Table schema: `aurix`
12. Port: 8115 (currently 8103 — change in application.yml)
13. Gateway route: `/api/cartoes/**` -> `http://localhost:8115`, StripPrefix=0
14. Controller base path: `/api/cartoes`
15. Follow existing patterns from `aurix-poupanca` for service/controller/dto/event/client/config structure
16. API design per Section 4.3 of design spec (21 endpoints)
17. Existing Cartao, Fatura, TransacaoCartao entities must be preserved and expanded (add fields, not removed)

---

## Task 1 — pom.xml: Add missing dependencies

**Dependency:** NONE (can run first)

**Files:**
- `apps/backend/aurix-cartoes/pom.xml` — modify

**Steps:**
1. Read `apps/backend/aurix-cartoes/pom.xml`
2. Add these dependencies (between existing `spring-boot-starter-test` and closing `</dependencies>`):
   - `spring-boot-starter-data-redis`
   - `spring-kafka`
   - `h2` (test scope)
   - `spring-kafka-test` (test scope)
   - `testcontainers-junit-jupiter` (test scope)
   - `testcontainers-postgresql` (test scope)
3. Also add `aurix-organization` dependency for organization entity access
4. Run `mvn clean install -DskipTests -pl aurix-cartoes -am` to verify POM compiles

## Task 2 — application.yml: Update config for port 8115, Kafka, Redis, Security

**Dependency:** NONE (can run in parallel with Task 1)

**Files:**
- `apps/backend/aurix-cartoes/src/main/resources/application.yml` — modify

**Changes:**
- server.port: 8103 -> 8115
- Add Kafka producer/consumer config (same pattern as aurix-poupanca)
- Add Redis config
- Add JPA/Hibernate SQL logging
- Add JHipster-style multi-profile (dev/prod)
- Add security JWT config
- Add actuator endpoints

---

## Task 3 — package-info.java files + directory scaffold

**Dependency:** NONE (can run in parallel with Tasks 1-2)

**Files to CREATE** (each with `@NullMarked`):
- `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/package-info.java`
- `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/entity/package-info.java`
- `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/repository/package-info.java`
- `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/dto/package-info.java`
- `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/event/package-info.java`
- `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/client/package-info.java`
- `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/config/package-info.java`
- `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/service/package-info.java`
- `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/controller/package-info.java`
- `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/job/package-info.java`

Each file content:
```java
@NullMarked
package com.aurix.platform.cartoes.<subpackage>;

import org.jspecify.annotations.NullMarked;
```

Also create empty directories:
- `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/dto/`
- `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/event/`
- `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/client/`
- `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/config/`
- `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/job/`
- `apps/backend/aurix-cartoes/src/test/java/com/aurix/platform/cartoes/controller/`

---

## Task 4 — New entities: ProdutoCartao, LimiteCartao, LancamentoFatura, ParceiroBandeira, ParceiroAdquirente

**Dependency:** NONE (can run in parallel with Tasks 1-3)

### ProdutoCartao

```java
package com.aurix.platform.cartoes.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.*;
import java.math.BigDecimal;

@Entity
@Table(name = "produtos_cartao", schema = "aurix")
public class ProdutoCartao extends BaseEntity {
    private String nome;
    private String bandeira;
    private String adquirente;
    private BigDecimal anuidade;
    private BigDecimal taxaJuros;
    private BigDecimal taxaMora;
    private BigDecimal limiteMinimo;
    private BigDecimal limiteMaximo;
    private String programaPontos;
    private Boolean ativo = true;

    public enum Bandeira { VISA, MASTERCARD, ELO, PRIVATE_LABEL }
    public enum Adquirente { REDE, STONE, GETNET, OWN_ACQUIRER }

    // getters/setters/construtores/equals/hashCode/toString — @SuppressWarnings("all") delombok style
}
```

### LimiteCartao

```java
package com.aurix.platform.cartoes.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "limites_cartao", schema = "aurix")
public class LimiteCartao extends BaseEntity {
    private Long cartaoId;
    private BigDecimal limiteTotal;
    private BigDecimal limiteDisponivel;
    private BigDecimal limiteUtilizado;
    private LocalDateTime dataAtualizacao;

    // delombok
}
```

### LancamentoFatura

```java
package com.aurix.platform.cartoes.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "lancamentos_fatura", schema = "aurix")
public class LancamentoFatura extends BaseEntity {
    private Long faturaId;
    private String descricao;
    private BigDecimal valor;
    private LocalDateTime dataLancamento;
    private String categoria;

    // delombok
}
```

### ParceiroBandeira

```java
package com.aurix.platform.cartoes.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.*;

@Entity
@Table(name = "parceiros_bandeira", schema = "aurix")
public class ParceiroBandeira extends BaseEntity {
    private String nome; // VISA, MASTERCARD, ELO, PRIVATE_LABEL
    private String tipoEndpoint;
    @Column(columnDefinition = "JSONB")
    private String config;
    private Boolean ativo = true;

    // delombok
}
```

### ParceiroAdquirente

```java
package com.aurix.platform.cartoes.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.*;

@Entity
@Table(name = "parceiros_adquirente", schema = "aurix")
public class ParceiroAdquirente extends BaseEntity {
    private String nome; // REDE, STONE, GETNET, OWN_ACQUIRER
    private String tipoEndpoint;
    @Column(columnDefinition = "JSONB")
    private String config;
    private Boolean ativo = true;

    // delombok
}
```

---

## Task 5 — Expand existing entities: Cartao, Fatura, TransacaoCartao

**Dependency:** NONE (parallel, but must NOT conflict with Task 4)

### Cartao.java changes:
- ADD `private Long produtoId;`
- ADD `private Long bandeiraParceiroId;` (FK to ParceiroBandeira)
- ADD `private String tenantId;` (override BaseEntity's nullable)
- Ensure enum `TipoCartao` includes `MULTIFUNCIONAL`
- Ensure enum `BandeiraCartao` stays alongside new `ParceiroBandeira` concept
- ADD `private LocalDateTime dataCancelamento;`

### Fatura.java changes:
- ADD `private LocalDate dataFechamento;` (already exists as LocalDateTime — change to LocalDate or add LocalDate dataFechamentoDate)
- ADD `private String codigoTransacao;` (nullable, FK-like)
- ADD private fields: `dataVencimentoOriginal`, `dataPagamentoEfetivo`

### TransacaoCartao.java changes:
- ADD `private Long faturaId;` (nullable — assigned at billing time)
- ADD `@Enumerated(EnumType.STRING) private String modo;` (CREDITO, DEBITO) — new field
- ADD private fields: `codigoAutorizacao` (already has `autorizacao`), `moeda`
- Ensure statuses match design: AUTORIZADA, NEGADA, ESTORNADA, CANCELADA

IMPORTANT: Preserve ALL existing fields. Only add new ones. Do not break existing getters/setters/constructors. Append new fields with their own getters/setters.

---

## Task 6 — DTOs for all API endpoints

**Dependency:** Tasks 4-5 (entities must exist first)

Create all DTOs in `apps/backend/aurix-cartoes/src/main/java/com/aurix/platform/cartoes/dto/`:

- `EmitirCartaoRequest.java` — produtoId, contaId, titular, tipo
- `CartaoResponse.java` — full cartao data (masked number, no CVV)
- `BloquearCartaoRequest.java` — motivo
- `AjustarLimiteRequest.java` — novoLimite
- `LimiteCartaoResponse.java` — limiteTotal, limiteDisponivel, limiteUtilizado
- `AutorizarTransacaoRequest.java` — cartaoId, valor, estabelecimento, modo
- `TransacaoResponse.java` — id, cartaoId, valor, status, codigoAutorizacao, dataTransacao
- `FaturaResponse.java` — id, cartaoId, mesReferencia, valorTotal, valorPago, status, dataVencimento
- `FaturaDetalhadaResponse.java` — extends FaturaResponse + List<LancamentoFatura>
- `PagarFaturaRequest.java` — valorPagamento
- `ProdutoCartaoRequest.java` — nome, bandeira, adquirente, anuidade, etc.
- `ProdutoCartaoResponse.java` — full produto data
- `ParceiroBandeiraRequest.java` — nome, tipoEndpoint, config JSON
- `ParceiroBandeiraResponse.java`
- `ParceiroAdquirenteRequest.java` — nome, tipoEndpoint, config JSON
- `ParceiroAdquirenteResponse.java`

All DTOs follow the manual getters/setters pattern from `aurix-poupanca/dto/`.

---

## Task 7 — Repositories for new entities

**Dependency:** Tasks 4-5 (entities)

- `ProdutoCartaoRepository.java` — findByAtivoTrue, findByBandeira
- `LimiteCartaoRepository.java` — findByCartaoId
- `LancamentoFaturaRepository.java` — findByFaturaId
- `ParceiroBandeiraRepository.java` — findByNome, findByAtivoTrue
- `ParceiroAdquirenteRepository.java` — findByNome, findByAtivoTrue

Also ADD to existing `CartaoRepository.java`:
- `findByProdutoId(Long produtoId)`
- `findByBandeiraParceiroId(Long bandeiraParceiroId)`
- `findByTenantId(String tenantId)`

---

## Task 8 — Kafka Events

**Dependency:** NONE (pure records, no entity dependency)

Create event records in `event/`:

```java
// CartaoEmitidoEvent.java
public record CartaoEmitidoEvent(
    Long cartaoId, Long contaId, String nomePortador,
    String bandeira, String tipo, String tenantId
) {}

// TransacaoAutorizadaEvent.java
public record TransacaoAutorizadaEvent(
    String codigoTransacao, Long cartaoId, BigDecimal valor,
    String estabelecimento, String autorizacao, String status, String tenantId
) {}

// TransacaoEstornadaEvent.java
public record TransacaoEstornadaEvent(
    String codigoTransacao, Long cartaoId, BigDecimal valor, String tenantId
) {}

// FaturaFechadaEvent.java
public record FaturaFechadaEvent(
    Long faturaId, Long cartaoId, Integer mesReferencia,
    Integer anoReferencia, BigDecimal valorTotal, String tenantId
) {}

// FaturaPagaEvent.java
public record FaturaPagaEvent(
    Long faturaId, Long cartaoId, BigDecimal valorPago, String tenantId
) {}
```

Also create `CartoesKafkaConfig.java` with:
- Topic constants for each event type
- `@Bean NewTopic` for each topic (3 partitions, 1 replica)

---

## Task 9 — HTTP Clients

**Dependency:** NONE (interfaces, no entity dependency)

Create in `client/`:

### ContaCorrenteClient.java (same pattern as poupanca's)
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

### MlFraudClient.java (antifraude — aurix-ml)
```java
@HttpExchange("/api/ml/fraud")
public interface MlFraudClient {
    @PostExchange("/avaliar")
    FraudResponse avaliar(@RequestBody FraudRequest request);
    record FraudRequest(Long cartaoId, BigDecimal valor, String estabelecimento, String modo) {}
    record FraudResponse(String resultado, Double score, String recomendacao) {}
}
```

Also create placeholder interfaces for partner clients (not wired to real endpoints yet):
- `VisaClient.java`, `MastercardClient.java`, `EloClient.java`
- `RedeClient.java`, `StoneClient.java`, `GetNetClient.java`

### CartoesHttpConfig.java (like PoupancaHttpConfig)
```java
@Configuration
@EnableResilientMethods
@ImportHttpServices({ContaCorrenteClient.class, MlFraudClient.class,
    VisaClient.class, MastercardClient.class, EloClient.class,
    RedeClient.class, StoneClient.class, GetNetClient.class})
public class CartoesHttpConfig {}
```

---

## Task 10 — Services

**Dependency:** Tasks 4-9 (entities, repos, events, clients)

### CartaoService.java — REFACTOR existing
Preserve ALL existing methods. Add:

- `emitirCartao(EmitirCartaoRequest)` — accept DTO, add tenantId + tenantId from context, create LimiteCartao, publish CartaoEmitidoEvent
- `bloquearCartao(Long id, String motivo)` — existing, ensure is PATCH-style (no change needed)
- `desbloquearCartao(Long id)` — existing
- `cancelarCartao(Long id)` — new: set status CANCELADO, dataCancelamento
- `obterCartao(Long id)` — new: return CartaoResponse DTO
- `ajustarLimite(Long id, BigDecimal novoLimite)` — refactor to also create/update LimiteCartao entity
- `consultarLimite(Long id)` — new: return LimiteCartaoResponse

### FaturaService.java — NEW
- `listarFaturas(Long cartaoId)` — return list of FaturaResponse
- `obterFaturaDetalhada(Long faturaId)` — return FaturaDetalhadaResponse with LancamentoFatura list
- `pagarFatura(Long faturaId, PagarFaturaRequest)` — debit ContaCorrenteClient, update Fatura status, publish FaturaPagaEvent, update Cartao limite

### TransacaoCartaoService.java — NEW
- `autorizarTransacao(AutorizarTransacaoRequest)` — flow: AntifraudeService -> LimiteService -> cria TransacaoCartao -> publica TransacaoAutorizadaEvent
- `estornarTransacao(String codigoTransacao)` — set status ESTORNADA, creditar limite, publish TransacaoEstornadaEvent
- `listarTransacoes(Long cartaoId)` — return list

### LimiteCartaoService.java — NEW
- `consultarLimite(Long cartaoId)` — return LimiteCartao
- `ajustarLimite(Long cartaoId, BigDecimal novoLimite)` — create/update LimiteCartao, update Cartao limite fields
- `verificarDisponibilidade(Long cartaoId, BigDecimal valor)` — boolean check
- `debitarLimite(Long cartaoId, BigDecimal valor)` — reduce disponivel, increase utilizado
- `creditarLimite(Long cartaoId, BigDecimal valor)` — reverse operation

### ProdutoCartaoService.java — NEW
- CRUD for ProdutoCartao
- `listarProdutos()` — list active
- `criarProduto(ProdutoCartaoRequest)`
- `atualizarProduto(Long id, ProdutoCartaoRequest)`

### ParceiroService.java — NEW
- CRUD for ParceiroBandeira and ParceiroAdquirente
- `listarBandeiras()`, `configurarBandeira(ParceiroBandeiraRequest)`
- `listarAdquirentes()`, `configurarAdquirente(ParceiroAdquirenteRequest)`

### AntifraudeService.java — NEW
- `avaliarRisco(Long cartaoId, BigDecimal valor, String estabelecimento, String modo)`
- If MlFraudClient available -> call it -> return resultado
- If unavailable -> return "APROVADO" with score 0.0 (fallback)
- If alto risco -> "NEGADO"

---

## Task 11 — Controllers

**Dependency:** Tasks 6-10 (DTOs + services)

### CartaoController.java — REFACTOR existing (preserve all existing endpoints, add new ones)

Current base path: `/api/cartoes` — KEEP THIS.

Endpoints to ADD (matching Section 4.3):
```
POST   /api/cartoes/emitir                       -> emitirCartao
GET    /api/cartoes/{id}                          -> obterCartao
PATCH  /api/cartoes/{id}/bloquear                 -> bloquearCartao
PATCH  /api/cartoes/{id}/desbloquear              -> desbloquearCartao
PATCH  /api/cartoes/{id}/cancelar                 -> cancelarCartao
PATCH  /api/cartoes/{id}/limite                   -> ajustarLimite
GET    /api/cartoes/{id}/limite                   -> consultarLimite
```

NOTE: Existing POST endpoints `/{id}/bloquear`, `/{id}/desbloquear`, `/{id}/limite` will cause mapping conflicts with new PATCH ones. **Resolution:** Change existing POST mappings to PATCH to match spec. Keep backward compat by keeping original method names in service.

### TransacaoCartaoController.java — NEW
```
POST   /api/cartoes/transacoes/autorizar          -> autorizarTransacao
POST   /api/cartoes/transacoes/estornar           -> estornarTransacao
GET    /api/cartoes/transacoes/{cartaoId}          -> listarTransacoes
```

### FaturaController.java — NEW
```
GET    /api/cartoes/faturas/{cartaoId}             -> listarFaturas
GET    /api/cartoes/faturas/detalhe/{id}           -> obterFaturaDetalhada
POST   /api/cartoes/faturas/{id}/pagar             -> pagarFatura
```

### ProdutoCartaoController.java — NEW
```
GET    /api/cartoes/produtos                       -> listarProdutos
POST   /api/cartoes/produtos                       -> criarProduto
PUT    /api/cartoes/produtos/{id}                   -> atualizarProduto
```

### ParceiroController.java — NEW
```
GET    /api/cartoes/parceiros-bandeira             -> listarBandeiras
POST   /api/cartoes/parceiros-bandeira             -> configurarBandeira
GET    /api/cartoes/parceiros-adquirente           -> listarAdquirentes
POST   /api/cartoes/parceiros-adquirente           -> configurarAdquirente
```

All controllers follow the `aurix-poupanca` pattern: constructor injection, `@Tag` for Swagger, `@Operation` on each method, ResponseEntity return types.

---

## Task 12 — Job: FaturamentoJob + CartoesSecurityConfig + CartoesKafkaConfig

**Dependency:** Tasks 4-11

### FaturamentoJob.java
```java
package com.aurix.platform.cartoes.job;

@Component
public class FaturamentoJob {
    @Scheduled(cron = "0 0 2 1 * *") // 1st day of month at 02:00
    public void fecharFaturasMensais() {
        // For each cartao ATIVO:
        // 1. Aggregate TransacaoCartao from last month not yet in any fatura
        // 2. Create Fatura with status FECHADA
        // 3. Create LancamentoFatura for each transacao
        // 4. Publish FaturaFechadaEvent
        // 5. Error handling: try-catch per cartao, log and continue
    }

    @Scheduled(cron = "0 0 3 * * *") // daily at 03:00
    public void atualizarFaturasVencidas() {
        // Find faturas ABERTAS or FECHADAS past vencimento -> set VENCIDA
    }
}
```

### CartoesSecurityConfig.java
```java
@Configuration
@EnableWebSecurity
@Profile("!test")
public class CartoesSecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/**", "/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```

---

## Task 13 — Tests (by endpoint)

**Dependency:** Tasks 1-12 (all implementation must exist)

**Test structure** (follow `ContaPoupancaControllerTest.java` pattern exactly):
- `@SpringBootTest(classes = AurixCartoesApplication.class, webEnvironment = RANDOM_PORT)`
- `@ActiveProfiles("test")`
- `@Import(TestConfig.class)` with:
  - `SecurityFilterChain` permitAll
  - Mock `KafkaTemplate<String, Object>`
  - Mock `ContaCorrenteClient`
  - Mock `MlFraudClient`
- `@LocalServerPort` int port
- `RestTemplate rest`
- `url(String path)` helper

Test classes:
- `CartaoControllerTest.java` — test emitir, obter, bloquear, desbloquear, cancelar, ajustar limite, consultar limite
- `TransacaoCartaoControllerTest.java` — test autorizar (fluxo feliz), autorizar (limite insuficiente), autorizar (antifraude nega), estornar, listar
- `FaturaControllerTest.java` — test listar faturas, obter fatura detalhada, pagar fatura
- `ProdutoCartaoControllerTest.java` — test CRUD
- `ParceiroControllerTest.java` — test CRUD bandeira + adquirente

Each test class tests at minimum 2 scenarios per public endpoint: happy path + validation error.

Also add `application-test.yml` in `src/test/resources/` with H2 config.

---

## Task 14 — Cross-cutting updates

**Dependency:** Task 1 (POM) + all implementation

### Gateway route — add to `aurix-gateway/src/main/resources/application.yml`
Insert before the Swagger UI route:
```yaml
# AURIX Cartoes
- id: aurix-cartoes
  uri: http://localhost:8115
  predicates:
    - Path=/api/cartoes/**
  filters:
    - StripPrefix=0
```

### Parent POM — already has `aurix-cartoes` module (line 49 in pom.xml). No change needed.

### OpenAPI spec — after implementation, add via springdoc-openapi:
- Tag: `Cartoes` with description
- Paths for all 21 endpoints from Section 4.3
- Schemas for all entities and DTOs

---

## Execution Order (dependency groups)

```
Group A (parallel):
  Task 1  — pom.xml
  Task 2  — application.yml
  Task 3  — package-info.java scaffold
  Task 4  — new entities
  Task 5  — expand existing entities
  Task 8  — Kafka events

Group B (depends on A):
  Task 6  — DTOs (needs entities 4-5)
  Task 7  — new repositories (needs entities 4-5)
  Task 9  — HTTP clients + config (independent but logically grouped)

Group C (depends on B):
  Task 10 — Services (needs entities, repos, DTOs, events, clients)
  Task 11 — Controllers (needs services + DTOs)

Group D (depends on C):
  Task 12 — Job + Security config (needs services)
  Task 13 — Tests (needs everything)

Group E (depends on D):
  Task 14 — Cross-cutting updates

Gate: mvn clean install -DskipTests -pl aurix-cartoes -am (compilation check)
Gate: mvn test -pl aurix-cartoes (all tests green)
Gate: mvn pmd:check spotbugs:check checkstyle:check -pl aurix-cartoes (static analysis)
```
