# Crédito PJ Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend `aurix-credit` module with PJ credit support (CNPJ bureau, PJ decision rules, PJ eligibility rules, Limite Rotativo).

**Architecture:** Reuse `SolicitacaoCredito` entity + workflow for loan products; add `consultarScoreCNPJ()` to `CreditBureauService`; extend `DecisaoCreditoService` with PJ rules; add PJ field extractors to catalog's `ElegibilidadeService`; add PJ financial fields to `Cliente` entity; Limite Rotativo via `Conta.limiteCredito`.

**Tech Stack:** Java 25, Spring Boot 4.1.0, JPA/Hibernate, H2 (tests), PostgreSQL (prod)

## Global Constraints

- All code in `apps/backend/` directory
- Follow delombok pattern: manual getters/setters with `@java.lang.SuppressWarnings("all")`, `canEqual()`, PRIME=59
- No Lombok annotations anywhere
- Tests use `@SpringBootTest(webEnvironment = RANDOM_PORT)` + RestTemplate + `@MockitoBean`
- Schema `aurix` for all tables
- `Cliente` entity in `aurix-shared`, `SolicitacaoCredito` in `aurix-credit`

---

### Task 1: ProdutoCredito + Cliente — add LIMITE_ROTATIVO and PJ financial fields

**Files:**
- Modify: `apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/entity/Cliente.java` (add PJ financial fields)
- Modify: `apps/backend/aurix-credit/src/main/java/com/aurix/platform/credit/entity/ProdutoCredito.java` (add LIMITE_ROTATIVO to TipoCredito)
- Create/test: `apps/backend/aurix-credit/src/test/java/com/aurix/platform/credit/entity/ProdutoCreditoTest.java`
- Create/test: `apps/backend/aurix-shared/src/test/java/com/aurix/platform/shared/entity/ClientePJFieldsTest.java`

- [ ] **Step 1: Add LIMITE_ROTATIVO to ProdutoCredito.TipoCredito**

In `ProdutoCredito.java`, find the `TipoCredito` enum (contains `PESSOAL, CONSIGNADO, CDC, VEICULOS, IMOBILIARIO, CAPITAL_GIRO, OUTROS`). Add:

```java
        CAPITAL_GIRO("Capital de Giro"),
        LIMITE_ROTATIVO("Limite Rotativo"),
        OUTROS("Outros");
```

- [ ] **Step 2: Add PJ financial fields to Cliente entity**

In `Cliente.java`, after the existing PJ fields (`inscricaoMunicipal`), add:

```java
    private BigDecimal faturamentoMensal;
    private BigDecimal capitalSocial;
    private String cnaePrincipal;
    private String porte;
    private LocalDate dataConstituicao;
```

And add corresponding getters, setters, equals/hashCode/toString updates following the existing delombok pattern.

- [ ] **Step 3: Create ClientePJFieldsTest**

```java
package com.aurix.platform.shared.entity;

import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.time.LocalDate;
import static org.junit.jupiter.api.Assertions.*;

class ClientePJFieldsTest {

    @Test
    void deveArmazenarCamposFinanceirosPJ() {
        Cliente cliente = new Cliente();
        cliente.setTipoPessoa(Cliente.TipoPessoa.JURIDICA);
        cliente.setCnpj("12345678000190");
        cliente.setNomeRazaoSocial("Empresa Ltda");
        cliente.setFaturamentoMensal(BigDecimal.valueOf(500000));
        cliente.setCapitalSocial(BigDecimal.valueOf(200000));
        cliente.setCnaePrincipal("62.01-3");
        cliente.setPorte("EPP");
        cliente.setDataConstituicao(LocalDate.of(2018, 1, 1));

        assertEquals(BigDecimal.valueOf(500000), cliente.getFaturamentoMensal());
        assertEquals(BigDecimal.valueOf(200000), cliente.getCapitalSocial());
        assertEquals("62.01-3", cliente.getCnaePrincipal());
        assertEquals("EPP", cliente.getPorte());
        assertEquals(LocalDate.of(2018, 1, 1), cliente.getDataConstituicao());
    }
}
```

- [ ] **Step 4: Build and commit**

```bash
mvn compile -pl aurix-shared,aurix-credit -am
mvn test -pl aurix-shared -Dtest=ClientePJFieldsTest -Dsurefire.failIfNoSpecifiedTests=false
git add -A && git commit -m "feat(credit): add LIMITE_ROTATIVO + PJ financial fields on Cliente"
```

---

### Task 2: CNPJ bureau scoring + PJ decision rules

**Files:**
- Modify: `apps/backend/aurix-credit/src/main/java/com/aurix/platform/credit/service/CreditBureauService.java` (add `consultarScoreCNPJ`)
- Modify: `apps/backend/aurix-credit/src/main/java/com/aurix/platform/credit/service/CreditBureauStub.java` (implement stub)
- Modify: `apps/backend/aurix-credit/src/main/java/com/aurix/platform/credit/service/DecisaoCreditoService.java` (PJ decision rules)
- Create/test: `apps/backend/aurix-credit/src/test/java/com/aurix/platform/credit/service/DecisaoCreditoPJTest.java`

- [ ] **Step 1: Add consultarScoreCNPJ to CreditBureauService**

```java
public interface CreditBureauService {
    // existing
    int consultarScore(String cpf);
    String consultarRelatorio(String cpf);
    // new
    ScoreCNPJResult consultarScoreCNPJ(String cnpj);

    record ScoreCNPJResult(int score, BigDecimal faturamentoEstimado,
        String risco, String mensagem) {
        public static ScoreCNPJResult ok(int score, BigDecimal faturamento, String risco) {
            return new ScoreCNPJResult(score, faturamento, risco, null);
        }
        public static ScoreCNPJResult erro(String cnpj, String erro) {
            return new ScoreCNPJResult(0, BigDecimal.ZERO, "INDEFINIDO", erro);
        }
    }
}
```

- [ ] **Step 2: Implement stub in CreditBureauStub**

```java
@Override
public ScoreCNPJResult consultarScoreCNPJ(String cnpj) {
    if (cnpj == null || cnpj.replaceAll("\\D", "").length() != 14) {
        return ScoreCNPJResult.erro(cnpj, "CNPJ invalido");
    }
    return ScoreCNPJResult.ok(750, BigDecimal.valueOf(500_000), "MEDIO");
}
```

- [ ] **Step 3: Add PJ rules to DecisaoCreditoService**

In `DecisaoCreditoService`, create a new method:
```java
public ResultadoDecisao decidirPJ(SolicitacaoCredito solicitacao, String tenantId) {
    Cliente cliente = solicitacao.getCliente();
    ScoreCNPJResult scoreResult = creditBureauService.consultarScoreCNPJ(cliente.getCnpj());

    if (scoreResult.mensagem() != null) {
        return new ResultadoDecisao("REJEITADA", "CNPJ invalido: " + scoreResult.mensagem(), null);
    }

    // Score thresholds for PJ
    if (scoreResult.score() >= 500) {
        // Capacity check: faturamento >= valor * 0.3
        if (cliente.getFaturamentoMensal() != null
            && cliente.getFaturamentoMensal().compareTo(
                solicitacao.getValorSolicitado().multiply(BigDecimal.valueOf(0.3))) < 0) {
            return new ResultadoDecisao("REFER", "Faturamento insuficiente para o valor solicitado",
                scoreResult.score());
        }
        return new ResultadoDecisao("APROVADA", "Aprovado com score " + scoreResult.score(),
            scoreResult.score());
    } else if (scoreResult.score() <= 300) {
        return new ResultadoDecisao("REJEITADA", "Score baixo: " + scoreResult.score(),
            scoreResult.score());
    } else {
        return new ResultadoDecisao("REFER", "Score intermediario: " + scoreResult.score(),
            scoreResult.score());
    }
}
```

Then update the main `decidir()` method to check `cliente.getTipoPessoa()` and call `decidirPJ()` for JURIDICA clients.

Create a `record ResultadoDecisao(String status, String motivo, Integer score)` inside the service if not already present.

- [ ] **Step 4: Build and commit**

```bash
mvn compile -pl aurix-credit -am && mvn test -pl aurix-credit -am
git add -A && git commit -m "feat(credit): add CNPJ bureau scoring + PJ decision rules"
```

---

### Task 3: PJ eligibility rules in catalog module

**Files:**
- Modify: `apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/service/ElegibilidadeService.java` (add PJ field extractors)
- Modify: `apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/entity/config/ConfigCredito.java` (add PJ fields)
- Modify: `apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/service/ConfigService.java` (handle new fields)
- Create/test: `apps/backend/aurix-catalog/src/test/java/com/aurix/platform/catalog/service/ElegibilidadePJTest.java`

- [ ] **Step 1: Add PJ field extractors to ElegibilidadeService**

In `ElegibilidadeService`, find the switch statement that extracts field values from `Cliente`. Add new cases:

```java
case "FATURAMENTO_MENSAL" -> String.valueOf(cliente.getFaturamentoMensal());
case "CAPITAL_SOCIAL" -> String.valueOf(cliente.getCapitalSocial());
case "CNAE_PRINCIPAL" -> cliente.getCnaePrincipal();
case "PORTE_EMPRESA" -> cliente.getPorte();
case "TEMPO_EXISTENCIA_MESES" -> {
    if (cliente.getDataConstituicao() != null) {
        long meses = ChronoUnit.MONTHS.between(cliente.getDataConstituicao(), LocalDate.now());
        yield String.valueOf(meses);
    }
    yield null;
}
```

- [ ] **Step 2: Add PJ fields to ConfigCredito**

```java
private BigDecimal faturamentoMinimo;
private Integer tempoConstituicaoMinimoMeses;
@Column(columnDefinition = "TEXT")
private String cnaePermitidos;
```

Add getters/setters following the delombok pattern.

- [ ] **Step 3: Update ConfigService**

In `ConfigService`, when saving/loading `ConfigCredito`, include the new PJ fields.

- [ ] **Step 4: Build and commit**

```bash
mvn compile -pl aurix-catalog -am && mvn test -pl aurix-catalog -am
git add -A && git commit -m "feat(catalog): add PJ eligibility rules + ConfigCredito PJ fields"
```

---

### Task 4: Core API — accept PJ financial data on cliente creation

**Files:**
- Modify: `apps/backend/aurix-core/src/main/java/com/aurix/platform/core/service/ClienteService.java` (accept new fields)
- Modify: `apps/backend/aurix-core/src/main/java/com/aurix/platform/core/controller/ClienteController.java` (accept new fields in request body)
- Modify: `apps/backend/aurix-onboarding/src/main/java/com/aurix/platform/onboarding/client/CoreApiClient.java` (send new fields)
- Modify: `apps/backend/aurix-onboarding/src/main/java/com/aurix/platform/onboarding/service/OnboardingPJService.java` (populate from onboarding data)
- Create/test: `apps/backend/aurix-core/src/test/java/com/aurix/platform/core/integration/ClientePJFinancialFieldsTest.java`

- [ ] **Step 1: Update ClienteController to accept new fields**

The `POST /clientes` endpoint receives a `Map<String, Object>` body. Add extraction for:
```java
BigDecimal faturamentoMensal = body.get("faturamentoMensal") != null
    ? new BigDecimal(body.get("faturamentoMensal").toString()) : null;
BigDecimal capitalSocial = body.get("capitalSocial") != null
    ? new BigDecimal(body.get("capitalSocial").toString()) : null;
String cnaePrincipal = (String) body.get("cnaePrincipal");
String porte = (String) body.get("porte");
String dataConstituicao = (String) body.get("dataConstituicao");
```

Pass these to `clienteService.criarCliente()`.

- [ ] **Step 2: Update ClienteService.criarCliente()**

Add overloaded parameters or pass them via the existing Map. Set them on the Cliente entity before saving.

- [ ] **Step 3: Update CoreApiClient.criarClientePJeConta()**

Add new parameters and include them in the body Map:
```java
Map<String, Object> body = new HashMap<>(Map.of(
    "tipoPessoa", "JURIDICA", "cnpj", cnpj,
    "nomeRazaoSocial", razaoSocial, "email", email,
    "telefone", telefone != null ? telefone : "",
    "endereco", endereco != null ? endereco : "{}"
));
if (faturamentoMensal != null) body.put("faturamentoMensal", faturamentoMensal);
if (capitalSocial != null) body.put("capitalSocial", capitalSocial);
if (cnaePrincipal != null) body.put("cnaePrincipal", cnaePrincipal);
if (porte != null) body.put("porte", porte);
if (dataConstituicao != null) body.put("dataConstituicao", dataConstituicao);
```

Update the method signature to accept these new parameters.

- [ ] **Step 4: Update OnboardingPJService.consultarCNPJ()**

After creating the `Empresa` entity and before calling `coreApiClient.criarClientePJeConta()`, populate financial data from `SolicitacaoPJ` + `Empresa`:

```java
SolicitacaoPJ pjData = solicitacaoPJRepository.findBySolicitacaoId(solicitacaoId).orElse(null);
// Then pass pjData.getFaturamentoMensal(), pjData.getCapitalSocial(), etc.
// to coreApiClient.criarClientePJeConta()
```

- [ ] **Step 5: Build and commit**

```bash
mvn compile -pl aurix-core,aurix-onboarding -am && mvn test -pl aurix-core,aurix-onboarding -am 2>&1 | tail -15
git add -A && git commit -m "feat(core): accept PJ financial fields on cliente creation + sync from onboarding"
```

---

### Task 5: Limite Rotativo endpoint

**Files:**
- Modify: `apps/backend/aurix-credit/src/main/java/com/aurix/platform/credit/controller/SolicitacaoCreditoController.java`
- Modify: `apps/backend/aurix-credit/src/main/java/com/aurix/platform/credit/service/SolicitacaoCreditoService.java`
- Create/test: `apps/backend/aurix-credit/src/test/java/com/aurix/platform/credit/service/LimiteRotativoServiceTest.java`

- [ ] **Step 1: Create ContaServiceClient in aurix-credit**

Create `apps/backend/aurix-credit/src/main/java/com/aurix/platform/credit/client/ContaServiceClient.java`:

```java
package com.aurix.platform.credit.client;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;
import java.math.BigDecimal;
import java.util.Map;

@Component
public class ContaServiceClient {
    private final RestTemplate restTemplate;
    @Value("${aurix.credit.core-api-url:http://localhost:8081/api/core}")
    private String coreApiUrl;

    public boolean atualizarLimiteCredito(Long contaId, BigDecimal novoLimite) {
        String url = coreApiUrl + "/contas/" + contaId + "/limite-credito";
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        HttpEntity<Map<String, Object>> request = new HttpEntity<>(
            Map.of("limiteCredito", novoLimite), headers);
        try {
            ResponseEntity<Void> response = restTemplate.exchange(url,
                HttpMethod.PUT, request, Void.class);
            return response.getStatusCode().is2xxSuccessful();
        } catch (Exception e) {
            return false;
        }
    }

    public ContaServiceClient(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }
}
```

- [ ] **Step 2: Add liberarLimiteRotativo to SolicitarCreditoService**

```java
public SolicitacaoCreditoDTO liberarLimiteRotativo(Long solicitacaoId) {
    String tenantId = TenantContext.getTenantId();
    SolicitacaoCredito solicitacao = solicitacaoCreditoRepository.findById(solicitacaoId)
        .orElseThrow(() -> new IllegalArgumentException("Solicitacao nao encontrada"));

    if (solicitacao.getStatus() != StatusSolicitacao.APROVADA) {
        throw new IllegalArgumentException("Solicitacao deve estar APROVADA para liberar limite");
    }

    // Call core API to update business account credit limit
    BigDecimal valor = solicitacao.getValorAprovado() != null
        ? solicitacao.getValorAprovado()
        : solicitacao.getValorSolicitado();
    
    // Find the business account associated with this cliente
    // (The credit module calls an admin endpoint that handles finding + updating the account)
    boolean success = contaServiceClient.atualizarLimiteCredito(
        solicitacao.getCliente().getId(), valor);
    
    if (!success) {
        throw new RuntimeException("Falha ao atualizar limite de credito na conta empresarial");
    }

    solicitacao.setStatus(StatusSolicitacao.LIBERADO);
    solicitacao = solicitacaoCreditoRepository.save(solicitacao);

    log.info("Limite rotativo liberado: solicitacao {}, valor {}",
        solicitacaoId, valor);

    return converterParaDTO(solicitacao);
}
```

- [ ] **Step 2: Add endpoint to SolicitarCreditoController**

```java
@PostMapping("/{id}/liberar-limite")
@Operation(summary = "Liberar limite rotativo aprovado")
public ResponseEntity<SolicitacaoCreditoDTO> liberarLimite(@PathVariable Long id) {
    return ResponseEntity.ok(solicitacaoCreditoService.liberarLimiteRotativo(id));
}
```

- [ ] **Step 3: Build and commit**

```bash
mvn compile -pl aurix-credit -am && mvn test -pl aurix-credit -am
git add -A && git commit -m "feat(credit): add liberar-limite-rotativo endpoint"
```

---

### Task 6: Integration tests

**Files:**
- Create: `apps/backend/aurix-credit/src/test/java/com/aurix/platform/credit/integration/CreditoPJFlowIntegrationTest.java`

- [ ] **Step 1: Create CreditoPJFlowIntegrationTest**

```java
package com.aurix.platform.credit.integration;

import com.aurix.platform.credit.AurixCreditApplication;
import com.aurix.platform.credit.client.ContaServiceClient;
import com.aurix.platform.credit.entity.ProdutoCredito;
import com.aurix.platform.credit.service.CreditBureauService;
import com.aurix.platform.shared.dto.SolicitacaoCreditoDTO;
import com.aurix.platform.shared.entity.*;
import com.aurix.platform.shared.tenant.TenantContext;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.http.*;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.web.client.RestTemplate;

import java.math.BigDecimal;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.when;

@SpringBootTest(
    classes = AurixCreditApplication.class,
    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
    properties = "spring.security.enabled=false")
@ActiveProfiles("test")
class CreditoPJFlowIntegrationTest {

    @LocalServerPort
    private int port;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @MockitoBean
    private CreditBureauService creditBureauService;

    @MockitoBean
    private ContaServiceClient contaServiceClient;

    private RestTemplate rest;
    private Long pjClienteId;

    @BeforeEach
    void setUp() {
        TenantContext.setTenantId(TenantContext.DEFAULT_TENANT_ID);
        limparBanco();
        rest = new RestTemplate();
        rest.setErrorHandler(new DefaultResponseErrorHandler() {
            @Override
            public boolean hasError(HttpStatusCode statusCode) {
                return false;
            }
        });

        when(creditBureauService.consultarScoreCNPJ(anyString()))
            .thenReturn(new CreditBureauService.ScoreCNPJResult(
                750, BigDecimal.valueOf(500_000), "MEDIO", null));
        when(contaServiceClient.atualizarLimiteCredito(any(), any()))
            .thenReturn(true);

        pjClienteId = criarClientePJ();
    }

    private void limparBanco() {
        jdbcTemplate.execute("SET REFERENTIAL_INTEGRITY FALSE");
        for (String table : List.of("aurix.solicitacoes_credito",
            "aurix.contas", "aurix.clientes")) {
            jdbcTemplate.execute("DELETE FROM " + table);
        }
        jdbcTemplate.execute("SET REFERENTIAL_INTEGRITY TRUE");
    }

    private String url(String path) {
        return "http://localhost:" + port + "/api/credit" + path;
    }

    private Long criarClientePJ() {
        // Create via JPA directly for test setup
        Cliente cliente = new Cliente();
        cliente.setTenantId(TenantContext.DEFAULT_TENANT_ID);
        cliente.setTipoPessoa(Cliente.TipoPessoa.JURIDICA);
        cliente.setCnpj("12345678000190");
        cliente.setNomeRazaoSocial("Empresa Ltda");
        cliente.setFaturamentoMensal(BigDecimal.valueOf(500_000));
        cliente.setCapitalSocial(BigDecimal.valueOf(200_000));
        cliente.setCnaePrincipal("62.01-3");
        cliente.setPorte("EPP");
        cliente.setDataConstituicao(java.time.LocalDate.of(2018, 1, 1));
        return jdbcTemplate.queryForObject(
            "INSERT INTO aurix.clientes (tenant_id, tipo_pessoa, cnpj, nome_razao_social, email, faturamento_mensal, capital_social, cnae_principal, porte, data_constituicao) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?) RETURNING id",
            Long.class, TenantContext.DEFAULT_TENANT_ID, "JURIDICA",
            "12345678000190", "Empresa Ltda", "contato@empresa.com",
            BigDecimal.valueOf(500_000), BigDecimal.valueOf(200_000),
            "62.01-3", "EPP", java.sql.Date.valueOf("2018-01-01"));
    }

    @Test
    void fluxoCapitalGiroPJ_deveAprovar() {
        // Create a CAPITAL_GIRO solicitation
        Map<String, Object> request = Map.of(
            "clienteId", pjClienteId,
            "valorSolicitado", 100000,
            "prazoMeses", 12,
            "produtoCreditoId", 1
        );

        ResponseEntity<SolicitacaoCreditoDTO> createResponse = rest.postForEntity(
            url("/solicitacoes"), request, SolicitacaoCreditoDTO.class);
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        Long solicId = createResponse.getBody().getId();

        // Execute decision
        ResponseEntity<SolicitacaoCreditoDTO> decisionResponse = rest.postForEntity(
            url("/decisao/" + solicId), null, SolicitacaoCreditoDTO.class);
        assertThat(decisionResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(decisionResponse.getBody().getStatus()).isEqualTo("APROVADA");
    }

    @Test
    void fluxoLimiteRotativo_deveLiberar() {
        Map<String, Object> request = Map.of(
            "clienteId", pjClienteId,
            "valorSolicitado", 50000,
            "prazoMeses", 0,
            "produtoCreditoId", 1
        );

        ResponseEntity<SolicitacaoCreditoDTO> createResponse = rest.postForEntity(
            url("/solicitacoes"), request, SolicitacaoCreditoDTO.class);
        Long solicId = createResponse.getBody().getId();

        // Approve
        rest.postForEntity(url("/decisao/" + solicId), null, SolicitacaoCreditoDTO.class);

        // Liberar limite
        ResponseEntity<SolicitacaoCreditoDTO> liberarResponse = rest.postForEntity(
            url("/solicitacoes/" + solicId + "/liberar-limite"),
            null, SolicitacaoCreditoDTO.class);
        assertThat(liberarResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(liberarResponse.getBody().getStatus()).isEqualTo("LIBERADO");
    }
}
```

- [ ] **Step 2: Build and verify**

```bash
mvn test -pl aurix-credit -am 2>&1 | tail -15
```

- [ ] **Step 3: Verify full regression**

```bash
mvn test -pl aurix-core,aurix-credit,aurix-catalog,aurix-onboarding -am 2>&1 | tail -15
```

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "test(credit): add PJ credit flow integration tests"
```
