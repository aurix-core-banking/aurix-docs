# KYC / Bureau / Fraud Integration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace bureau and KYC stubs with real multi-provider integrations (Serasa, Quod, Unico, ClearSale) using Gateway + Strategy pattern.

**Architecture:** Gateway pattern with ordered fallback for bureau (Serasa → Quod → stub in dev); direct implementation for KYC (Unico in prod, stub otherwise); new FraudService for ClearSale. Circuit breaker via resilience4j.

**Tech Stack:** Spring Boot 4.1, resilience4j-spring-boot3 (via aureus-bacen reference), RestTemplate, WireMock (test), JUnit 5.

## Global Constraints

- Follow existing delombok pattern (explicit builder/getter/setter/constructor)
- Use `@Profile("!producao")` for stubs (KycProviderStub); `@Profile("dev|test")` for BureauStub
- New interfaces go in `com.aureus.platform.onboarding.service` (matching KycProviderService / BureauService location)
- Config goes in `application.yml` (default/stub) and `application-prod.yml` (real providers)
- Dependency `io.github.resilience4j:resilience4j-spring-boot3` (already used by aureus-bacen, managed by spring-cloud BOM)
- Tests use WireMock for HTTP provider simulation
- All existing tests must pass after each task

---
### Task 0: Dependencies + application-prod.yml

**Files:**
- Modify: `backend/aureus-onboarding/pom.xml`
- Create: `backend/aureus-onboarding/src/main/resources/application-prod.yml`
- Modify: `backend/aureus-onboarding/src/main/resources/application.yml`

**Interfaces:**
- Produces: `resilience4j-spring-boot3` on classpath; application-prod.yml with provider configs; default (stub) config in application.yml

- [ ] **Step 1: Add resilience4j to pom.xml**

Add inside `<dependencies>` after the existing Spring Boot starters:

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

Note: `spring-cloud-dependencies` BOM in root pom manages versions at `2025.1.2`.

- [ ] **Step 2: Create application-prod.yml**

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
  datasource:
    hikari:
      maximum-pool-size: 30
      minimum-idle: 5

aureus:
  onboarding:
    bureau:
      serasa:
        url: https://api.serasa.com.br/v2/score
        api-key: ${SERASA_API_KEY}
        timeout: 5000
      quod:
        url: https://api.quod.com.br/v1/consultas
        api-key: ${QUOD_API_KEY}
        timeout: 5000
    kyc:
      unico:
        url: https://api.unico.io/v1/biometrics
        api-key: ${UNICO_API_KEY}
    fraud:
      cleansale:
        url: https://api.clearsale.com.br/v1/orders
        api-key: ${CLEARSALE_API_KEY}

logging:
  level:
    com.aureus.platform: INFO
```

- [ ] **Step 3: Add default config to application.yml**

Append before `management:` block:

```yaml
aureus:
  onboarding:
    bureau:
      serasa:
        url: http://localhost:9999/bureau/serasa
        api-key: stub-key
        timeout: 5000
      quod:
        url: http://localhost:9999/bureau/quod
        api-key: stub-key
        timeout: 5000
    kyc:
      unico:
        url: http://localhost:9999/kyc/unico
        api-key: stub-key
    fraud:
      cleansale:
        url: http://localhost:9999/fraud/clearsale
        api-key: stub-key
```

Place these keys under the existing `aureus.onboarding` block (after `tenant-header`).

- [ ] **Step 4: Commit**

```bash
git add backend/aureus-onboarding/pom.xml backend/aureus-onboarding/src/main/resources/application.yml backend/aureus-onboarding/src/main/resources/application-prod.yml
git commit -m "chore(onboarding): add resilience4j deps, prod config, default provider endpoints"
```

---
### Task 1: BureauProvider interface + BureauGateway + BureauStub @Profile fix

**Files:**
- Create: `backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/BureauProvider.java`
- Create: `backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/BureauGateway.java`
- Create: `backend/aureus-onboarding/src/test/java/com/aureus/platform/onboarding/service/BureauGatewayTest.java`
- Modify: `backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/BureauStub.java`

**Interfaces:**
- Produces: `BureauProvider` interface with method `ResultadoBureau consultar(String cpf)`
- Produces: `BureauGateway` implementing `BureauService`, injecting `List<BureauProvider>` with fallback logic
- Consumes: `BureauService` (existing), `BureauService.ResultadoBureau` (existing)

- [ ] **Step 1: Write BureauProvider interface**

```java
package com.aureus.platform.onboarding.service;

public interface BureauProvider {
    BureauService.ResultadoBureau consultar(String cpf);
}
```

- [ ] **Step 2: Write BureauGateway**

```java
package com.aureus.platform.onboarding.service;

import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class BureauGateway implements BureauService {
    @java.lang.SuppressWarnings("all")
    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(BureauGateway.class);

    private final List<BureauProvider> providers;

    public BureauGateway(List<BureauProvider> providers) {
        this.providers = providers;
    }

    @Override
    public ResultadoBureau consultar(String cpf) {
        for (BureauProvider provider : providers) {
            try {
                log.debug("Tentando provider {}", provider.getClass().getSimpleName());
                ResultadoBureau result = provider.consultar(cpf);
                if (result != null) {
                    log.debug("Provider {} retornou score={}", provider.getClass().getSimpleName(), result.score());
                    return result;
                }
            } catch (Exception e) {
                log.warn("Provider {} falhou: {}", provider.getClass().getSimpleName(), e.getMessage());
            }
        }
        throw new IllegalStateException("Todos os provedores de consulta de CPF falharam");
    }
}
```

- [ ] **Step 3: Fix BureauStub — implementa BureauProvider em vez de BureauService**

Mudar para `implements BureauProvider` e adicionar `@Profile("dev|test")`. Isso evita conflito com `BureauGateway` (que implementa `BureauService`) e faz o stub ser o fallback final na lista do gateway.

```java
package com.aureus.platform.onboarding.service;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;

@Service
@Profile("dev|test")
public class BureauStub implements BureauProvider {
    @java.lang.SuppressWarnings("all")
    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(BureauStub.class);

    @Override
    public BureauService.ResultadoBureau consultar(String cpf) {
        log.debug("Bureau stub: consulta CPF {}", cpf);
        return new BureauService.ResultadoBureau(600, "REGULAR", "Consulta simulada - integrar Serasa/SPC/Quod");
    }
}
```

- [ ] **Step 4: Write BureauGatewayTest**

```java
package com.aureus.platform.onboarding.service;

import org.junit.jupiter.api.Test;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class BureauGatewayTest {

    private record StubProvider(String name, BureauService.ResultadoBureau result, boolean fail) implements BureauProvider {
        @Override
        public BureauService.ResultadoBureau consultar(String cpf) {
            if (fail) throw new RuntimeException(name + " failed");
            return result;
        }
    }

    @Test
    void deveUsarPrimeiroProvedorQuandoDisponivel() {
        BureauService.ResultadoBureau expected = new BureauService.ResultadoBureau(700, "REGULAR", "OK");
        BureauProvider p1 = new StubProvider("p1", expected, false);
        BureauProvider p2 = new StubProvider("p2", null, false);
        BureauGateway gateway = new BureauGateway(List.of(p1, p2));

        BureauService.ResultadoBureau result = gateway.consultar("52998224725");

        assertThat(result.score()).isEqualTo(700);
    }

    @Test
    void deveFazerFallbackQuandoPrimeiroFalha() {
        BureauService.ResultadoBureau expected = new BureauService.ResultadoBureau(500, "REGULAR", "fallback");
        BureauProvider p1 = new StubProvider("p1", null, true);
        BureauProvider p2 = new StubProvider("p2", expected, false);
        BureauGateway gateway = new BureauGateway(List.of(p1, p2));

        BureauService.ResultadoBureau result = gateway.consultar("52998224725");

        assertThat(result.score()).isEqualTo(500);
    }

    @Test
    void deveLancarExcecaoQuandoTodosFalham() {
        BureauProvider p1 = new StubProvider("p1", null, true);
        BureauProvider p2 = new StubProvider("p2", null, true);
        BureauGateway gateway = new BureauGateway(List.of(p1, p2));

        assertThatThrownBy(() -> gateway.consultar("52998224725"))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("Todos os provedores");
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

```bash
mvn -pl aureus-onboarding test -Dtest=BureauGatewayTest -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: BUILD SUCCESS, 3 tests pass

- [ ] **Step 6: Commit**

```bash
git add backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/BureauProvider.java backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/BureauGateway.java backend/aureus-onboarding/src/test/java/com/aureus/platform/onboarding/service/BureauGatewayTest.java backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/BureauStub.java
git commit -m "feat(onboarding): add BureauProvider + BureauGateway with fallback chain"
```

---
### Task 2: SerasaProvider + WireMock test

**Files:**
- Create: `backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/SerasaProvider.java`
- Create: `backend/aureus-onboarding/src/test/java/com/aureus/platform/onboarding/service/SerasaProviderTest.java`

**Interfaces:**
- Consumes: `BureauProvider` (from Task 1), RestTemplate
- Produces: implementation of `BureauProvider` that maps Serasa API responses

- [ ] **Step 1: Write SerasaProvider**

```java
package com.aureus.platform.onboarding.service;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class SerasaProvider implements BureauProvider {
    @java.lang.SuppressWarnings("all")
    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(SerasaProvider.class);

    private final RestTemplate restTemplate;
    private final String url;
    private final String apiKey;

    public SerasaProvider(RestTemplate restTemplate,
                          @Value("${aureus.onboarding.bureau.serasa.url}") String url,
                          @Value("${aureus.onboarding.bureau.serasa.api-key}") String apiKey) {
        this.restTemplate = restTemplate;
        this.url = url;
        this.apiKey = apiKey;
    }

    @Override
    public BureauService.ResultadoBureau consultar(String cpf) {
        log.debug("Consultando Serasa para CPF {}", cpf);
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("X-API-Key", apiKey);
        var request = new HttpEntity<>(new SerasaRequest(cpf), headers);
        try {
            SerasaResponse response = restTemplate.exchange(url, HttpMethod.POST, request, SerasaResponse.class).getBody();
            if (response == null) {
                throw new RuntimeException("Resposta nula da Serasa");
            }
            return new BureauService.ResultadoBureau(response.score(), response.situacao(), "Serasa: " + response.mensagem());
        } catch (Exception e) {
            log.warn("Erro ao consultar Serasa: {}", e.getMessage());
            throw e;
        }
    }

    @com.fasterxml.jackson.annotation.JsonIgnoreProperties(ignoreUnknown = true)
    record SerasaRequest(String cpf) {}

    @com.fasterxml.jackson.annotation.JsonIgnoreProperties(ignoreUnknown = true)
    record SerasaResponse(int score, String situacao, String mensagem) {}
}
```

- [ ] **Step 2: Write SerasaProviderTest (WireMock)**

```java
package com.aureus.platform.onboarding.service;

import com.github.tomakehurst.wiremock.junit5.WireMockTest;
import org.junit.jupiter.api.Test;
import org.springframework.web.client.RestTemplate;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

@WireMockTest(httpPort = 9999)
class SerasaProviderTest {

    private final RestTemplate restTemplate = new RestTemplate();

    @Test
    void deveConsultarScoreComSucesso() {
        stubFor(post(urlEqualTo("/bureau/serasa"))
            .withRequestBody(containing("52998224725"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {"score": 750, "situacao": "REGULAR", "mensagem": "OK"}
                    """)));

        SerasaProvider provider = new SerasaProvider(restTemplate,
            "http://localhost:9999/bureau/serasa", "test-key");

        BureauService.ResultadoBureau result = provider.consultar("52998224725");

        assertThat(result.score()).isEqualTo(750);
        assertThat(result.situacao()).isEqualTo("REGULAR");
    }

    @Test
    void deveLancarExcecaoQuandoApiRetornaErro() {
        stubFor(post(urlEqualTo("/bureau/serasa"))
            .willReturn(aResponse().withStatus(500)));

        SerasaProvider provider = new SerasaProvider(restTemplate,
            "http://localhost:9999/bureau/serasa", "test-key");

        org.junit.jupiter.api.Assertions.assertThrows(Exception.class,
            () -> provider.consultar("52998224725"));
    }
}
```

- [ ] **Step 3: Add WireMock dependency to pom.xml**

Add to `<dependencies>` (test scope):

```xml
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <version>3.12.1</version>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 4: Run tests**

```bash
mvn -pl aureus-onboarding test -Dtest="SerasaProviderTest,BureauGatewayTest" -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: BUILD SUCCESS, 5 tests pass

- [ ] **Step 5: Commit**

```bash
git add backend/aureus-onboarding/pom.xml backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/SerasaProvider.java backend/aureus-onboarding/src/test/java/com/aureus/platform/onboarding/service/SerasaProviderTest.java
git commit -m "feat(onboarding): add SerasaProvider with WireMock test"
```

---
### Task 3: QuodProvider + WireMock test

**Files:**
- Create: `backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/QuodProvider.java`
- Create: `backend/aureus-onboarding/src/test/java/com/aureus/platform/onboarding/service/QuodProviderTest.java`

- [ ] **Step 1: Write QuodProvider**

```java
package com.aureus.platform.onboarding.service;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class QuodProvider implements BureauProvider {
    @java.lang.SuppressWarnings("all")
    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(QuodProvider.class);

    private final RestTemplate restTemplate;
    private final String url;
    private final String apiKey;

    public QuodProvider(RestTemplate restTemplate,
                        @Value("${aureus.onboarding.bureau.quod.url}") String url,
                        @Value("${aureus.onboarding.bureau.quod.api-key}") String apiKey) {
        this.restTemplate = restTemplate;
        this.url = url;
        this.apiKey = apiKey;
    }

    @Override
    public BureauService.ResultadoBureau consultar(String cpf) {
        log.debug("Consultando Quod para CPF {}", cpf);
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("Authorization", "Bearer " + apiKey);
        var request = new HttpEntity<>(new QuodRequest(cpf), headers);
        try {
            QuodResponse response = restTemplate.exchange(url, HttpMethod.POST, request, QuodResponse.class).getBody();
            if (response == null) {
                throw new RuntimeException("Resposta nula do Quod");
            }
            return new BureauService.ResultadoBureau(response.score(), response.situacao(), "Quod: " + response.mensagem());
        } catch (Exception e) {
            log.warn("Erro ao consultar Quod: {}", e.getMessage());
            throw e;
        }
    }

    @com.fasterxml.jackson.annotation.JsonIgnoreProperties(ignoreUnknown = true)
    record QuodRequest(String cpf) {}

    @com.fasterxml.jackson.annotation.JsonIgnoreProperties(ignoreUnknown = true)
    record QuodResponse(int score, String situacao, String mensagem) {}
}
```

- [ ] **Step 2: Write QuodProviderTest**

```java
package com.aureus.platform.onboarding.service;

import com.github.tomakehurst.wiremock.junit5.WireMockTest;
import org.junit.jupiter.api.Test;
import org.springframework.web.client.RestTemplate;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

@WireMockTest(httpPort = 9999)
class QuodProviderTest {

    private final RestTemplate restTemplate = new RestTemplate();

    @Test
    void deveConsultarScoreComSucesso() {
        stubFor(post(urlEqualTo("/bureau/quod"))
            .withRequestBody(containing("52998224725"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {"score": 620, "situacao": "REGULAR", "mensagem": "OK"}
                    """)));

        QuodProvider provider = new QuodProvider(restTemplate,
            "http://localhost:9999/bureau/quod", "test-key");

        BureauService.ResultadoBureau result = provider.consultar("52998224725");

        assertThat(result.score()).isEqualTo(620);
        assertThat(result.situacao()).isEqualTo("REGULAR");
    }

    @Test
    void deveLancarExcecaoQuandoApiRetornaErro() {
        stubFor(post(urlEqualTo("/bureau/quod"))
            .willReturn(aResponse().withStatus(500)));

        QuodProvider provider = new QuodProvider(restTemplate,
            "http://localhost:9999/bureau/quod", "test-key");

        org.junit.jupiter.api.Assertions.assertThrows(Exception.class,
            () -> provider.consultar("52998224725"));
    }
}
```

- [ ] **Step 3: Run tests**

```bash
mvn -pl aureus-onboarding test -Dtest="QuodProviderTest,SerasaProviderTest,BureauGatewayTest" -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: BUILD SUCCESS, 7 tests pass

- [ ] **Step 4: Commit**

```bash
git add backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/QuodProvider.java backend/aureus-onboarding/src/test/java/com/aureus/platform/onboarding/service/QuodProviderTest.java
git commit -m "feat(onboarding): add QuodProvider with WireMock test"
```

---
### Task 4: FraudService + FraudStub + SolicitacaoOnboarding field + OnboardingPFService integration

**Files:**
- Create: `backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/FraudService.java`
- Create: `backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/FraudStub.java`
- Create: `backend/aureus-onboarding/src/test/java/com/aureus/platform/onboarding/service/FraudStubTest.java`
- Modify: `backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/entity/SolicitacaoOnboarding.java`
- Modify: `backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/OnboardingPFService.java`
- Modify: `backend/aureus-onboarding/src/test/java/com/aureus/platform/onboarding/integration/OnboardingPFFlowIntegrationTest.java`
- Modify: `backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/dto/SolicitacaoContaResponse.java`

**Interfaces:**
- Produces: `FraudService` with method `ResultadoFraude analisar(String cpf, String nome, String email, String telefone)`
- Produces: `FraudStub` (`@Profile("!producao")`) returning approved
- Consumes: `FraudService` in `OnboardingPFService.solicitarAberturaConta()`

- [ ] **Step 1: Write FraudService interface**

```java
package com.aureus.platform.onboarding.service;

public interface FraudService {

    ResultadoFraude analisar(String cpf, String nome, String email, String telefone);

    record ResultadoFraude(boolean aprovado, String codigo, String mensagem, int risco) {}
}
```

- [ ] **Step 2: Write FraudStub**

```java
package com.aureus.platform.onboarding.service;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;

@Service
@Profile("!producao")
public class FraudStub implements FraudService {
    @java.lang.SuppressWarnings("all")
    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(FraudStub.class);

    @Override
    public ResultadoFraude analisar(String cpf, String nome, String email, String telefone) {
        log.debug("Fraude stub: analisando CPF {}", cpf);
        return new ResultadoFraude(true, "APROVADO_STUB", "Stub de fraude - integrar ClearSale", 0);
    }
}
```

- [ ] **Step 3: Write FraudStubTest**

```java
package com.aureus.platform.onboarding.service;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class FraudStubTest {

    private final FraudStub stub = new FraudStub();

    @Test
    void deveRetornarAprovado() {
        FraudService.ResultadoFraude result = stub.analisar("52998224725", "Maria", "maria@teste.com", "11999999999");
        assertThat(result.aprovado()).isTrue();
        assertThat(result.risco()).isZero();
        assertThat(result.codigo()).isEqualTo("APROVADO_STUB");
    }
}
```

- [ ] **Step 4: Add riscoFraude field to SolicitacaoOnboarding**

Add field after `observacoesAnalista`:

```java
@Column(name = "risco_fraude")
private Integer riscoFraude;
```

Add to constructor (after `observacoesAnalista` parameter), to builder, to `getRiscoFraude()` getter, and to the `from()` static factory in SolicitacaoContaResponse:

In `SolicitacaoContaResponse.java`, add field:
```java
private Integer riscoFraude;
```

Add to builder, constructor, getter/setter, and `from()`:
```java
.riscoFraude(onboarding.getRiscoFraude())
```

- [ ] **Step 5: Wire FraudService into OnboardingPFService**

Add to constructor parameters, add field:
```java
private final FraudService fraudService;
```

In `solicitarAberturaConta()`, after Bureau call and before creating entities:

```java
FraudService.ResultadoFraude fraude = fraudService.analisar(
    request.getCpf(), request.getNome(), request.getEmail(), request.getTelefone());
```

Pass `riscoFraude` to builder and set initial status based on risk:

```java
.onboarding(SolicitacaoOnboarding.builder()
    .riscoFraude(fraude.risco())
    ...
```

- [ ] **Step 6: Update OnboardingPFFlowIntegrationTest**

Add `@MockitoBean` for FraudService (mock returns approved):

```java
@MockitoBean
private FraudService fraudService;

when(fraudService.analisar(anyString(), anyString(), anyString(), anyString()))
    .thenReturn(new FraudService.ResultadoFraude(true, "APROVADO", "Mock", 0));
```

Import `FraudService`.

Also need to adjust `limparBanco()` if `risco_fraude` column was added (no change needed — H2 DDL auto handles new column via `create-drop`).

- [ ] **Step 7: Run all onboarding tests**

```bash
mvn -pl aureus-onboarding test -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 8: Commit**

```bash
git add backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/FraudService.java backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/FraudStub.java backend/aureus-onboarding/src/test/java/com/aureus/platform/onboarding/service/FraudStubTest.java backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/entity/SolicitacaoOnboarding.java backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/OnboardingPFService.java backend/aureus-onboarding/src/test/java/com/aureus/platform/onboarding/integration/OnboardingPFFlowIntegrationTest.java backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/dto/SolicitacaoContaResponse.java
git commit -m "feat(onboarding): add FraudService + FraudStub + riscoFraude field, wire into OnboardingPFService"
```

---
### Task 5: ClearSaleProvider + WireMock test

**Files:**
- Create: `backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/ClearSaleProvider.java`
- Create: `backend/aureus-onboarding/src/test/java/com/aureus/platform/onboarding/service/ClearSaleProviderTest.java`

- [ ] **Step 1: Write ClearSaleProvider**

```java
package com.aureus.platform.onboarding.service;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
@Profile("producao")
public class ClearSaleProvider implements FraudService {
    @java.lang.SuppressWarnings("all")
    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(ClearSaleProvider.class);

    private final RestTemplate restTemplate;
    private final String url;
    private final String apiKey;

    public ClearSaleProvider(RestTemplate restTemplate,
                              @Value("${aureus.onboarding.fraud.cleansale.url}") String url,
                              @Value("${aureus.onboarding.fraud.cleansale.api-key}") String apiKey) {
        this.restTemplate = restTemplate;
        this.url = url;
        this.apiKey = apiKey;
    }

    @Override
    public ResultadoFraude analisar(String cpf, String nome, String email, String telefone) {
        log.debug("Analisando fraude via ClearSale para CPF {}", cpf);
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("API-Key", apiKey);
        var request = new HttpEntity<>(new ClearSaleRequest(cpf, nome, email, telefone), headers);
        try {
            ClearSaleResponse response = restTemplate.exchange(url, HttpMethod.POST, request, ClearSaleResponse.class).getBody();
            if (response == null) {
                throw new RuntimeException("Resposta nula do ClearSale");
            }
            boolean aprovado = "APROVADO".equalsIgnoreCase(response.recomendacao());
            return new ResultadoFraude(aprovado, response.codigo(), "ClearSale: " + response.mensagem(), response.risco());
        } catch (Exception e) {
            log.warn("Erro ao consultar ClearSale, assumindo aprovado: {}", e.getMessage());
            return new ResultadoFraude(true, "FALHA_PROVEDOR", "ClearSale indisponivel, assumindo aprovado", 0);
        }
    }

    @com.fasterxml.jackson.annotation.JsonIgnoreProperties(ignoreUnknown = true)
    record ClearSaleRequest(String cpf, String nome, String email, String telefone) {}

    @com.fasterxml.jackson.annotation.JsonIgnoreProperties(ignoreUnknown = true)
    record ClearSaleResponse(String recomendacao, String codigo, String mensagem, int risco) {}
}
```

Note: ClearSale returns `ResultadoFraude(true, ...)` on failure — this is intentional per the design doc (log warning, assume approved to avoid blocking onboarding).

- [ ] **Step 2: Write ClearSaleProviderTest**

```java
package com.aureus.platform.onboarding.service;

import com.github.tomakehurst.wiremock.junit5.WireMockTest;
import org.junit.jupiter.api.Test;
import org.springframework.web.client.RestTemplate;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

@WireMockTest(httpPort = 9999)
class ClearSaleProviderTest {

    private final RestTemplate restTemplate = new RestTemplate();

    @Test
    void deveRetornarAprovado() {
        stubFor(post(urlEqualTo("/fraud/clearsale"))
            .withRequestBody(containing("52998224725"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {"recomendacao": "APROVADO", "codigo": "CS_OK", "mensagem": "OK", "risco": 0}
                    """)));

        ClearSaleProvider provider = new ClearSaleProvider(restTemplate,
            "http://localhost:9999/fraud/clearsale", "test-key");

        FraudService.ResultadoFraude result = provider.analisar("52998224725", "Maria", "maria@teste.com", "11999999999");

        assertThat(result.aprovado()).isTrue();
        assertThat(result.codigo()).isEqualTo("CS_OK");
    }

    @Test
    void deveRetornarReprovado() {
        stubFor(post(urlEqualTo("/fraud/clearsale"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {"recomendacao": "REPROVADO", "codigo": "CS_REJECT", "mensagem": "Alto risco", "risco": 85}
                    """)));

        ClearSaleProvider provider = new ClearSaleProvider(restTemplate,
            "http://localhost:9999/fraud/clearsale", "test-key");

        FraudService.ResultadoFraude result = provider.analisar("52998224725", "Maria", "maria@teste.com", "11999999999");

        assertThat(result.aprovado()).isFalse();
        assertThat(result.risco()).isEqualTo(85);
    }

    @Test
    void deveAssumirAprovadoQuandoApiFalha() {
        stubFor(post(urlEqualTo("/fraud/clearsale"))
            .willReturn(aResponse().withStatus(500)));

        ClearSaleProvider provider = new ClearSaleProvider(restTemplate,
            "http://localhost:9999/fraud/clearsale", "test-key");

        FraudService.ResultadoFraude result = provider.analisar("52998224725", "Maria", "maria@teste.com", "11999999999");

        assertThat(result.aprovado()).isTrue();
        assertThat(result.codigo()).isEqualTo("FALHA_PROVEDOR");
    }
}
```

- [ ] **Step 3: Run tests**

```bash
mvn -pl aureus-onboarding test -Dtest="ClearSaleProviderTest,FraudStubTest" -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: BUILD SUCCESS, 4 tests pass

- [ ] **Step 4: Commit**

```bash
git add backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/ClearSaleProvider.java backend/aureus-onboarding/src/test/java/com/aureus/platform/onboarding/service/ClearSaleProviderTest.java
git commit -m "feat(onboarding): add ClearSaleProvider with WireMock test"
```

---
### Task 6: UnicoProvider + WireMock test

**Files:**
- Create: `backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/UnicoProvider.java`
- Create: `backend/aureus-onboarding/src/test/java/com/aureus/platform/onboarding/service/UnicoProviderTest.java`

- [ ] **Step 1: Write UnicoProvider**

```java
package com.aureus.platform.onboarding.service;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;
import java.util.List;

@Service
@Profile("producao")
public class UnicoProvider implements KycProviderService {
    @java.lang.SuppressWarnings("all")
    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(UnicoProvider.class);

    private final RestTemplate restTemplate;
    private final String url;
    private final String apiKey;

    public UnicoProvider(RestTemplate restTemplate,
                         @Value("${aureus.onboarding.kyc.unico.url}") String url,
                         @Value("${aureus.onboarding.kyc.unico.api-key}") String apiKey) {
        this.restTemplate = restTemplate;
        this.url = url;
        this.apiKey = apiKey;
    }

    @Override
    public ResultadoKyc validarDocumentos(String cpf, List<DocumentoInfo> documentos, String selfieBase64) {
        log.debug("Enviando documentos para Unico KYC, CPF {}", cpf);
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("X-API-Key", apiKey);
        var request = new HttpEntity<>(new UnicoRequest(cpf, documentos, selfieBase64), headers);
        try {
            UnicoResponse response = restTemplate.exchange(url, HttpMethod.POST, request, UnicoResponse.class).getBody();
            if (response == null) {
                throw new RuntimeException("Resposta nula da Unico");
            }
            return new ResultadoKyc(response.aprovado(), response.codigo(), "Unico: " + response.mensagem());
        } catch (Exception e) {
            log.warn("Erro ao consultar Unico KYC: {}", e.getMessage());
            throw e;
        }
    }

    @com.fasterxml.jackson.annotation.JsonIgnoreProperties(ignoreUnknown = true)
    record UnicoRequest(String cpf, List<DocumentoInfo> documentos, String selfieBase64) {}

    @com.fasterxml.jackson.annotation.JsonIgnoreProperties(ignoreUnknown = true)
    record UnicoResponse(boolean aprovado, String codigo, String mensagem) {}
}
```

- [ ] **Step 2: Write UnicoProviderTest**

```java
package com.aureus.platform.onboarding.service;

import com.github.tomakehurst.wiremock.junit5.WireMockTest;
import org.junit.jupiter.api.Test;
import org.springframework.web.client.RestTemplate;
import java.util.List;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

@WireMockTest(httpPort = 9999)
class UnicoProviderTest {

    private final RestTemplate restTemplate = new RestTemplate();

    @Test
    void deveAprovarKYC() {
        stubFor(post(urlEqualTo("/kyc/unico"))
            .withRequestBody(containing("52998224725"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {"aprovado": true, "codigo": "BIO_APROVADO", "mensagem": "Biometria OK"}
                    """)));

        UnicoProvider provider = new UnicoProvider(restTemplate,
            "http://localhost:9999/kyc/unico", "test-key");

        KycProviderService.ResultadoKyc result = provider.validarDocumentos(
            "52998224725", List.of(), "base64...");

        assertThat(result.aprovado()).isTrue();
        assertThat(result.codigo()).isEqualTo("BIO_APROVADO");
    }

    @Test
    void deveReprovarKYC() {
        stubFor(post(urlEqualTo("/kyc/unico"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {"aprovado": false, "codigo": "BIO_REJEITADO", "mensagem": "Face nao confere"}
                    """)));

        UnicoProvider provider = new UnicoProvider(restTemplate,
            "http://localhost:9999/kyc/unico", "test-key");

        KycProviderService.ResultadoKyc result = provider.validarDocumentos(
            "52998224725", List.of(), "base64...");

        assertThat(result.aprovado()).isFalse();
        assertThat(result.codigo()).isEqualTo("BIO_REJEITADO");
    }
}
```

- [ ] **Step 3: Run tests**

```bash
mvn -pl aureus-onboarding test -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: BUILD SUCCESS, all tests pass (including full suite)

- [ ] **Step 4: Commit**

```bash
git add backend/aureus-onboarding/src/main/java/com/aureus/platform/onboarding/service/UnicoProvider.java backend/aureus-onboarding/src/test/java/com/aureus/platform/onboarding/service/UnicoProviderTest.java
git commit -m "feat(onboarding): add UnicoProvider with WireMock test"
```

---
## Self-Review Checklist

- **Spec coverage:** All spec items covered:
  - BureauProvider interface + BureauGateway with fallback → Task 1
  - SerasaProvider → Task 2
  - QuodProvider → Task 3
  - FraudService + FraudStub → Task 4
  - ClearSaleProvider → Task 5
  - UnicoProvider → Task 6
  - BureauStub `@Profile` fix → Task 1
  - `riscoFraude` field + OnboardingPFService wiring → Task 4
  - Config files → Task 0
- **Placeholder scan:** No TBD/TODO found
- **Type consistency:** `ResultadoBureau`, `ResultadoKyc`, `ResultadoFraude` consistent across all tasks
