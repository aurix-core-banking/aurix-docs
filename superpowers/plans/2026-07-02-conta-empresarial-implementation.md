# Conta Empresarial Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add EMPRESARIAL as a new TipoConta value to enable business accounts for PJ clients.

**Architecture:** Single enum value addition to `Conta.TipoConta` (in `aurix-shared`) + validation in `ContaService.criarConta()` (in `aurix-core`) + populate `clienteTipoPessoa` on DTO response.

**Tech Stack:** Java 25, Spring Boot 4.1.0, JPA/Hibernate, H2 (tests), PostgreSQL (prod)

## Global Constraints

- All code in `apps/backend/` directory
- Follow delombok pattern: manual getters/setters with `@java.lang.SuppressWarnings("all")`, `canEqual()`, PRIME=59
- No Lombok annotations anywhere
- Tests in `aurix-core` use `@SpringBootTest(webEnvironment = RANDOM_PORT)` + RestTemplate + `@MockitoBean`
- Schema `aurix` for all tables
- API spec via springdoc-openapi

---

### Task 1: Add EMPRESARIAL to TipoConta + validation + tests

**Files:**
- Modify: `apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/entity/Conta.java` (add EMPRESARIAL to TipoConta enum)
- Modify: `apps/backend/aurix-core/src/main/java/com/aurix/platform/core/service/ContaService.java` (validation + DTO population)
- Modify: API spec via springdoc-openapi (add EMPRESARIAL to tipoConta enum)
- Create/test: `apps/backend/aurix-core/src/test/java/com/aurix/platform/core/integration/ContaEmpresarialIntegrationTest.java`

- [ ] **Step 1: Add EMPRESARIAL to TipoConta enum**

In `apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/entity/Conta.java`, line 163, after `SALARIO("Conta Salário")`:

```java
        /**
         * Conta Empresarial.
         */
        EMPRESARIAL("Conta Empresarial");
```

- [ ] **Step 2: Update ContaService.criarConta() — validate PJ requirement**

In `apps/backend/aurix-core/src/main/java/com/aurix/platform/core/service/ContaService.java`, after line 36 (loading cliente), add:

```java
        if (contaDTO.getTipoConta() == Conta.TipoConta.EMPRESARIAL
                && cliente.getTipoPessoa() != com.aurix.platform.shared.entity.Cliente.TipoPessoa.JURIDICA) {
            throw new IllegalArgumentException("Conta empresarial requer cliente pessoa juridica");
        }
```

- [ ] **Step 3: Populate clienteTipoPessoa in converterParaDTO**

In `converterParaDTO()` at line 240 (after `setStatus`), add:

```java
        dto.setClienteTipoPessoa(conta.getCliente().getTipoPessoa().name());
```

- [ ] **Step 4: Update API spec**

Add `EMPRESARIAL` to the `tipoConta` enum via springdoc-openapi. The exact location depends on the file — locate the `tipoConta` enum and add the value. For example:

```yaml
        tipoConta:
          type: string
          enum: [CORRENTE, POUPANCA, SALARIO, EMPRESARIAL]
```

- [ ] **Step 5: Create integration test**

Create `apps/backend/aurix-core/src/test/java/com/aurix/platform/core/integration/ContaEmpresarialIntegrationTest.java`:

```java
package com.aurix.platform.core.integration;

import com.aurix.platform.core.AurixCoreApplication;
import com.aurix.platform.shared.dto.ClienteDTO;
import com.aurix.platform.shared.dto.ContaDTO;
import com.aurix.platform.shared.entity.Conta;
import com.aurix.platform.shared.entity.Cliente;
import com.aurix.platform.shared.tenant.TenantContext;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.http.*;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.web.client.HttpClientErrorException;
import org.springframework.web.client.RestTemplate;

import java.math.BigDecimal;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.assertThrows;

@SpringBootTest(
    classes = AurixCoreApplication.class,
    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
    properties = "spring.security.enabled=false")
@ActiveProfiles("test")
class ContaEmpresarialIntegrationTest {

    @LocalServerPort
    private int port;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    private RestTemplate rest;
    private Long pjClienteId;
    private Long pfClienteId;

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
        pjClienteId = criarClientePJ();
        pfClienteId = criarClientePF();
    }

    private void limparBanco() {
        jdbcTemplate.execute("SET REFERENTIAL_INTEGRITY FALSE");
        for (String table : List.of("aurix.contas", "aurix.clientes")) {
            jdbcTemplate.execute("DELETE FROM " + table);
        }
        jdbcTemplate.execute("SET REFERENTIAL_INTEGRITY TRUE");
    }

    private String url(String path) {
        return "http://localhost:" + port + "/api/core" + path;
    }

    private Long criarClientePJ() {
        Map<String, Object> body = Map.of(
            "tipoPessoa", "JURIDICA",
            "cnpj", "12345678000190",
            "nomeRazaoSocial", "Empresa Exemplo Ltda",
            "email", "contato@empresa.com"
        );
        ResponseEntity<Map> response = rest.postForEntity(url("/clientes"), body, Map.class);
        return ((Number) response.getBody().get("id")).longValue();
    }

    private Long criarClientePF() {
        Map<String, Object> body = Map.of(
            "tipoPessoa", "FISICA",
            "cpf", "52998224725",
            "nome", "Joao Silva",
            "email", "joao@teste.com"
        );
        ResponseEntity<Map> response = rest.postForEntity(url("/clientes"), body, Map.class);
        return ((Number) response.getBody().get("id")).longValue();
    }

    @Test
    void criarContaEmpresarialParaPJ_deveRetornar201() {
        ContaDTO request = new ContaDTO();
        request.setClienteId(pjClienteId);
        request.setTipoConta(Conta.TipoConta.EMPRESARIAL);

        ResponseEntity<ContaDTO> response = rest.postForEntity(
            url("/contas"), request, ContaDTO.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().getTipoConta()).isEqualTo(Conta.TipoConta.EMPRESARIAL);
        assertThat(response.getBody().getClienteTipoPessoa()).isEqualTo("JURIDICA");
        assertThat(response.getBody().getNumeroConta()).isNotNull();
    }

    @Test
    void criarContaEmpresarialParaPF_deveRetornarErro() {
        ContaDTO request = new ContaDTO();
        request.setClienteId(pfClienteId);
        request.setTipoConta(Conta.TipoConta.EMPRESARIAL);

        assertThrows(HttpClientErrorException.class, () ->
            rest.postForEntity(url("/contas"), request, ContaDTO.class));
    }

    @Test
    void converterParaDTO_deveIncluirClienteTipoPessoa() {
        ContaDTO request = new ContaDTO();
        request.setClienteId(pjClienteId);
        request.setTipoConta(Conta.TipoConta.CORRENTE);

        ResponseEntity<ContaDTO> response = rest.postForEntity(
            url("/contas"), request, ContaDTO.class);

        assertThat(response.getBody().getClienteTipoPessoa()).isEqualTo("JURIDICA");
    }
}
```

- [ ] **Step 6: Build and verify all tests pass**

```bash
mvn test -pl aurix-core -am
```

Expected: Tests run: ?, Failures: 0, Errors: 0

Then verify onboarding tests still pass:

```bash
mvn test -pl aurix-onboarding -am
```

Expected: Tests run: 79, Failures: 0, Errors: 0, Skipped: 1

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "feat(core): add EMPRESARIAL TipoConta for business accounts"
```
