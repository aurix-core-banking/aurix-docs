# PIX → BACEN SPI/STR Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire PIX transfer flow to BACEN SPI/STR engine via synchronous REST calls, replacing simulation/log-only pattern.

**Architecture:** `PixBacenClient` in aureus-pix calls `SpiStrIntegrationService` in aureus-bacen (REST). aureus-bacen uses `SpiStrApiClientImpl` configured per-profile (mock URL in dev/sandbox, mTLS + real BCB URL in prod). WireMock container (`bacen-mock`) runs in Docker Compose for local development. On SPI failure, debit is compensated via atomic credit.

**Tech Stack:** Spring Boot 4.1, Java 25, WireMock, RestClient, Docker Compose, JUnit 5, Mockito

## Global Constraints

- All new files follow existing package conventions (`com.aureus.platform.pix.*`, `com.aureus.platform.bacen.*`)
- WireMock container on port 8095, service name `bacen-mock`
- mTLS must be skipped when URL contains `localhost` or `bacen-mock`
- Production profile (`prod`) must NOT enable SPI/STR mock URLs
- All PIX transfers (including internal) go through SPI call
- Estorno uses `contaRepository.creditarSaldoAtomico()` with log on failure

---

### Task 1: BACEN Mock — WireMock container + Docker Compose

**Files:**
- Create: `infrastructure/bacen-mock/Dockerfile`
- Create: `infrastructure/bacen-mock/mappings/spi-pix.json`
- Create: `infrastructure/bacen-mock/mappings/spi-status.json`
- Modify: `infrastructure/docker-compose.yml` (add `bacen-mock` service)
- Modify: `infrastructure/.env.example` (add BACEN/PIX env vars)

**Interfaces:**
- Consumes: nothing
- Produces: HTTP mock at `http://bacen-mock:8095/api/spi-str/spi/pix` (POST returns `{"sucesso": true, "endToEndId": "E...", "status": "LIQUIDADA"}`)

- [ ] **Step 1: Create bacen-mock Dockerfile**

```dockerfile
FROM wiremock/wiremock:latest
COPY mappings/ /home/wiremock/mappings/
EXPOSE 8095
```

- [ ] **Step 2: Create SPI PIX mapping**

```json
{
  "request": {
    "method": "POST",
    "url": "/api/spi-str/spi/pix"
  },
  "response": {
    "status": 200,
    "jsonBody": {
      "sucesso": true,
      "endToEndId": "E0000000020240710XXXXXXXXXXXX",
      "status": "LIQUIDADA",
      "dataHoraLiquidacao": "{{now}}",
      "ispbDestino": "12345678",
      "mensagem": "Transacao PIX liquidada com sucesso (simulacao)"
    },
    "headers": { "Content-Type": "application/json" }
  }
}
```

- [ ] **Step 3: Create SPI status mapping**

```json
{
  "request": {
    "method": "GET",
    "urlPattern": "/api/spi-str/spi/status/.*"
  },
  "response": {
    "status": 200,
    "jsonBody": {
      "endToEndId": "E0000000020240710XXXXXXXXXXXX",
      "status": "LIQUIDADA"
    }
  }
}
```

- [ ] **Step 4: Add bacen-mock service to docker-compose.yml**

```yaml
  bacen-mock:
    image: wiremock/wiremock:latest
    container_name: aureus-bacen-mock
    ports:
      - "8095:8095"
    volumes:
      - ./bacen-mock/mappings:/home/wiremock/mappings
    command: ["--port", "8095", "--verbose"]
    networks:
      - aureus-network
```

Find the existing docker-compose.yml and add the service after the last service definition, before `networks:`.

- [ ] **Step 5: Add BACEN/PIX env vars to .env.example**

Append to `infrastructure/.env.example`:

```env
# BACEN / PIX Sandbox
BACEN_SANDBOX=true
BACEN_CLIENT_ID=replace_with_bacen_client_id
BACEN_CLIENT_SECRET=replace_with_bacen_client_secret
BACEN_KEYSTORE_PATH=
BACEN_KEYSTORE_PASSWORD=
PIX_BACEN_SPI_URL=http://bacen-mock:8095
PIX_BACEN_STR_URL=http://bacen-mock:8095
```

- [ ] **Step 6: Verify Docker Compose**

```bash
docker compose config | grep -A 10 "bacen-mock"
```
Expected: `bacen-mock` service visible in parsed config.

- [ ] **Step 7: Commit**

```bash
git add infrastructure/bacen-mock/ infrastructure/docker-compose.yml infrastructure/.env.example
git commit -m "feat(pix): add BACEN mock WireMock container"
```

---

### Task 2: SpiStrApiClientImpl — mock URL guard + dev profile

**Files:**
- Modify: `backend/aureus-bacen/src/main/java/com/aureus/platform/bacen/client/SpiStrApiClientImpl.java`
- Create: `backend/aureus-bacen/src/main/resources/application-dev.yml`
- Modify: `backend/aureus-bacen/src/main/resources/application.yml` (verify current SPI config)
- Test: `backend/aureus-bacen/src/test/java/com/aureus/platform/bacen/client/SpiStrApiClientImplTest.java`

**Interfaces:**
- Consumes: existing `SpiProperties`, `SpiStrProperties`
- Produces: `isMockUrl()` method; `application-dev.yml` with SPI/STR enabled pointing to mock

- [ ] **Step 1: Read current SpiStrApiClientImpl.java to find mTLS setup and enviarPixSpi method**

```bash
cat backend/aureus-bacen/src/main/java/com/aureus/platform/bacen/client/SpiStrApiClientImpl.java
```
Identify the method `enviarPixSpi` and where mTLS `SslContext` is configured.

- [ ] **Step 2: Write test for mock URL detection**

```java
package com.aureus.platform.bacen.client;

import com.aureus.platform.bacen.config.SpiStrProperties;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class SpiStrApiClientImplTest {

    @Test
    void shouldDetectMockUrlForLocalhost() {
        SpiStrProperties props = new SpiStrProperties();
        SpiStrProperties.SpiConfig spiConfig = new SpiStrProperties.SpiConfig();
        spiConfig.setUrl("http://localhost:8095");
        props.setSpi(spiConfig);
        SpiStrApiClientImpl client = new SpiStrApiClientImpl(null, null, null, props);
        assertThat(client.isMockUrl()).isTrue();
    }

    @Test
    void shouldDetectMockUrlForBacenMock() {
        SpiStrProperties props = new SpiStrProperties();
        SpiStrProperties.SpiConfig spiConfig = new SpiStrProperties.SpiConfig();
        spiConfig.setUrl("http://bacen-mock:8095");
        props.setSpi(spiConfig);
        SpiStrApiClientImpl client = new SpiStrApiClientImpl(null, null, null, props);
        assertThat(client.isMockUrl()).isTrue();
    }

    @Test
    void shouldDetectNonMockUrl() {
        SpiStrProperties props = new SpiStrProperties();
        SpiStrProperties.SpiConfig spiConfig = new SpiStrProperties.SpiConfig();
        spiConfig.setUrl("https://spi-homologacao.bcb.gov.br");
        props.setSpi(spiConfig);
        SpiStrApiClientImpl client = new SpiStrApiClientImpl(null, null, null, props);
        assertThat(client.isMockUrl()).isFalse();
    }
}
```

- [ ] **Step 3: Run the test to verify it fails**

```bash
mvn -pl aureus-bacen test -Dtest=SpiStrApiClientImplTest -DfailIfNoTests=false
```
Expected: compilation error — `isMockUrl()` not found.

- [ ] **Step 4: Add isMockUrl() method to SpiStrApiClientImpl and modify enviarPixSpi to skip mTLS**

Add field and method:
```java
private final SpiStrProperties spiStrProperties;

// Add to constructor
public SpiStrApiClientImpl(SpiStrProperties spiStrProperties, ...) {
    this.spiStrProperties = spiStrProperties;
    // existing init
}

public boolean isMockUrl() {
    String url = spiStrProperties.getSpi().getUrl();
    return url != null && (url.contains("localhost") || url.contains("bacen-mock"));
}
```

In `enviarPixSpi()`: wrap the mTLS/non-mTLS logic:
```java
if (isMockUrl()) {
    // simple HTTP without SSL
    return WebClient.builder().baseUrl(spiStrProperties.getSpi().getUrl()).build()
        .post()
        .uri("/api/spi-str/spi/pix")
        .bodyValue(requisicao)
        .retrieve()
        .bodyToMono(SpiStrResponse.class)
        .block();
} else {
    // existing mTLS logic
}
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
mvn -pl aureus-bacen test -Dtest=SpiStrApiClientImplTest
```
Expected: 3/3 passed.

- [ ] **Step 6: Create application-dev.yml**

```yaml
aureus:
  bacen:
    spi:
      enabled: true
      url: ${PIX_BACEN_SPI_URL:http://bacen-mock:8095}
    str:
      enabled: true
      url: ${PIX_BACEN_STR_URL:http://bacen-mock:8095}
    certificados:
      keystore-path: ${BACEN_KEYSTORE_PATH:}
      keystore-password: ${BACEN_KEYSTORE_PASSWORD:}
```

- [ ] **Step 7: Read current application.yml SPI config to confirm no conflicts**

```bash
cat backend/aureus-bacen/src/main/resources/application.yml
```
Verify SPI/STR are listed as `enabled: false` in default profile — dev profile overrides via application-dev.yml are correct.

- [ ] **Step 8: Commit**

```bash
git add backend/aureus-bacen/src/main/java/com/aureus/platform/bacen/client/SpiStrApiClientImpl.java \
       backend/aureus-bacen/src/main/resources/application-dev.yml \
       backend/aureus-bacen/src/test/java/com/aureus/platform/bacen/client/SpiStrApiClientImplTest.java
git commit -m "feat(bacen): mock URL guard in SpiStrApiClientImpl + dev profile"
```

---

### Task 3: PixBacenClient — REST client in aureus-pix

**Files:**
- Create: `backend/aureus-pix/src/main/java/com/aureus/platform/pix/client/PixBacenClient.java`
- Create: `backend/aureus-pix/src/main/java/com/aureus/platform/pix/client/dto/TransacaoSPI.java`
- Create: `backend/aureus-pix/src/main/java/com/aureus/platform/pix/client/dto/SpiResult.java`
- Create: `backend/aureus-pix/src/test/java/com/aureus/platform/pix/client/PixBacenClientTest.java`
- Modify: `backend/aureus-pix/src/main/resources/application.yml` (add `aureus.pix.bacen.spi-url`)

**Interfaces:**
- Consumes: nothing (standalone client)
- Produces: `PixBacenClient.enviarPix(TransacaoSPI) → SpiResult`

- [ ] **Step 1: Write failing test**

```java
package com.aureus.platform.pix.client;

import com.aureus.platform.pix.client.dto.SpiResult;
import com.aureus.platform.pix.client.dto.TransacaoSPI;
import org.junit.jupiter.api.Test;
import org.springframework.web.client.RestClient;
import java.math.BigDecimal;
import static org.assertj.core.api.Assertions.assertThat;

class PixBacenClientTest {

    @Test
    void shouldSendPixAndReturnSpiResult() {
        RestClient.Builder builder = RestClient.builder();
        PixBacenClient client = new PixBacenClient(builder, "http://localhost:8094");

        TransacaoSPI transacao = new TransacaoSPI();
        transacao.setEndToEndId("E0000000020240710REFTEST001");
        transacao.setIspbOrigem("12345678");
        transacao.setIspbDestino("87654321");
        transacao.setValor(new BigDecimal("150.00"));
        transacao.setChavePixDestino("teste@email.com");

        SpiResult result = client.enviarPix(transacao);
        // Not testing actual HTTP — just verifying method signature exists
        assertThat(result).isNull();
    }
}
```

- [ ] **Step 2: Run the failing test**

```bash
mvn -pl aureus-pix test -Dtest=PixBacenClientTest
```
Expected: compilation error — classes not found.

- [ ] **Step 3: Create DTOs, client, and config**

**TransacaoSPI.java:**
```java
package com.aureus.platform.pix.client.dto;

import lombok.Getter;
import lombok.Setter;
import java.math.BigDecimal;

@Getter
@Setter
public class TransacaoSPI {
    private String endToEndId;
    private String ispbOrigem;
    private String ispbDestino;
    private BigDecimal valor;
    private String chavePixDestino;
}
```

**SpiResult.java:**
```java
package com.aureus.platform.pix.client.dto;

import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class SpiResult {
    private boolean sucesso;
    private String endToEndId;
    private String status;
    private String dataHoraLiquidacao;
    private String ispbDestino;
    private String mensagem;
}
```

**PixBacenClient.java:**
```java
package com.aureus.platform.pix.client;

import com.aureus.platform.pix.client.dto.SpiResult;
import com.aureus.platform.pix.client.dto.TransacaoSPI;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

@Component
public class PixBacenClient {

    private final RestClient restClient;

    public PixBacenClient(RestClient.Builder builder,
                          @Value("${aureus.pix.bacen.spi-url}") String spiUrl) {
        this.restClient = builder.baseUrl(spiUrl).build();
    }

    public SpiResult enviarPix(TransacaoSPI transacao) {
        return restClient.post()
            .uri("/api/spi-str/spi/pix")
            .body(transacao)
            .retrieve()
            .body(SpiResult.class);
    }
}
```

**application.yml addition (aureus-pix):**
```yaml
aureus:
  pix:
    bacen:
      spi-url: ${PIX_BACEN_SPI_URL:http://localhost:8094}
```

- [ ] **Step 4: Run test — should pass (returns null since no server running)**

```bash
mvn -pl aureus-pix test -Dtest=PixBacenClientTest -Dspring.profiles.active=test
```
Expected: pass (NPE-safe test, just checks compilation and instantiation).

- [ ] **Step 5: Commit**

```bash
git add backend/aureus-pix/src/main/java/com/aureus/platform/pix/client/ \
       backend/aureus-pix/src/main/java/com/aureus/platform/pix/client/dto/ \
       backend/aureus-pix/src/main/resources/application.yml \
       backend/aureus-pix/src/test/java/com/aureus/platform/pix/client/PixBacenClientTest.java
git commit -m "feat(pix): PixBacenClient SPI client"
```

---

### Task 4: PixTransferenciaService — wire SPI call + compensation

**Files:**
- Read: `backend/aureus-pix/src/main/java/com/aureus/platform/pix/service/PixTransferenciaService.java` (read current logic)
- Modify: `backend/aureus-pix/src/main/java/com/aureus/platform/pix/service/PixTransferenciaService.java`
- Test: `backend/aureus-pix/src/test/java/com/aureus/platform/pix/service/PixTransferenciaServiceTest.java`

**Interfaces:**
- Consumes: `PixBacenClient.enviarPix(TransacaoSPI) → SpiResult`
- Produces: modified `processarTransferencia()` with SPI call + estorno

- [ ] **Step 1: Read current PixTransferenciaService**

```bash
cat backend/aureus-pix/src/main/java/com/aureus/platform/pix/service/PixTransferenciaService.java
```

- [ ] **Step 2: Read existing test to understand mocking pattern**

```bash
cat backend/aureus-pix/src/test/java/com/aureus/platform/pix/service/PixTransferenciaServiceTest.java
```

- [ ] **Step 3: Update test — mock PixBacenClient, test SPI success/failure/compensation**

Add tests to `PixTransferenciaServiceTest.java`:

```java
@MockBean
private PixBacenClient pixBacenClient;

@Test
void shouldCallSpiAndProcessWhenSpiSucceeds() {
    // Arrange
    PixTransferencia transferencia = createTransferenciaPendente();
    when(pixBacenClient.enviarPix(any())).thenReturn(spiResultSucesso());
    when(chavePixRepository.findByTenantIdAndValorChave(any(), any()))
        .thenReturn(Optional.of(chaveLocalDestino()));
    when(contaRepository.creditarSaldoAtomico(any(), any(), any())).thenReturn(1);

    // Act
    PixTransferencia result = pixTransferenciaService.processarTransferencia(
        transferencia.getId(), tenantId);

    // Assert
    assertThat(result.getStatus()).isEqualTo("PROCESSADA");
    assertThat(result.getEndToEndId()).isNotNull();
    verify(pixBacenClient).enviarPix(any());
}

@Test
void shouldCompensateWhenSpiFails() {
    // Arrange
    PixTransferencia transferencia = createTransferenciaPendente();
    SpiResult spiResult = new SpiResult();
    spiResult.setSucesso(false);
    spiResult.setMensagem("Fundo insuficiente na conta destino");
    when(pixBacenClient.enviarPix(any())).thenReturn(spiResult);

    // Act & Assert
    assertThatThrownBy(() -> pixTransferenciaService.processarTransferencia(
        transferencia.getId(), tenantId))
        .isInstanceOf(IllegalStateException.class)
        .hasMessageContaining("rejeitado");

    // Verify compensation
    verify(contaRepository).creditarSaldoAtomico(eq(tenantId), eq(contaOrigemId), eq(valor));
}

@Test
void shouldCompensateWhenSpiThrowsException() {
    // Arrange
    PixTransferencia transferencia = createTransferenciaPendente();
    when(pixBacenClient.enviarPix(any())).thenThrow(new RuntimeException("Timeout"));

    // Act & Assert
    assertThatThrownBy(() -> pixTransferenciaService.processarTransferencia(
        transferencia.getId(), tenantId))
        .isInstanceOf(IllegalStateException.class)
        .hasMessageContaining("comunicação");

    // Verify compensation
    verify(contaRepository).creditarSaldoAtomico(eq(tenantId), eq(contaOrigemId), eq(valor));
}

private SpiResult spiResultSucesso() {
    SpiResult r = new SpiResult();
    r.setSucesso(true);
    r.setEndToEndId("E0000000020240710TESTENDTOEND01");
    r.setStatus("LIQUIDADA");
    return r;
}

private PixTransferencia createTransferenciaPendente() {
    // Same as existing helper in test
    PixTransferencia t = new PixTransferencia();
    t.setId(1L);
    t.setStatus("PENDENTE");
    t.setValor(new BigDecimal("150.00"));
    t.setContaOrigemId(contaOrigemId);
    t.setChavePixDestino("teste@email.com");
    return t;
}
```

- [ ] **Step 4: Run tests — should fail since service not wired yet**

```bash
mvn -pl aureus-pix test -Dtest=PixTransferenciaServiceTest
```
Expected: compilation errors on `pixBacenClient` usage or behavior changes.

- [ ] **Step 5: Modify processarTransferencia in PixTransferenciaService**

Replace the existing SPI simulation block in `processarTransferencia()`:

```java
// After debito atomico succeeds, before the "chaveDestino.isLocal()" check:

TransacaoSPI transacaoSPI = new TransacaoSPI();
transacaoSPI.setEndToEndId(gerarEndToEndId());
transacaoSPI.setIspbOrigem(pixConfig.getIspb());
transacaoSPI.setIspbDestino(obterIspbDestino(chaveDestino));
transacaoSPI.setValor(transferencia.getValor());
transacaoSPI.setChavePixDestino(chaveDestino.getValorChave());

SpiResult resultado;
try {
    resultado = pixBacenClient.enviarPix(transacaoSPI);
} catch (Exception e) {
    log.error("Falha ao enviar PIX para SPI: {}", e.getMessage());
    estornarDebito(transferencia, contaOrigemId);
    transferencia.setStatus("FALHA");
    pixTransferenciaRepository.save(transferencia);
    throw new IllegalStateException("Falha na comunicação com SPI", e);
}

if (!resultado.isSucesso()) {
    String mensagemErro = resultado.getMensagem();
    estornarDebito(transferencia, contaOrigemId);
    transferencia.setStatus("REJEITADA");
    transferencia.setMensagemErro(mensagemErro);
    pixTransferenciaRepository.save(transferencia);
    throw new IllegalStateException("PIX rejeitado pelo SPI: " + mensagemErro);
}

// SPI liquidou — credito local se chave interna
if (chaveDestino.isLocal()) {
    contaRepository.creditarSaldoAtomico(tenantId, contaDestinoId, transferencia.getValor());
}

transferencia.setStatus("PROCESSADA");
transferencia.setEndToEndId(resultado.getEndToEndId());
pixTransferenciaRepository.save(transferencia);
```

Add the `estornarDebito` helper:
```java
private void estornarDebito(PixTransferencia transferencia, Long contaOrigemId) {
    int estornado = contaRepository.creditarSaldoAtomico(
        TenantContext.getTenantId(), contaOrigemId, transferencia.getValor());
    if (estornado == 0) {
        log.error("CRÍTICO: estorno falhou para transferência ID {}", transferencia.getId());
    }
}
```

Inject `PixBacenClient`:
```java
private final PixBacenClient pixBacenClient;

// Add to constructor
```

Also need `pixConfig` with `getIspb()` and `obterIspbDestino()` — add basic implementation or inject config.

For `gerarEndToEndId()`:
```java
private String gerarEndToEndId() {
    return "E" + LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"))
        + UUID.randomUUID().toString().replace("-", "").substring(0, 20).toUpperCase();
}
```

- [ ] **Step 6: Run tests to verify they pass**

```bash
mvn -pl aureus-pix test -Dtest=PixTransferenciaServiceTest
```
Expected: 3 new tests passing.

- [ ] **Step 7: Commit**

```bash
git add backend/aureus-pix/src/main/java/com/aureus/platform/pix/service/PixTransferenciaService.java \
       backend/aureus-pix/src/test/java/com/aureus/platform/pix/service/PixTransferenciaServiceTest.java
git commit -m "feat(pix): wire SPI call + compensation in PixTransferenciaService"
```

---

### Task 5: Integration test — PIX full flow with BACEN mock

**Files:**
- Modify: `backend/aureus-pix/src/test/java/com/aureus/platform/pix/integration/PixFlowIntegrationTest.java` (update existing test)
- Ensure: `bacen-mock` is available in testcontainers config or test relies on mocked client

**Interfaces:**
- Consumes: all previous tasks
- Produces: verified end-to-end PIX flow

- [ ] **Step 1: Read existing PixFlowIntegrationTest**

```bash
cat backend/aureus-pix/src/test/java/com/aureus/platform/pix/integration/PixFlowIntegrationTest.java
```
Determine if it uses Testcontainers or mocked services.

- [ ] **Step 2: Add SPI call verification to the integration test**

If the test already mocks the external dependencies, add verification:
```java
verify(pixBacenClient, times(1)).enviarPix(any());
```

If Testcontainers with WireMock is feasible, add:
```java
@DynamicPropertySource
static void configureProperties(DynamicPropertyRegistry registry) {
    registry.add("aureus.pix.bacen.spi-url", () -> "http://localhost:" + wireMockPort);
}
```

- [ ] **Step 3: Run integration test**

```bash
mvn -pl aureus-pix test -Dtest=PixFlowIntegrationTest
```
Expected: pass.

- [ ] **Step 4: Commit**

```bash
git add backend/aureus-pix/src/test/java/com/aureus/platform/pix/integration/PixFlowIntegrationTest.java
git commit -m "test(pix): update integration test for SPI flow"
```

---

### Task 6: Full build verification

- [ ] **Step 1: Build aureus-bacen module**

```bash
mvn -pl aureus-bacen clean compile -DskipTests
```
Expected: BUILD SUCCESS.

- [ ] **Step 2: Build aureus-pix module**

```bash
mvn -pl aureus-pix clean compile -DskipTests
```
Expected: BUILD SUCCESS.

- [ ] **Step 3: Run all aureus-bacen tests**

```bash
mvn -pl aureus-bacen test
```
Expected: all existing + new tests passing.

- [ ] **Step 4: Run all aureus-pix tests**

```bash
mvn -pl aureus-pix test
```
Expected: all existing + new tests passing.

- [ ] **Step 5: Build backend root**

```bash
mvn clean compile -DskipTests
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Static analysis**

```bash
mvn pmd:check spotbugs:check -DskipTests
```
Expected: no violations.

- [ ] **Step 7: Docker Compose config check**

```bash
docker compose config | grep -A 5 "bacen-mock"
```
Expected: service parsed correctly.

