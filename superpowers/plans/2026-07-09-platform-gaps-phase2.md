# Platform Gaps Phase 2 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fill 5 remaining platform gaps: OpenFinance stubs, auth/password flows, Kafka staging infra, missing web pages, ML/BI/Chatbot stub refactoring.

**Architecture:** 5 independent gaps, zero cross-dependencies. Each gap can be built in any order. Backend gaps (1, 2, 5) use Spring Boot 4.1 / Java 25 patterns. Infra gap (3) is Docker Compose. Frontend gap (4) uses React + MUI 5 patterns from existing pages.

**Tech Stack:** Java 25, Spring Boot 4.1, Maven (backend); React, MUI 5, react-router-dom, numeral, date-fns (frontend); Docker Compose, Kafka 7.4.0 (infra)

## Global Constraints

- All Java code: Project Lombok (`@Slf4j`), constructor injection, `TenantContext.getTenantId()`
- All Java exceptions: `IllegalArgumentException` for business errors, `IllegalStateException` for state errors
- Backend tests: JUnit 5 + Testcontainers (require Docker)
- Frontend pages: functional components, `export default function`, MUI 5, Portuguese labels, `apiService` for API calls, `numeral` + `date-fns` with `ptBR` locale
- Frontend tests: co-located `.test.js` files, Jest + React Testing Library
- Commit messages: conventional commits (`feat:`, `fix:`, `refactor:`, etc.)
- Profile naming: analytics uses `@Profile("!prod")`, credit uses `@Profile("!producao")` (keep existing convention)

---

## File Map

### Gap 1 — OpenFinance
- Modify: `backend/aureus-openfinance/src/main/java/com/aureus/platform/openfinance/controller/OpenFinanceController.java` (lines 93, 115)
- Modify: `backend/aureus-openfinance/src/main/java/com/aureus/platform/openfinance/service/OpenFinanceDataService.java`
- Create/test: `backend/aureus-openfinance/src/test/java/com/aureus/platform/openfinance/service/OpenFinanceDataServiceTest.java`
- Create/test: `backend/aureus-openfinance/src/test/java/com/aureus/platform/openfinance/controller/OpenFinanceControllerTest.java`

### Gap 2 — Auth/Password
- Create: `backend/aureus-security/src/main/java/com/aureus/platform/security/entity/PasswordResetToken.java`
- Create: `backend/aureus-security/src/main/java/com/aureus/platform/security/entity/RefreshToken.java`
- Create: `backend/aureus-security/src/main/java/com/aureus/platform/security/repository/PasswordResetTokenRepository.java`
- Create: `backend/aureus-security/src/main/java/com/aureus/platform/security/repository/RefreshTokenRepository.java`
- Create: `backend/aureus-security/src/main/java/com/aureus/platform/security/dto/ForgotPasswordRequestDTO.java`
- Create: `backend/aureus-security/src/main/java/com/aureus/platform/security/dto/ResetPasswordRequestDTO.java`
- Create: `backend/aureus-security/src/main/java/com/aureus/platform/security/dto/RefreshTokenRequestDTO.java`
- Modify: `backend/aureus-security/src/main/java/com/aureus/platform/security/controller/AuthController.java`
- Modify: `backend/aureus-security/src/main/java/com/aureus/platform/security/service/AuthService.java`
- Create/test: `backend/aureus-security/src/test/java/com/aureus/platform/security/service/AuthServiceTest.java`
- Create/test: `backend/aureus-security/src/test/java/com/aureus/platform/security/controller/AuthControllerTest.java`

### Gap 3 — Kafka Staging
- Modify: `aureus-data-platform/kafka/docker-compose.staging.yml`
- Modify: `aureus-data-platform/kafka/.env.example`

### Gap 4 — Web Pages
- Create: `frontend/aureus-web/src/pages/Extrato.js`
- Create: `frontend/aureus-web/src/pages/Extrato.test.js`
- Create: `frontend/aureus-web/src/pages/Transferencia.js`
- Create: `frontend/aureus-web/src/pages/Transferencia.test.js`
- Create: `frontend/aureus-web/src/pages/Pagamento.js`
- Create: `frontend/aureus-web/src/pages/Pagamento.test.js`
- Create: `frontend/aureus-web/src/pages/Recarga.js`
- Create: `frontend/aureus-web/src/pages/Recarga.test.js`
- Modify: `frontend/aureus-web/src/App.js`
- Modify: `frontend/aureus-web/src/components/Sidebar.js`

### Gap 5 — ML/BI/Chatbot
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/FraudService.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/CreditScoreService.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/ChatbotService.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/BiService.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/MlFraudServiceStub.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/CreditScoreStubService.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/ChatbotStubService.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/BiStubService.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/MlFraudServiceProd.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/CreditScoreServiceProd.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/ChatbotServiceProd.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/BiServiceProd.java`
- Modify/rename: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/controller/MlStubController.java` → `MlController.java`
- Modify/rename: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/controller/ChatbotStubController.java` → `ChatbotController.java`
- Modify/rename: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/controller/BiStubController.java` → `BiController.java`
- Modify: `backend/aureus-analytics/src/test/java/com/aureus/platform/analytics/integration/AnalyticsFlowIntegrationTest.java`

---

## Independent Tasks

Each gap is fully independent. They can be executed in parallel by separate agents.

---

### Task G1-1: OpenFinanceDataService — add credit cards and identifications methods

**Files:**
- Modify: `backend/aureus-openfinance/src/main/java/com/aureus/platform/openfinance/service/OpenFinanceDataService.java`
- Test: `backend/aureus-openfinance/src/test/java/com/aureus/platform/openfinance/service/OpenFinanceDataServiceTest.java`

**Interfaces:**
- Consumes: existing `TokenOpenFinance`, `ConsentimentoOpenFinance`, `CoreApiClient`
- Produces: `List<Map<String, Object>> listarCartoesCreditoPorToken(TokenOpenFinance)`, `List<Map<String, Object>> listarIdentificacoesPessoaisPorToken(TokenOpenFinance)`

- [ ] **Step 1: Read existing OpenFinanceDataService.java**

Read the file to understand existing patterns.

- [ ] **Step 2: Write failing test for listarCartoesCreditoPorToken**

```java
package com.aureus.platform.openfinance.service;

import com.aureus.platform.openfinance.entity.TokenOpenFinance;
import com.aureus.platform.openfinance.entity.ConsentimentoOpenFinance;
import com.aureus.platform.shared.client.CoreApiClient;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class OpenFinanceDataServiceTest {

    @Mock private CoreApiClient coreApiClient;
    private OpenFinanceDataService service;

    @BeforeEach
    void setUp() {
        service = new OpenFinanceDataService(coreApiClient);
    }

    @Test
    void shouldListCreditCardsForAuthorizedAccounts() {
        var token = new TokenOpenFinance();
        token.setConsentId(1L);
        var consent = new ConsentimentoOpenFinance();
        consent.setPermissions(Set.of(ConsentimentoOpenFinance.TipoConsentimento.CREDIT_CARDS_ACCOUNTS));
        consent.setContasAutorizadas(Set.of("acc-1", "acc-2"));
        when(coreApiClient.obterConsentimento(1L)).thenReturn(consent);
        when(coreApiClient.getCreditCards(List.of("acc-1", "acc-2")))
            .thenReturn(List.of(Map.of("id", "card-1", "brand", "VISA")));

        List<Map<String, Object>> result = service.listarCartoesCreditoPorToken(token);

        assertNotNull(result);
        assertEquals(1, result.size());
        assertEquals("VISA", result.get(0).get("brand"));
        verify(coreApiClient).getCreditCards(List.of("acc-1", "acc-2"));
    }

    @Test
    void shouldReturnEmptyWhenNoCreditCardsPermission() {
        var token = new TokenOpenFinance();
        token.setConsentId(2L);
        var consent = new ConsentimentoOpenFinance();
        consent.setPermissions(Set.of(ConsentimentoOpenFinance.TipoConsentimento.ACCOUNTS));
        consent.setContasAutorizadas(Set.of("acc-1"));
        when(coreApiClient.obterConsentimento(2L)).thenReturn(consent);

        List<Map<String, Object>> result = service.listarCartoesCreditoPorToken(token);

        assertNotNull(result);
        assertTrue(result.isEmpty());
        verify(coreApiClient, never()).getCreditCards(any());
    }

    @Test
    void shouldListPersonalIdentifications() {
        var token = new TokenOpenFinance();
        token.setConsentId(3L);
        token.setUserId(42L);
        var consent = new ConsentimentoOpenFinance();
        consent.setPermissions(Set.of(ConsentimentoOpenFinance.TipoConsentimento.IDENTIFICACAO));
        when(coreApiClient.obterConsentimento(3L)).thenReturn(consent);
        when(coreApiClient.getPersonalIdentifications(42L))
            .thenReturn(Map.of("nome", "João", "cpf", "12345678901"));

        List<Map<String, Object>> result = service.listarIdentificacoesPessoaisPorToken(token);

        assertNotNull(result);
        assertEquals(1, result.size());
        assertEquals("João", result.get(0).get("civilName"));
    }

    @Test
    void shouldReturnEmptyListWhenNoIdentificacaoPermission() {
        var token = new TokenOpenFinance();
        token.setConsentId(4L);
        var consent = new ConsentimentoOpenFinance();
        consent.setPermissions(Set.of(ConsentimentoOpenFinance.TipoConsentimento.ACCOUNTS));
        when(coreApiClient.obterConsentimento(4L)).thenReturn(consent);

        List<Map<String, Object>> result = service.listarIdentificacoesPessoaisPorToken(token);

        assertNotNull(result);
        assertTrue(result.isEmpty());
        verify(coreApiClient, never()).getPersonalIdentifications(any());
    }
}
```

Note: Adjust entity imports to match actual package names. The entities are in `com.aureus.platform.openfinance.entity.*`. If `setContasAutorizadas` or `setUserId` don't exist on `TokenOpenFinance`, use actual setter names from the entity.

- [ ] **Step 3: Run test to verify it fails**

```bash
mvn -pl aureus-openfinance -am test -Dtest=OpenFinanceDataServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: COMPILATION ERROR (methods don't exist yet)

- [ ] **Step 4: Add methods to OpenFinanceDataService**

Read the existing file first, then add after `listarTransacoesPorConta`:

```java
public List<Map<String, Object>> listarCartoesCreditoPorToken(TokenOpenFinance token) {
    ConsentimentoOpenFinance consent = coreApiClient.obterConsentimento(token.getConsentId());
    if (!consent.getPermissions().contains(ConsentimentoOpenFinance.TipoConsentimento.CREDIT_CARDS_ACCOUNTS)) {
        log.warn("Consentimento {} não possui permissão para cartões de crédito", token.getConsentId());
        return List.of();
    }
    List<String> contasAutorizadas = new ArrayList<>(consent.getContasAutorizadas());
    if (contasAutorizadas.isEmpty()) {
        return List.of();
    }
    return coreApiClient.getCreditCards(contasAutorizadas);
}

public List<Map<String, Object>> listarIdentificacoesPessoaisPorToken(TokenOpenFinance token) {
    ConsentimentoOpenFinance consentimento = coreApiClient.obterConsentimento(token.getConsentId());
    if (!consentimento.getPermissions().contains(ConsentimentoOpenFinance.TipoConsentimento.IDENTIFICACAO)) {
        log.warn("Consentimento {} não possui permissão para identificação pessoal", token.getConsentId());
        return List.of();
    }
    Map<String, Object> dados = coreApiClient.getPersonalIdentifications(token.getUserId());
    return List.of(Objects.requireNonNullElse(dados, Map.of()));
}
```

- [ ] **Step 5: Run test to verify it passes**

```bash
mvn test -pl aureus-openfinance -am -Dtest=OpenFinanceDataServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add backend/aureus-openfinance/src/main/java/com/aureus/platform/openfinance/service/OpenFinanceDataService.java backend/aureus-openfinance/src/test/java/com/aureus/platform/openfinance/service/OpenFinanceDataServiceTest.java
git commit -m "feat(openfinance): add credit cards and personal identifications data methods"
```

---

### Task G1-2: Update OpenFinanceController — replace TODO stubs

**Files:**
- Modify: `backend/aureus-openfinance/src/main/java/com/aureus/platform/openfinance/controller/OpenFinanceController.java`
- Test: `backend/aureus-openfinance/src/test/java/com/aureus/platform/openfinance/controller/OpenFinanceControllerTest.java`

**Interfaces:**
- Consumes: `openFinanceDataService.listarCartoesCreditoPorToken()`, `openFinanceDataService.listarIdentificacoesPessoaisPorToken()`
- Produces: fully functional `/credit-cards-accounts` and `/customers/personal/identifications` endpoints

- [ ] **Step 1: Write the failing test**

```java
package com.aureus.platform.openfinance.controller;

import com.aureus.platform.openfinance.service.OpenFinanceDataService;
import com.aureus.platform.openfinance.service.TokenOpenFinanceService;
import com.aureus.platform.openfinance.service.LogAcessoOpenFinanceService;
import com.aureus.platform.openfinance.entity.TokenOpenFinance;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(OpenFinanceController.class)
class OpenFinanceControllerTest {

    @Autowired private MockMvc mockMvc;
    @MockitoBean private TokenOpenFinanceService tokenService;
    @MockitoBean private LogAcessoOpenFinanceService logService;
    @MockitoBean private OpenFinanceDataService dataService;

    @Test
    void shouldReturnCreditCardsList() throws Exception {
        when(tokenService.validarToken(anyString())).thenReturn(new TokenOpenFinance());
        when(tokenService.verificarRateLimit(anyString())).thenReturn(true);
        when(dataService.listarCartoesCreditoPorToken(any()))
            .thenReturn(List.of(Map.of("brand", "VISA")));

        mockMvc.perform(get("/open-finance/credit-cards-accounts")
                .header("Authorization", "Bearer test-token"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.creditCards[0].brand").value("VISA"));
    }

    @Test
    void testReturnPersonalIdentifications() throws Exception {
        when(tokenService.validarToken(anyString())).thenReturn(new TokenOpenFinance());
        when(tokenService.verificarRateLimit(anyString())).thenReturn(true);
        when(dataService.listarIdentificacoesPessoaisPorToken(any()))
            .thenReturn(List.of(Map.of("civilName", "João")));

        mockMvc.perform(get("/open-finance/customers/personal/identifications")
                .header("Authorization", "Bearer test-token"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.personalIdentifications[0].civilName").value("João"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl aureus-openfinance -am -Dtest=OpenFinanceControllerTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: FAIL (controller still returns hardcoded `"[]"`)

- [ ] **Step 3: Replace TODO stubs in OpenFinanceController**

Replace lines 93-95:
```java
List<Map<String, Object>> creditCards = openFinanceDataService.listarCartoesCreditoPorToken(token);
Map<String, Object> response = Map.of(
    "data", Map.of("creditCards", creditCards),
    "links", Map.of("self", "/open-finance/credit-cards-accounts"),
    "meta", Map.of("totalRecords", creditCards.size(), "totalPages", creditCards.isEmpty() ? 0 : 1)
);
return ResponseEntity.ok(response);
```

Replace lines 115-117:
```java
List<Map<String, Object>> identifications = openFinanceDataService.listarIdentificacoesPessoaisPorToken(token);
Map<String, Object> response = Map.of(
    "data", Map.of("personalIdentifications", identifications),
    "links", Map.of("self", "/open-finance/customers/personal/identifications"),
    "meta", Map.of("totalRecords", identifications.size(), "totalPages", identifications.isEmpty() ? 0 : 1)
);
return ResponseEntity.ok(response);
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -pl aureus-openfinance -am -Dtest=OpenFinanceControllerTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/aureus-openfinance/src/main/java/com/aureus/platform/openfinance/controller/OpenFinanceController.java backend/aureus-openfinance/src/test/java/com/aureus/platform/openfinance/controller/OpenFinanceControllerTest.java
git commit -m "feat(openfinance): implement credit cards and personal identifications endpoints"
```

---

### Task G2-1: Create entities and repositories for password reset and refresh tokens

**Files:**
- Create: `backend/aureus-security/src/main/java/com/aureus/platform/security/entity/PasswordResetToken.java`
- Create: `backend/aureus-security/src/main/java/com/aureus/platform/security/entity/RefreshToken.java`
- Create: `backend/aureus-security/src/main/java/com/aureus/platform/security/repository/PasswordResetTokenRepository.java`
- Create: `backend/aureus-security/src/main/java/com/aureus/platform/security/repository/RefreshTokenRepository.java`
- Create: `backend/aureus-security/src/main/java/com/aureus/platform/security/dto/ForgotPasswordRequestDTO.java`
- Create: `backend/aureus-security/src/main/java/com/aureus/platform/security/dto/ResetPasswordRequestDTO.java`
- Create: `backend/aureus-security/src/main/java/com/aureus/platform/security/dto/RefreshTokenRequestDTO.java`
- Test: created manually via schema check (no service yet)

**Interfaces:**
- Produces: JPA entities + repositories consumed by AuthService

- [ ] **Step 1: Create PasswordResetToken entity**

```java
package com.aureus.platform.security.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.NotBlank;
import java.time.LocalDateTime;

@Entity
@Table(name = "password_reset_tokens", schema = "aureus")
public class PasswordResetToken {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "usuario_id", nullable = false)
    private Long usuarioId;

    @NotBlank
    @Column(nullable = false, unique = true)
    private String token;

    @Column(name = "expira_em", nullable = false)
    private LocalDateTime expiraEm;

    @Column(nullable = false)
    private boolean utilizado = false;

    @Column(name = "criado_em", nullable = false)
    private LocalDateTime criadoEm = LocalDateTime.now();

    public PasswordResetToken() {}

    public PasswordResetToken(Long usuarioId, String token, LocalDateTime expiraEm) {
        this.usuarioId = usuarioId;
        this.token = token;
        this.expiraEm = expiraEm;
    }

    public Long getId() { return id; }
    public Long getUsuarioId() { return usuarioId; }
    public String getToken() { return token; }
    public LocalDateTime getExpiraEm() { return expiraEm; }
    public boolean isUtilizado() { return utilizado; }
    public LocalDateTime getCriadoEm() { return criadoEm; }
    public void setUtilizado(boolean utilizado) { this.utilizado = utilizado; }
}
```

- [ ] **Step 2: Create RefreshToken entity**

```java
package com.aureus.platform.security.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.NotBlank;
import java.time.LocalDateTime;

@Entity
@Table(name = "refresh_tokens", schema = "aureus")
public class RefreshToken {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "usuario_id", nullable = false)
    private Long usuarioId;

    @NotBlank
    @Column(nullable = false, unique = true)
    private String token;

    @Column(name = "expira_em", nullable = false)
    private LocalDateTime expiraEm;

    @Column(nullable = false)
    private boolean revogado = false;

    @Column(name = "criado_em", nullable = false)
    private LocalDateTime criadoEm = LocalDateTime.now();

    public RefreshToken() {}

    public RefreshToken(Long usuarioId, String token, LocalDateTime expiraEm) {
        this.usuarioId = usuarioId;
        this.token = token;
        this.expiraEm = expiraEm;
    }

    public Long getId() { return id; }
    public Long getUsuarioId() { return usuarioId; }
    public String getToken() { return token; }
    public LocalDateTime getExpiraEm() { return expiraEm; }
    public boolean isRevogado() { return revogado; }
    public LocalDateTime getCriadoEm() { return criadoEm; }
    public void setRevogado(boolean revogado) { this.revogado = revogado; }
}
```

- [ ] **Step 3: Create PasswordResetTokenRepository**

```java
package com.aureus.platform.security.repository;

import com.aureus.platform.security.entity.PasswordResetToken;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface PasswordResetTokenRepository extends JpaRepository<PasswordResetToken, Long> {
    Optional<PasswordResetToken> findByToken(String token);
    void deleteByUsuarioId(Long usuarioId);
}
```

- [ ] **Step 4: Create RefreshTokenRepository**

```java
package com.aureus.platform.security.repository;

import com.aureus.platform.security.entity.RefreshToken;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {
    Optional<RefreshToken> findByToken(String token);
    Optional<RefreshToken> findByUsuarioIdAndRevogadoFalse(Long usuarioId);
}
```

- [ ] **Step 5: Create DTOs**

```java
// ForgotPasswordRequestDTO.java
package com.aureus.platform.security.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;

public class ForgotPasswordRequestDTO {
    @NotBlank @Email
    private String email;

    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}

// ResetPasswordRequestDTO.java
package com.aureus.platform.security.dto;

import jakarta.validation.constraints.NotBlank;

public class ResetPasswordRequestDTO {
    @NotBlank
    private String token;

    @NotBlank
    private String novaSenha;

    public String getToken() { return token; }
    public void setToken(String token) { this.token = token; }
    public String getNovaSenha() { return novaSenha; }
    public void setNovaSenha(String novaSenha) { this.novaSenha = novaSenha; }
}

// RefreshTokenRequestDTO.java
package com.aureus.platform.security.dto;

import jakarta.validation.constraints.NotBlank;

public class RefreshTokenRequestDTO {
    @NotBlank
    private String refreshToken;

    public String getRefreshToken() { return refreshToken; }
    public void setRefreshToken(String refreshToken) { this.refreshToken = refreshToken; }
}
```

- [ ] **Step 6: Compile to verify**

```bash
mvn compile -pl aureus-security -am
```

Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add backend/aureus-security/src/main/java/com/aureus/platform/security/entity/PasswordResetToken.java backend/aureus-security/src/main/java/com/aureus/platform/security/entity/RefreshToken.java backend/aureus-security/src/main/java/com/aureus/platform/security/repository/PasswordResetTokenRepository.java backend/aureus-security/src/main/java/com/aureus/platform/security/repository/RefreshTokenRepository.java backend/aureus-security/src/main/java/com/aureus/platform/security/dto/ForgotPasswordRequestDTO.java backend/aureus-security/src/main/java/com/aureus/platform/security/dto/ResetPasswordRequestDTO.java backend/aureus-security/src/main/java/com/aureus/platform/security/dto/RefreshTokenRequestDTO.java
git commit -m "feat(security): add password reset and refresh token entities, repositories, DTOs"
```

---

### Task G2-2: Add forgot-password, reset-password, refresh-token and timed lockout to AuthService

**Files:**
- Modify: `backend/aureus-security/src/main/java/com/aureus/platform/security/service/AuthService.java`
- Test: `backend/aureus-security/src/test/java/com/aureus/platform/security/service/AuthServiceTest.java`

**Interfaces:**
- Consumes: `PasswordResetTokenRepository`, `RefreshTokenRepository`, `PasswordEncoder`
- Produces: `forgotPassword()`, `resetPassword()`, `refreshToken()` methods, modified `login()`

- [ ] **Step 1: Write failing test**

```java
package com.aureus.platform.security.service;

import com.aureus.platform.security.entity.PasswordResetToken;
import com.aureus.platform.security.entity.RefreshToken;
import com.aureus.platform.security.repository.PasswordResetTokenRepository;
import com.aureus.platform.security.repository.RefreshTokenRepository;
import com.aureus.platform.security.repository.UsuarioRepository;
import com.aureus.platform.shared.dto.LoginRequestDTO;
import com.aureus.platform.shared.entity.Usuario;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.security.crypto.password.PasswordEncoder;

import java.time.LocalDateTime;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class AuthServiceTest {

    @Mock private UsuarioRepository usuarioRepository;
    @Mock private JwtService jwtService;
    @Mock private PasswordEncoder passwordEncoder;
    @Mock private PasswordResetTokenRepository passwordResetTokenRepository;
    @Mock private RefreshTokenRepository refreshTokenRepository;
    private AuthService authService;

    @BeforeEach
    void setUp() {
        authService = new AuthService(usuarioRepository, jwtService, passwordEncoder,
            passwordResetTokenRepository, refreshTokenRepository);
    }

    @Test
    void shouldCreateAndReturnPasswordResetToken() {
        var user = new Usuario();
        user.setId(1L);
        user.setEmail("test@test.com");
        user.setAtivo(true);
        when(usuarioRepository.findByEmailAndAtivoTrue("test@test.com")).thenReturn(Optional.of(user));

        authService.forgotPassword("test@test.com");

        verify(passwordResetTokenRepository).deleteByUsuarioId(1L);
        verify(passwordResetTokenRepository).save(argThat(token ->
            token.getUsuarioId().equals(1L) &&
            token.getToken() != null &&
            !token.isUtilizado()
        ));
    }

    @Test
    void shouldThrowWhenEmailNotFound() {
        when(usuarioRepository.findByEmailAndAtivoTrue("unknown@test.com")).thenReturn(Optional.empty());
        assertThrows(IllegalArgumentException.class,
            () -> authService.forgotPassword("unknown@test.com"));
    }

    @Test
    void shouldResetPasswordWithValidToken() {
        var existingToken = new PasswordResetToken(1L, "valid-token", LocalDateTime.now().plusMinutes(10));
        when(passwordResetTokenRepository.findByToken("valid-token")).thenReturn(Optional.of(existingToken));
        var user = new Usuario();
        user.setId(1L);
        user.setSenha("old-hash");
        when(usuarioRepository.findById(1L)).thenReturn(Optional.of(user));
        when(passwordEncoder.encode("new-pass")).thenReturn("new-hash");

        authService.resetPassword("valid-token", "new-pass");

        assertTrue(existingToken.isUtilizado());
        assertEquals("new-hash", user.getSenha());
        assertFalse(user.getContaBloqueada());
        verify(usuarioRepository).save(user);
        verify(passwordResetTokenRepository).save(existingToken);
    }

    @Test
    void shouldThrowWhenResetTokenExpired() {
        var expiredToken = new PasswordResetToken(1L, "expired-token", LocalDateTime.now().minusMinutes(1));
        when(passwordResetTokenRepository.findByToken("expired-token")).thenReturn(Optional.of(expiredToken));
        assertThrows(IllegalStateException.class,
            () -> authService.resetPassword("expired-token", "new-pass"));
    }

    @Test
    void shouldThrowWhenResetTokenAlreadyUsed() {
        var usedToken = new PasswordResetToken(1L, "used-token", LocalDateTime.now().plusMinutes(10));
        usedToken.setUtilizado(true);
        when(passwordResetTokenRepository.findByToken("used-token")).thenReturn(Optional.of(usedToken));
        assertThrows(IllegalStateException.class,
            () -> authService.resetPassword("used-token", "new-pass"));
    }

    @Test
    void shouldGenerateNewRefreshToken() {
        var oldRefresh = new RefreshToken(1L, "old-refresh", LocalDateTime.now().plusDays(7));
        when(refreshTokenRepository.findByToken("old-refresh")).thenReturn(Optional.of(oldRefresh));
        var user = new Usuario();
        user.setId(1L);
        user.setEmail("test@test.com");
        user.setNome("Test");
        when(usuarioRepository.findById(1L)).thenReturn(Optional.of(user));
        when(jwtService.generateToken(anyString(), anyLong(), anyString(), anySet(), anySet()))
            .thenReturn("new-jwt");

        var response = authService.refreshToken("old-refresh");

        assertTrue(oldRefresh.isRevogado());
        assertNotNull(response);
        assertEquals("new-jwt", response.getToken());
        verify(refreshTokenRepository).save(any(RefreshToken.class));
    }

    @Test
    void shouldAutoUnlockAfterTimeout() {
        var user = new Usuario();
        user.setId(1L);
        user.setEmail("test@test.com");
        user.setSenha("hash");
        user.setContaBloqueada(true);
        user.setUltimoLogin(LocalDateTime.now().minusMinutes(10));
        when(usuarioRepository.findByEmailAndAtivoTrue("test@test.com")).thenReturn(Optional.of(user));
        when(passwordEncoder.matches("correct", "hash")).thenReturn(true);
        when(jwtService.generateToken(any(), any(), any(), any(), any())).thenReturn("token");

        authService.login(new LoginRequestDTO("test@test.com", "correct"));

        assertFalse(user.getContaBloqueada());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl aureus-security -am -Dtest=AuthServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: FAIL (compilation — AuthService doesn't have new constructor or methods yet)

- [ ] **Step 3: Implement new methods in AuthService**

Read existing `AuthService.java` first. Then modify the constructor to accept the two new repositories, and add:

```java
// New fields
private final PasswordResetTokenRepository passwordResetTokenRepository;
private final RefreshTokenRepository refreshTokenRepository;

// Updated constructor
public AuthService(final UsuarioRepository usuarioRepository, final JwtService jwtService,
    final PasswordEncoder passwordEncoder,
    final PasswordResetTokenRepository passwordResetTokenRepository,
    final RefreshTokenRepository refreshTokenRepository) {
    this.usuarioRepository = usuarioRepository;
    this.jwtService = jwtService;
    this.passwordEncoder = passwordEncoder;
    this.passwordResetTokenRepository = passwordResetTokenRepository;
    this.refreshTokenRepository = refreshTokenRepository;
}
```

Add `login()` modification — replace the block check section:
```java
if (usuario.getContaBloqueada()) {
    if (usuario.getUltimoLogin() != null &&
        usuario.getUltimoLogin().plusMinutes(5).isAfter(LocalDateTime.now())) {
        throw new IllegalStateException("Conta temporariamente bloqueada. Tente novamente em 5 minutos.");
    }
    usuario.setContaBloqueada(false); // auto-unlock after timeout
}
```

Add new methods:
```java
@Transactional
public void forgotPassword(String email) {
    log.info("Solicitação de reset de senha para email: {}", email);
    Usuario usuario = usuarioRepository.findByEmailAndAtivoTrue(email)
        .orElseThrow(() -> new IllegalArgumentException("Email não encontrado"));
    passwordResetTokenRepository.deleteByUsuarioId(usuario.getId());
    String token = UUID.randomUUID().toString();
    PasswordResetToken resetToken = new PasswordResetToken(usuario.getId(), token, LocalDateTime.now().plusMinutes(15));
    passwordResetTokenRepository.save(resetToken);
    log.info("Token de reset gerado para usuário {}: {}", usuario.getId(), token);
    // Em prod: enviar email com o token
}

@Transactional
public void resetPassword(String token, String novaSenha) {
    log.info("Recebida solicitação de reset de senha com token");
    PasswordResetToken resetToken = passwordResetTokenRepository.findByToken(token)
        .orElseThrow(() -> new IllegalArgumentException("Token inválido"));
    if (resetToken.isUtilizado()) {
        throw new IllegalStateException("Token já utilizado");
    }
    if (resetToken.getExpiraEm().isBefore(LocalDateTime.now())) {
        throw new IllegalStateException("Token expirado");
    }
    Usuario usuario = usuarioRepository.findById(resetToken.getUsuarioId())
        .orElseThrow(() -> new IllegalArgumentException("Usuário não encontrado"));
    usuario.setSenha(passwordEncoder.encode(novaSenha));
    usuario.setDataExpiracaoSenha(LocalDateTime.now().plusDays(90));
    usuario.resetarTentativasLogin();
    usuario.setContaBloqueada(false);
    resetToken.setUtilizado(true);
    usuarioRepository.save(usuario);
    passwordResetTokenRepository.save(resetToken);
    log.info("Senha resetada com sucesso para usuário ID: {}", usuario.getId());
}

@Transactional
public LoginResponseDTO refreshToken(String refreshTokenValue) {
    log.info("Recebida solicitação de refresh token");
    RefreshToken oldToken = refreshTokenRepository.findByToken(refreshTokenValue)
        .orElseThrow(() -> new IllegalArgumentException("Refresh token inválido"));
    if (oldToken.isRevogado()) {
        throw new IllegalStateException("Refresh token já revogado");
    }
    if (oldToken.getExpiraEm().isBefore(LocalDateTime.now())) {
        throw new IllegalStateException("Refresh token expirado");
    }
    oldToken.setRevogado(true);
    refreshTokenRepository.save(oldToken);
    Usuario usuario = usuarioRepository.findById(oldToken.getUsuarioId())
        .orElseThrow(() -> new IllegalArgumentException("Usuário não encontrado"));
    Set<String> roles = usuario.getRoles().stream().map(role -> role.getNome()).collect(Collectors.toSet());
    Set<String> permissions = usuario.getRoles().stream()
        .flatMap(role -> role.getPermissions().stream())
        .map(permission -> permission.getNome())
        .collect(Collectors.toSet());
    String newJwt = jwtService.generateToken(usuario.getEmail(), usuario.getId(), usuario.getNome(), roles, permissions);
    String newRefresh = UUID.randomUUID().toString();
    RefreshToken newToken = new RefreshToken(usuario.getId(), newRefresh, LocalDateTime.now().plusDays(7));
    refreshTokenRepository.save(newToken);
    log.info("Refresh token gerado para usuário ID: {}", usuario.getId());
    return LoginResponseDTO.builder()
        .token(newJwt)
        .tipoToken("Bearer")
        .usuarioId(usuario.getId())
        .nome(usuario.getNome())
        .email(usuario.getEmail())
        .roles(roles)
        .permissions(permissions)
        .dataExpiracao(LocalDateTime.now().plusHours(24))
        .ultimoLogin(usuario.getUltimoLogin())
        .build();
}
```

Add imports:
```java
import com.aureus.platform.security.entity.PasswordResetToken;
import com.aureus.platform.security.entity.RefreshToken;
import com.aureus.platform.security.repository.PasswordResetTokenRepository;
import com.aureus.platform.security.repository.RefreshTokenRepository;
import java.util.UUID;
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -pl aureus-security -am -Dtest=AuthServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/aureus-security/src/main/java/com/aureus/platform/security/service/AuthService.java backend/aureus-security/src/test/java/com/aureus/platform/security/service/AuthServiceTest.java
git commit -m "feat(security): add forgot-password, reset-password, refresh-token, timed lockout"
```

---

### Task G2-3: Add new endpoints to AuthController

**Files:**
- Modify: `backend/aureus-security/src/main/java/com/aureus/platform/security/controller/AuthController.java`
- Test: `backend/aureus-security/src/test/java/com/aureus/platform/security/controller/AuthControllerTest.java`

**Interfaces:**
- Consumes: `authService.forgotPassword()`, `authService.resetPassword()`, `authService.refreshToken()`
- Produces: 3 new REST endpoints validated

- [ ] **Step 1: Write the failing test**

```java
package com.aureus.platform.security.controller;

import com.aureus.platform.security.service.AuthService;
import com.aureus.platform.shared.dto.LoginResponseDTO;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(AuthController.class)
class AuthControllerTest {

    @Autowired private MockMvc mockMvc;
    @Autowired private ObjectMapper objectMapper;
    @MockitoBean private AuthService authService;

    @Test
    void forgotPasswordShouldReturnOk() throws Exception {
        mockMvc.perform(post("/auth/forgot-password")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"email\":\"test@test.com\"}"))
            .andExpect(status().isOk());
        verify(authService).forgotPassword("test@test.com");
    }

    @Test
    void resetPasswordShouldReturnNoContent() throws Exception {
        mockMvc.perform(post("/auth/reset-password")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"token\":\"abc\",\"novaSenha\":\"New@1234\"}"))
            .andExpect(status().isNoContent());
        verify(authService).resetPassword("abc", "New@1234");
    }

    @Test
    void refreshTokenShouldReturnLoginResponse() throws Exception {
        when(authService.refreshToken("valid-refresh"))
            .thenReturn(LoginResponseDTO.builder().token("new-jwt").tipoToken("Bearer").build());

        mockMvc.perform(post("/auth/refresh")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"refreshToken\":\"valid-refresh\"}"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.token").value("new-jwt"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl aureus-security -am -Dtest=AuthControllerTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: FAIL

- [ ] **Step 3: Add endpoints to AuthController**

Read existing file, then add after `toggleBlockUser`:

```java
@PostMapping("/forgot-password")
@Operation(summary = "Solicitar reset de senha", description = "Envia token de reset de senha por email")
public ResponseEntity<Void> forgotPassword(@Valid @RequestBody ForgotPasswordRequestDTO request) {
    log.info("Recebida solicitação de forgot-password para email: {}", request.getEmail());
    authService.forgotPassword(request.getEmail());
    return ResponseEntity.ok().build();
}

@PostMapping("/reset-password")
@Operation(summary = "Resetar senha", description = "Reseta a senha usando token de recuperação")
public ResponseEntity<Void> resetPassword(@Valid @RequestBody ResetPasswordRequestDTO request) {
    log.info("Recebida solicitação de reset-password");
    authService.resetPassword(request.getToken(), request.getNovaSenha());
    return ResponseEntity.noContent().build();
}

@PostMapping("/refresh")
@Operation(summary = "Refresh token", description = "Gera novo JWT e refresh token a partir de refresh token válido")
public ResponseEntity<LoginResponseDTO> refresh(@Valid @RequestBody RefreshTokenRequestDTO request) {
    log.info("Recebida solicitação de refresh token");
    LoginResponseDTO response = authService.refreshToken(request.getRefreshToken());
    return ResponseEntity.ok(response);
}
```

Add imports:
```java
import com.aureus.platform.security.dto.ForgotPasswordRequestDTO;
import com.aureus.platform.security.dto.ResetPasswordRequestDTO;
import com.aureus.platform.security.dto.RefreshTokenRequestDTO;
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn test -pl aureus-security -am -Dtest=AuthControllerTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/aureus-security/src/main/java/com/aureus/platform/security/controller/AuthController.java backend/aureus-security/src/test/java/com/aureus/platform/security/controller/AuthControllerTest.java
git commit -m "feat(security): add forgot-password, reset-password, refresh-token endpoints"
```

---

### Task G3-1: Add Schema Registry and Kafka Connect to staging compose

**Files:**
- Modify: `aureus-data-platform/kafka/docker-compose.staging.yml`
- Modify: `aureus-data-platform/kafka/.env.example`

- [ ] **Step 1: Read current docker-compose.staging.yml**

Read the file to identify insertion points.

- [ ] **Step 2: Add schema-registry and kafka-connect services**

Insert after kafka-3 block (before kafka-ui):

```yaml
  schema-registry:
    image: confluentinc/cp-schema-registry:7.4.0
    hostname: schema-registry
    container_name: aureus-schema-registry-staging
    depends_on:
      - kafka-1
      - kafka-2
      - kafka-3
    ports:
      - "${STAGING_SCHEMA_REGISTRY_PORT:-8081}:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: 'kafka-1:29092,kafka-2:29093,kafka-3:29094'
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
    networks:
      - aureus-data-network-staging

  kafka-connect:
    image: confluentinc/cp-kafka-connect:7.4.0
    hostname: kafka-connect
    container_name: aureus-kafka-connect-staging
    depends_on:
      - kafka-1
      - kafka-2
      - kafka-3
      - schema-registry
    ports:
      - "${STAGING_KAFKA_CONNECT_PORT:-8083}:8083"
    environment:
      CONNECT_BOOTSTRAP_SERVERS: 'kafka-1:29092,kafka-2:29093,kafka-3:29094'
      CONNECT_REST_PORT: 8083
      CONNECT_GROUP_ID: "connect-cluster-staging"
      CONNECT_CONFIG_STORAGE_TOPIC: "connect-configs-staging"
      CONNECT_OFFSET_STORAGE_TOPIC: "connect-offsets-staging"
      CONNECT_STATUS_STORAGE_TOPIC: "connect-status-staging"
      CONNECT_KEY_CONVERTER: "io.confluent.connect.avro.AvroConverter"
      CONNECT_VALUE_CONVERTER: "io.confluent.connect.avro.AvroConverter"
      CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL: 'http://schema-registry:8081'
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: 'http://schema-registry:8081'
      CONNECT_LOG4J_LOGGERS: "org.apache.kafka.connect=INFO"
      CONNECT_PLUGIN_PATH: "/usr/share/java,/etc/kafka-connect/jars"
    networks:
      - aureus-data-network-staging
```

- [ ] **Step 3: Update .env.example**

Read current `.env.example`, then add to it:

```env
# Staging-specific ports (dev uses KAFKA_UI_PORT above)
STAGING_KAFKA_UI_PORT=8085
STAGING_SCHEMA_REGISTRY_PORT=8081
STAGING_KAFKA_CONNECT_PORT=8083
STAGING_KAFKA_REPLICATION_FACTOR=3
```

- [ ] **Step 4: Validate YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('aureus-data-platform/kafka/docker-compose.staging.yml'))" && echo "Valid YAML"
```

Expected: Valid YAML

- [ ] **Step 5: Commit**

```bash
git add aureus-data-platform/kafka/docker-compose.staging.yml aureus-data-platform/kafka/.env.example
git commit -m "feat(kafka): add schema registry and kafka connect to staging profile"
```

---

### Task G4-1: Create Extrato page with tests

**Files:**
- Create: `frontend/aureus-web/src/pages/Extrato.js`
- Create: `frontend/aureus-web/src/pages/Extrato.test.js`

**Interfaces:**
- Consumes: `user` prop, `apiService.getContas()` (or mock data)
- Produces: standalone Extrato page component + passing tests

- [ ] **Step 1: Write the failing test**

```jsx
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import Extrato from './Extrato';

jest.mock('../../services/apiService', () => ({
  apiService: {
    getContas: jest.fn().mockResolvedValue([
      { id: 1, numero: '12345-6', tipo: 'Corrente', saldo: 15000 }
    ])
  }
}));

describe('Extrato', () => {
  test('renders page title', () => {
    render(<Extrato user={{ nome: 'João' }} />);
    expect(screen.getByText(/Extrato/i)).toBeInTheDocument();
  });

  test('renders account selector', async () => {
    render(<Extrato user={{ nome: 'João' }} />);
    await waitFor(() => {
      expect(screen.getByText('12345-6')).toBeInTheDocument();
    });
  });

  test('renders date filter inputs', () => {
    render(<Extrato user={{ nome: 'João' }} />);
    expect(screen.getByLabelText(/Data Inicial/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/Data Final/i)).toBeInTheDocument();
  });

  test('renders transactions table', async () => {
    render(<Extrato user={{ nome: 'João' }} />);
    await waitFor(() => {
      expect(screen.getByText('PIX recebido')).toBeInTheDocument();
    });
  });

  test('renders download PDF button', () => {
    render(<Extrato user={{ nome: 'João' }} />);
    expect(screen.getByText('Download PDF')).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd frontend && npm test Extrato.test.js 2>&1 | head -50
```

Expected: FAIL (module not found)

- [ ] **Step 3: Create Extrato.js**

```jsx
import React, { useState, useEffect } from 'react';
import {
  Box, Card, CardContent, Typography, Grid, Button,
  Table, TableBody, TableCell, TableContainer, TableHead, TableRow,
  Paper, TextField, FormControl, InputLabel, Select, MenuItem, Alert,
  Dialog, DialogTitle, DialogContent, DialogActions, Skeleton
} from '@mui/material';
import { AccountBalance, Download, Receipt } from '@mui/icons-material';
import { apiService } from '../services/apiService';
import numeral from 'numeral';
import { format } from 'date-fns';
import { ptBR } from 'date-fns/locale';

function Extrato({ user }) {
  const [loading, setLoading] = useState(true);
  const [contas, setContas] = useState([]);
  const [contaSelecionada, setContaSelecionada] = useState('');
  const [transacoes, setTransacoes] = useState([]);
  const [dataInicial, setDataInicial] = useState('');
  const [dataFinal, setDataFinal] = useState('');

  useEffect(() => {
    carregarContas();
  }, []);

  const carregarContas = async () => {
    try {
      setLoading(true);
      const data = await apiService.getContas(user?.contaId || 1);
      setContas(Array.isArray(data) ? data : []);
    } catch {
      setContas([
        { id: 1, numero: '12345-6', tipo: 'Corrente', saldo: 15750.50 },
        { id: 2, numero: '67890-1', tipo: 'Poupança', saldo: 25000 },
      ]);
    } finally {
      setLoading(false);
    }
  };

  const carregarExtrato = () => {
    setTransacoes([
      { id: 1, data: '2024-01-15', descricao: 'PIX recebido - João Silva', valor: 500, saldo: 15750.50 },
      { id: 2, data: '2024-01-14', descricao: 'PIX enviado - Maria Santos', valor: -250, saldo: 15250.50 },
      { id: 3, data: '2024-01-13', descricao: 'PIX recebido - Freelance', valor: 1200, saldo: 15500.50 },
    ]);
  };

  const formatCurrency = (value) =>
    new Intl.NumberFormat('pt-BR', { style: 'currency', currency: 'BRL' }).format(value);

  const formatDate = (dateStr) =>
    dateStr ? format(new Date(dateStr), 'dd/MM/yyyy', { locale: ptBR }) : '';

  if (loading) {
    return (
      <Box sx={{ p: 3 }}>
        <Skeleton variant="text" width="60%" height={40} />
        <Skeleton variant="rectangular" height={300} sx={{ mt: 2 }} />
      </Box>
    );
  }

  return (
    <Box sx={{ p: 3 }}>
      <Box display="flex" justifyContent="space-between" alignItems="center" mb={3}>
        <Typography variant="h4">Extrato</Typography>
        <Button variant="contained" startIcon={<Download />}>Download PDF</Button>
      </Box>

      <Card sx={{ mb: 3 }}>
        <CardContent>
          <Grid container spacing={2} alignItems="center">
            <Grid item xs={12} md={4}>
              <FormControl fullWidth>
                <InputLabel>Conta</InputLabel>
                <Select value={contaSelecionada} onChange={(e) => setContaSelecionada(e.target.value)}>
                  {contas.map((c) => (
                    <MenuItem key={c.id} value={c.id}>{c.tipo} - {c.numero}</MenuItem>
                  ))}
                </Select>
              </FormControl>
            </Grid>
            <Grid item xs={12} md={3}>
              <TextField label="Data Inicial" type="date" fullWidth
                InputLabelProps={{ shrink: true }}
                value={dataInicial} onChange={(e) => setDataInicial(e.target.value)} />
            </Grid>
            <Grid item xs={12} md={3}>
              <TextField label="Data Final" type="date" fullWidth
                InputLabelProps={{ shrink: true }}
                value={dataFinal} onChange={(e) => setDataFinal(e.target.value)} />
            </Grid>
            <Grid item xs={12} md={2}>
              <Button variant="contained" fullWidth onClick={carregarExtrato}>
                Filtrar
              </Button>
            </Grid>
          </Grid>
        </CardContent>
      </Card>

      <Card>
        <CardContent>
          <Typography variant="h6" gutterBottom>Lançamentos</Typography>
          <TableContainer component={Paper}>
            <Table>
              <TableHead>
                <TableRow>
                  <TableCell>Data</TableCell>
                  <TableCell>Descrição</TableCell>
                  <TableCell>Valor</TableCell>
                  <TableCell>Saldo</TableCell>
                </TableRow>
              </TableHead>
              <TableBody>
                {transacoes.length === 0 ? (
                  <TableRow><TableCell colSpan={4} align="center">Nenhuma transação encontrada</TableCell></TableRow>
                ) : transacoes.map((t) => (
                  <TableRow key={t.id}>
                    <TableCell>{formatDate(t.data)}</TableCell>
                    <TableCell>{t.descricao}</TableCell>
                    <TableCell sx={{ color: t.valor > 0 ? 'success.main' : 'error.main' }}>
                      {t.valor > 0 ? '+' : ''}{formatCurrency(t.valor)}
                    </TableCell>
                    <TableCell>{formatCurrency(t.saldo)}</TableCell>
                  </TableRow>
                ))}
              </TableBody>
            </Table>
          </TableContainer>
        </CardContent>
      </Card>
    </Box>
  );
}

export default Extrato;
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd frontend && npx jest Extrato.test.js --no-coverage 2>&1
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add frontend/aureus-web/src/pages/Extrato.js frontend/aureus-web/src/pages/Extrato.test.js
git commit -m "feat(web): add Extrato page with filters and transactions table"
```

---

### Task G4-2: Create Transferencia page with tests

**Files:**
- Create: `frontend/aureus-web/src/pages/Transferencia.js`
- Create: `frontend/aureus-web/src/pages/Transferencia.test.js`

**Interfaces:**
- Consumes: `user` prop
- Produces: standalone Transferencia page component with passing tests

- [ ] **Step 1: Write the failing test**

```jsx
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import Transferencia from './Transferencia';

describe('Transferencia', () => {
  test('renders page title', () => {
    render(<Transferencia user={{ nome: 'João' }} />);
    expect(screen.getByText(/Transferência/i)).toBeInTheDocument();
  });

  test('renders transfer type selector', () => {
    render(<Transferencia user={{ nome: 'João' }} />);
    expect(screen.getByText('TED')).toBeInTheDocument();
    expect(screen.getByText('DOC')).toBeInTheDocument();
    expect(screen.getByText('PIX')).toBeInTheDocument();
  });

  test('renders destination account fields', () => {
    render(<Transferencia user={{ nome: 'João' }} />);
    expect(screen.getByLabelText(/Banco/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/Agência/i)).toBeInTheDocument();
  });

  test('renders confirmation dialog on submit', () => {
    render(<Transferencia user={{ nome: 'João' }} />);
    const submitBtn = screen.getByText('Transferir');
    fireEvent.click(submitBtn);
    expect(screen.getByText(/Confirmar Transferência/i)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd frontend && npm test Transferencia.test.js 2>&1 | head -50
```

Expected: FAIL (module not found)

- [ ] **Step 3: Create Transferencia.js**

```jsx
import React, { useState } from 'react';
import {
  Box, Card, CardContent, Typography, Grid, Button, TextField,
  FormControl, InputLabel, Select, MenuItem, Dialog, DialogTitle,
  DialogContent, DialogActions, Alert, Stepper, Step, StepLabel
} from '@mui/material';
import { SwapHoriz } from '@mui/icons-material';

function Transferencia({ user }) {
  const [tipo, setTipo] = useState('PIX');
  const [banco, setBanco] = useState('');
  const [agencia, setAgencia] = useState('');
  const [conta, setConta] = useState('');
  const [digito, setDigito] = useState('');
  const [chavePix, setChavePix] = useState('');
  const [valor, setValor] = useState('');
  const [dataAgendamento, setDataAgendamento] = useState('');
  const [confirmOpen, setConfirmOpen] = useState(false);
  const [success, setSuccess] = useState(false);

  const handleSubmit = (e) => {
    e.preventDefault();
    setConfirmOpen(true);
  };

  const handleConfirm = () => {
    setConfirmOpen(false);
    setSuccess(true);
    setTimeout(() => setSuccess(false), 5000);
  };

  return (
    <Box sx={{ p: 3 }}>
      <Box display="flex" alignItems="center" mb={3} gap={1}>
        <SwapHoriz />
        <Typography variant="h4">Transferência</Typography>
      </Box>

      {success && <Alert severity="success" sx={{ mb: 2 }}>Transferência realizada com sucesso!</Alert>}

      <Card>
        <CardContent>
          <Stepper activeStep={0} sx={{ mb: 3 }}>
            <Step><StepLabel>Dados da Transferência</StepLabel></Step>
            <Step><StepLabel>Confirmação</StepLabel></Step>
            <Step><StepLabel>Concluído</StepLabel></Step>
          </Stepper>

          <Grid container spacing={2}>
            <Grid item xs={12}>
              <FormControl fullWidth>
                <InputLabel>Tipo</InputLabel>
                <Select value={tipo} onChange={(e) => setTipo(e.target.value)}>
                  <MenuItem value="TED">TED</MenuItem>
                  <MenuItem value="DOC">DOC</MenuItem>
                  <MenuItem value="PIX">PIX</MenuItem>
                </Select>
              </FormControl>
            </Grid>

            {tipo === 'PIX' ? (
              <Grid item xs={12}>
                <TextField fullWidth label="Chave PIX (CPF/CNPJ/Email/Telefone)"
                  value={chavePix} onChange={(e) => setChavePix(e.target.value)} />
              </Grid>
            ) : (
              <>
                <Grid item xs={12} md={4}>
                  <TextField fullWidth label="Banco" value={banco}
                    onChange={(e) => setBanco(e.target.value)} />
                </Grid>
                <Grid item xs={12} md={3}>
                  <TextField fullWidth label="Agência" value={agencia}
                    onChange={(e) => setAgencia(e.target.value)} />
                </Grid>
                <Grid item xs={12} md={3}>
                  <TextField fullWidth label="Conta" value={conta}
                    onChange={(e) => setConta(e.target.value)} />
                </Grid>
                <Grid item xs={12} md={2}>
                  <TextField fullWidth label="Dígito" value={digito}
                    onChange={(e) => setDigito(e.target.value)} />
                </Grid>
              </>
            )}

            <Grid item xs={12} md={6}>
              <TextField fullWidth label="Valor" type="number" value={valor}
                onChange={(e) => setValor(e.target.value)} />
            </Grid>
            <Grid item xs={12} md={6}>
              <TextField fullWidth label="Data de Agendamento" type="date"
                InputLabelProps={{ shrink: true }} value={dataAgendamento}
                onChange={(e) => setDataAgendamento(e.target.value)} />
            </Grid>

            <Grid item xs={12}>
              <Button variant="contained" fullWidth size="large"
                onClick={handleSubmit} disabled={!valor}>
                Transferir
              </Button>
            </Grid>
          </Grid>
        </CardContent>
      </Card>

      <Dialog open={confirmOpen} onClose={() => setConfirmOpen(false)} maxWidth="sm" fullWidth>
        <DialogTitle>Confirmar Transferência</DialogTitle>
        <DialogContent>
          <Typography>Tipo: {tipo}</Typography>
          <Typography>Valor: R$ {valor}</Typography>
          {tipo === 'PIX' ? (
            <Typography>Chave PIX: {chavePix}</Typography>
          ) : (
            <>
              <Typography>Banco: {banco}</Typography>
              <Typography>Agência: {agencia}</Typography>
              <Typography>Conta: {conta}-{digito}</Typography>
            </>
          )}
          {dataAgendamento && <Typography>Agendado para: {dataAgendamento}</Typography>}
        </DialogContent>
        <DialogActions>
          <Button onClick={() => setConfirmOpen(false)}>Cancelar</Button>
          <Button variant="contained" onClick={handleConfirm}>Confirmar</Button>
        </DialogActions>
      </Dialog>
    </Box>
  );
}

export default Transferencia;
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd frontend && npx jest Transferencia.test.js --no-coverage 2>&1
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add frontend/aureus-web/src/pages/Transferencia.js frontend/aureus-web/src/pages/Transferencia.test.js
git commit -m "feat(web): add Transferencia page with TED/DOC/PIX forms"
```

---

### Task G4-3: Create Pagamento page with tests

**Files:**
- Create: `frontend/aureus-web/src/pages/Pagamento.js`
- Create: `frontend/aureus-web/src/pages/Pagamento.test.js`

- [ ] **Step 1: Write failing test**

```jsx
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import Pagamento from './Pagamento';

describe('Pagamento', () => {
  test('renders page title', () => {
    render(<Pagamento user={{ nome: 'João' }} />);
    expect(screen.getByText(/Pagamento/i)).toBeInTheDocument();
  });

  test('renders barcode input', () => {
    render(<Pagamento user={{ nome: 'João' }} />);
    expect(screen.getByLabelText(/Código de Barras/i)).toBeInTheDocument();
  });

  test('renders payment method after barcode entry', () => {
    render(<Pagamento user={{ nome: 'João' }} />);
    const input = screen.getByLabelText(/Código de Barras/i);
    fireEvent.change(input, { target: { value: '12345678901234567890123456789012345678901234' } });
    expect(screen.getByText(/Vencimento/i)).toBeInTheDocument();
  });

  test('renders confirmation dialog on pay', () => {
    render(<Pagamento user={{ nome: 'João' }} />);
    const payBtn = screen.getByText('Pagar');
    fireEvent.click(payBtn);
    expect(screen.getByText(/Confirmar Pagamento/i)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd frontend && npm test Pagamento.test.js 2>&1 | head -50
```

Expected: FAIL

- [ ] **Step 3: Create Pagamento.js**

```jsx
import React, { useState } from 'react';
import {
  Box, Card, CardContent, Typography, Grid, Button, TextField,
  Dialog, DialogTitle, DialogContent, DialogActions, Alert, Chip
} from '@mui/material';
import { Payment } from '@mui/icons-material';
import numeral from 'numeral';
import { format } from 'date-fns';
import { ptBR } from 'date-fns/locale';

function Pagamento({ user }) {
  const [codigoBarras, setCodigoBarras] = useState('');
  const [boletoInfo, setBoletoInfo] = useState(null);
  const [confirmOpen, setConfirmOpen] = useState(false);
  const [success, setSuccess] = useState(false);

  const handleConsultar = () => {
    setBoletoInfo({
      codigoBarras,
      vencimento: '2024-02-15',
      valorTotal: 1250.00,
      multa: 12.50,
      juros: 5.00,
      status: 'PENDENTE',
      beneficiario: 'Concessionária ABC Ltda',
    });
  };

  const handlePagar = () => setConfirmOpen(true);

  const handleConfirm = () => {
    setConfirmOpen(false);
    setSuccess(true);
    setTimeout(() => setSuccess(false), 5000);
  };

  return (
    <Box sx={{ p: 3 }}>
      <Box display="flex" alignItems="center" mb={3} gap={1}>
        <Payment />
        <Typography variant="h4">Pagamento</Typography>
      </Box>

      {success && <Alert severity="success" sx={{ mb: 2 }}>Pagamento realizado com sucesso!</Alert>}

      <Card sx={{ mb: 3 }}>
        <CardContent>
          <Grid container spacing={2} alignItems="center">
            <Grid item xs={12} md={10}>
              <TextField fullWidth label="Código de Barras" value={codigoBarras}
                onChange={(e) => setCodigoBarras(e.target.value)}
                placeholder="Digite o código de barras do boleto" />
            </Grid>
            <Grid item xs={12} md={2}>
              <Button variant="contained" fullWidth onClick={handleConsultar}
                disabled={!codigoBarras}>
                Consultar
              </Button>
            </Grid>
          </Grid>
        </CardContent>
      </Card>

      {boletoInfo && (
        <Card>
          <CardContent>
            <Typography variant="h6" gutterBottom>Dados do Boleto</Typography>
            <Grid container spacing={2}>
              <Grid item xs={12} md={6}>
                <Typography variant="body2" color="text.secondary">Beneficiário</Typography>
                <Typography variant="body1">{boletoInfo.descricao}</Typography>
              </Grid>
              <Grid item xs={12} md={3}>
                <Typography variant="body2" color="text.secondary">Vencimento</Typography>
                <Typography variant="body1">
                  {format(new Date(boletoInfo.vencimento), 'dd/MM/yyyy', { locale: ptBR })}
                </Typography>
              </Grid>
              <Grid item xs={12} md={3}>
                <Typography variant="body2" color="text.secondary">Status</Typography>
                <Chip label={boletoInfo.status} color="warning" size="small" />
              </Grid>
              <Grid item xs={12} md={4}>
                <Typography variant="body2" color="text.secondary">Valor Total</Typography>
                <Typography variant="h5">{numeral(boletoInfo.valor).format('$0,0.00')}</Typography>
              </Grid>
              <Grid item xs={12} md={4}>
                <Typography variant="body2" color="text.secondary">Multa</Typography>
                <Typography variant="body1">{numeral(boletoInfo.multa).format('$0,0.00')}</Typography>
              </Grid>
              <Grid item xs={12} md={4}>
                <Typography variant="body2" color="text.secondary">Juros</Typography>
                <Typography variant="body1">{numeral(boletoInfo.juros).format('$0,0.00')}</Typography>
              </Grid>
              <Grid item xs={12}>
                <Button variant="contained" fullWidth size="large" onClick={handlePagar}>
                  Pagar
                </Button>
              </Grid>
            </Grid>
          </CardContent>
        </Card>
      )}

      <Dialog open={confirmOpen} onClose={() => setConfirmOpen(false)} maxWidth="sm" fullWidth>
        <DialogTitle>Confirmar Pagamento</DialogTitle>
        <DialogContent>
          <Typography>Beneficiário: {boletoInfo?.descricao}</Typography>
          <Typography>Valor: {numeral(boletoInfo?.valor).format('$0,0.00')}</Typography>
          <Typography>Vencimento: {boletoInfo?.vencimento}</Typography>
        </DialogContent>
        <DialogActions>
          <Button onClick={() => setConfirmOpen(false)}>Cancelar</Button>
          <Button variant="contained" onClick={handleConfirm}>Confirmar Pagamento</Button>
        </DialogActions>
      </Dialog>
    </Box>
  );
}

export default Pagamento;
```

- [ ] **Step 4: Run test**

```bash
cd frontend && npx jest Pagamento.test.js --no-coverage 2>&1
```

Expected: PASS (or show failures to debug)

- [ ] **Step 5: Commit**

```bash
git add frontend/aureus-web/src/pages/Pagamento.js frontend/aureus-web/src/pages/Pagamento.test.js
git commit -m "feat(web): add Pagamento page with barcode lookup and payment"
```

---

### Task G4-4: Create Recarga page with tests, wire routes and sidebar

**Files:**
- Create: `frontend/aureus-web/src/pages/Recarga.js`
- Create: `frontend/aureus-web/src/pages/Recarga.test.js`
- Modify: `frontend/aureus-web/src/App.js`
- Modify: `frontend/aureus-web/src/components/Sidebar.js`

**Interfaces:**
- Produces: Recarga page, 4 new routes in App.js, 4 new sidebar links

- [ ] **Step 1: Write failing test for Recarga**

```jsx
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import Recarga from './Recarga';

describe('Recarga', () => {
  test('renders page title', () => {
    render(<Recarga user={{ nome: 'João' }} />);
    expect(screen.getByText(/Recarga/i)).toBeInTheDocument();
  });

  test('renders operator selector', () => {
    render(<Recarga user={{ nome: 'João' }} />);
    expect(screen.getByText('Vivo')).toBeInTheDocument();
    expect(screen.getByText('Claro')).toBeInTheDocument();
  });

  test('renders phone input', () => {
    render(<Recarga user={{ nome: 'João' }} />);
    expect(screen.getByLabelText(/Telefone/i)).toBeInTheDocument();
  });

  test('renders confirmation dialog on confirm', () => {
    render(<Recarga user={{ nome: 'João' }} />);
    const confirmBtn = screen.getByText('Confirmar');
    fireEvent.click(confirmBtn);
    expect(screen.getByText(/Confirmar Recarga/i)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Create Recarga.js**

```jsx
import React, { useState } from 'react';
import {
  Box, Card, CardContent, Typography, Grid, Button, TextField,
  FormControl, InputLabel, Select, MenuItem, Dialog, DialogTitle,
  DialogContent, DialogActions, Alert, ToggleButtonGroup, ToggleButton
} from '@mui/material';
import { PhoneAndroid } from '@mui/icons-material';
import numeral from 'numeral';
import { InputMask } from '@react-input/mask';

function Recarga({ user }) {
  const [operadora, setOperadora] = useState('');
  const [telefone, setTelefone] = useState('');
  const [valor, setValor] = useState('');
  const [outroValor, setOutroValor] = useState('');
  const [confirmOpen, setConfirmOpen] = useState(false);
  const [success, setSuccess] = useState(false);

  const valoresPadrao = ['10,00', '15,00', '20,00', '50,00', '100,00'];

  const handleValorClick = (v) => {
    setValor(v);
    setOutroValor('');
  };

  const handleConfirmar = () => setConfirmOpen(true);

  const handleConfirm = () => {
    setConfirmOpen(false);
    setSuccess(true);
    setTimeout(() => setSuccess(false), 5000);
  };

  const valorFinal = outroValor || valor;

  return (
    <Box sx={{ p: 3 }}>
      <Box display="flex" alignItems="center" mb={3} gap={1}>
        <PhoneAndroid />
        <Typography variant="h4">Recarga</Typography>
      </Box>

      {success && <Alert severity="success" sx={{ mb: 2 }}>Recarga realizada com sucesso!</Alert>}

      <Card>
        <CardContent>
          <Grid container spacing={3}>
            <Grid item xs={12} md={4}>
              <FormControl fullWidth>
                <InputLabel>Operadora</InputLabel>
                <Select value={operadora} onChange={(e) => setOperadora(e.target.value)}>
                  <MenuItem value="Vivo">Vivo</MenuItem>
                  <MenuItem value="Claro">Claro</MenuItem>
                  <MenuItem value="TIM">TIM</MenuItem>
                  <MenuItem value="Oi">Oi</MenuItem>
                </Select>
              </FormControl>
            </Grid>
            <Grid item xs={12} md={4}>
              <TextField fullWidth label="Telefone" value={telefone}
                onChange={(e) => setTelefone(e.target.value)}
                placeholder="(11) 99999-9999" />
            </Grid>
            <Grid item xs={12} md={4}>
              <FormControl fullWidth>
                <InputLabel>Valor</InputLabel>
                <Select value={outroValor ? 'outro' : valor} onChange={(e) => {
                  if (e.target.value === 'outro') {
                    setOutroValor('');
                    setValor('');
                  } else {
                    setValor(e.target.value);
                    setOutroValor('');
                  }
                }}>
                  {valoresPadrao.map((v) => (
                    <MenuItem key={v} value={v}>R$ {v}</MenuItem>
                  ))}
                  <MenuItem value="outro">Outro valor</MenuItem>
                </Select>
              </FormControl>
            </Grid>
            {outroValor !== undefined && !outroValor ? (
              <Grid item xs={12} md={4}>
                <TextField fullWidth label="Outro Valor" type="number"
                  value={outroValor} onChange={(e) => setOutroValor(e.target.value)} />
              </Grid>
            ) : null}
            <Grid item xs={12}>
              <Button variant="contained" fullWidth size="large"
                onClick={handleConfirmar} disabled={!operadora || !telefone || !valorFinal}>
                Confirmar
              </Button>
            </Grid>
          </Grid>
        </CardContent>
      </Card>

      <Dialog open={confirmOpen} onClose={() => setConfirmOpen(false)} maxWidth="sm" fullWidth>
        <DialogTitle>Confirmar Recarga</DialogTitle>
        <DialogContent>
          <Typography>Operadora: {operadora}</Typography>
          <Typography>Telefone: {telefone}</Typography>
          <Typography>Valor: R$ {valorFinal}</Typography>
        </DialogContent>
        <DialogActions>
          <Button onClick={() => setConfirmOpen(false)}>Cancelar</Button>
          <Button variant="contained" onClick={handleConfirm}>Confirmar Recarga</Button>
        </DialogActions>
      </Dialog>
    </Box>
  );
}

export default Recarga;
```

- [ ] **Step 3: Run Recarga test and fix if needed**

```bash
cd frontend && npx jest Recarga.test.js --no-coverage 2>&1
```

- [ ] **Step 4: Update App.js — add imports and routes**

Read existing `App.js`, then add imports:
```jsx
import Extrato from './pages/Extrato';
import Transferencia from './pages/Transferencia';
import Pagamento from './pages/Pagamento';
import Recarga from './pages/Recarga';
```

Add routes after existing `/credito` route:
```jsx
<Route path="/extrato" element={<Extrato user={user} />} />
<Route path="/transferencia" element={<Transferencia user={user} />} />
<Route path="/pagamento" element={<Pagamento user={user} />} />
<Route path="/recarga" element={<Recarga user={user} />} />
```

- [ ] **Step 5: Update Sidebar.js — add navigation links**

Read existing `Sidebar.js`, find the menu items list, and add 4 new items after the existing items (before the `divider` if any):

```jsx
<ListItem disablePadding>
  <ListItemButton component={Link} to="/extrato" selected={location.pathname === '/extrato'}>
    <ListItemIcon><Receipt /></ListItemIcon>
    <ListItemText primary="Extrato" />
  </ListItemButton>
</ListItem>
<ListItem disablePadding>
  <ListItemButton component={Link} to="/transferencia" selected={location.pathname === '/transferencia'}>
    <ListItemIcon><SwapHoriz /></ListItemIcon>
    <ListItemText primary="Transferência" />
  </ListItemButton>
</ListItem>
<ListItem disablePadding>
  <ListItemButton component={Link} to="/pagamento" selected={location.pathname === '/pagamento'}>
    <ListItemIcon><Payment /></ListItemIcon>
    <ListItemText primary="Pagamento" />
  </ListItemButton>
</ListItem>
<ListItem disablePadding>
  <ListItemButton component={Link} to="/recarga" selected={location.pathname === '/recarga'}>
    <ListItemIcon><PhoneAndroid /></ListItemIcon>
    <ListItemText primary="Recarga" />
  </ListItemButton>
</ListItem>
```

Ensure the imports in `Sidebar.js` include the new icons: `Receipt`, `SwapHoriz`, `Payment`, `PhoneAndroid` from `@mui/icons-material`.

- [ ] **Step 6: Run all page tests to verify**

```bash
cd frontend && npx jest Extrato Transferencia Pagamento Recarga --no-coverage 2>&1
```

Expected: All PASS

- [ ] **Step 7: Commit**

```bash
git add frontend/aureus-web/src/pages/Recarga.js frontend/aureus-web/src/pages/Recarga.test.js frontend/aureus-web/src/pages/Extrato.js frontend/aureus-web/src/pages/Extrato.test.js frontend/aureus-web/src/pages/Transferencia.js frontend/aureus-web/src/pages/Transferencia.test.js frontend/aureus-web/src/pages/Pagamento.js frontend/aureus-web/src/pages/Pagamento.test.js frontend/aureus-web/src/App.js frontend/aureus-web/src/components/Sidebar.js
git commit -m "feat(web): add Extrato, Transferencia, Pagamento, Recarga pages with routes and sidebar"
```

---

### Task G5-1: Create service interfaces for ML, Chatbot, BI

**Files:**
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/FraudService.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/CreditScoreService.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/ChatbotService.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/BiService.java`

- [ ] **Step 1: Create FraudService interface**

```java
package com.aureus.platform.analytics.service;

import java.util.Map;

public interface FraudService {
    Map<String, Object> avaliarFraude(Map<String, Object> transacao);
}
```

- [ ] **Step 2: Create CreditScoreService interface**

```java
package com.aureus.platform.analytics.service;

import java.util.Map;

public interface CreditScoreService {
    Map<String, Object> obterScore(String clienteId);
}
```

- [ ] **Step 3: Create ChatbotService interface**

```java
package com.aureus.platform.analytics.service;

import java.util.Map;

public interface ChatbotService {
    Map<String, Object> processarMensagem(String texto);
}
```

- [ ] **Step 4: Create BiService interface**

```java
package com.aureus.platform.analytics.service;

import java.util.Map;

public interface BiService {
    Map<String, Object> obterKpis();
    Map<String, Object> obterDashboard();
}
```

- [ ] **Step 5: Compile to verify**

```bash
mvn -pl aureus-analytics -am compile
```

Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/FraudService.java backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/CreditScoreService.java backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/ChatbotService.java backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/BiService.java
git commit -m "refactor(analytics): create service interfaces for ML, chatbot, BI"
```

---

### Task G5-2: Create stub implementations with @Profile("!prod")

**Files:**
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/MlFraudServiceStub.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/CreditScoreStubService.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/ChatbotStubService.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/BiStubService.java`

- [ ] **Step 1: Create MlFraudServiceStub**

```java
package com.aureus.platform.analytics.service;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;
import java.util.Map;

@Service
@Profile("!prod")
public class MlFraudServiceStub implements FraudService {

    @Override
    public Map<String, Object> avaliarFraude(Map<String, Object> transacao) {
        return Map.of(
            "riscoFraude", 0.02,
            "aprovado", true,
            "modelo", "stub",
            "mensagem", "Implementar modelo real de ML"
        );
    }
}
```

- [ ] **Step 2: Create CreditScoreStubService**

```java
package com.aureus.platform.analytics.service;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;
import java.util.Map;

@Service
@Profile("!prod")
public class CreditScoreStubService implements CreditScoreService {

    @Override
    public Map<String, Object> obterScore(String clienteId) {
        return Map.of(
            "clienteId", clienteId,
            "score", 750,
            "faixa", "BAIXO_RISCO",
            "modelo", "stub"
        );
    }
}
```

- [ ] **Step 3: Create ChatbotStubService**

```java
package com.aureus.platform.analytics.service;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;
import java.util.Map;

@Service
@Profile("!prod")
public class ChatbotStubService implements ChatbotService {

    @Override
    public Map<String, Object> processarMensagem(String texto) {
        return Map.of(
            "resposta", "Funcionalidade em implementação. Em breve teremos atendimento automatizado.",
            "intencao", "stub",
            "escalarParaHumano", false
        );
    }
}
```

- [ ] **Step 4: Create BiStubService**

```java
package com.aureus.platform.analytics.service;

import com.aureus.platform.analytics.repository.MetricaRepository;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;
import java.util.HashMap;
import java.util.Map;

@Service
@Profile("!prod")
public class BiStubService implements BiService {

    private final MetricaRepository metricaRepository;

    public BiStubService(MetricaRepository metricaRepository) {
        this.metricaRepository = metricaRepository;
    }

    @Override
    public Map<String, Object> obterKpis() {
        long totalMetricas = metricaRepository.count();
        Map<String, Object> kpis = new HashMap<>();
        kpis.put("totalMetricas", totalMetricas);
        kpis.put("transacoesHoje", 0);
        kpis.put("volumePixHoje", 0);
        kpis.put("status", "stub");
        return kpis;
    }

    @Override
    public Map<String, Object> obterDashboard() {
        return Map.of("kpis", Map.of("metricas", metricaRepository.count()), "alertas", 0, "status", "stub");
    }
}
```

- [ ] **Step 5: Compile to verify**

```bash
mvn -pl aureus-analytics -am compile
```

Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/MlFraudServiceStub.java backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/CreditScoreStubService.java backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/ChatbotStubService.java backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/BiStubService.java
git commit -m "refactor(analytics): add stub service implementations with @Profile(!prod)"
```

---

### Task G5-3: Create prod placeholder implementations

**Files:**
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/MlFraudServiceProd.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/CreditScoreServiceProd.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/ChatbotServiceProd.java`
- Create: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/BiServiceProd.java`

- [ ] **Step 1: Create MlFraudServiceProd**

```java
package com.aureus.platform.analytics.service;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;
import java.util.Map;

@Service
@Profile("prod")
public class MlFraudServiceProd implements FraudService {

    @Override
    public Map<String, Object> avaliarFraude(Map<String, Object> transacao) {
        return Map.of("riscoFraude", 0.0, "aprovado", true, "modelo", "prod-placeholder");
    }
}
```

- [ ] **Step 2: Create CreditScoreServiceProd**

```java
package com.aureus.platform.analytics.service;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;
import java.util.Map;

@Service
@Profile("prod")
public class CreditScoreServiceProd implements CreditScoreService {

    @Override
    public Map<String, Object> obterScore(String clienteId) {
        return Map.of("clienteId", clienteId, "score", 500, "faixa", "MEDIO_RISCO", "modelo", "prod-placeholder");
    }
}
```

- [ ] **Step 3: Create ChatbotServiceProd**

```java
package com.aureus.platform.analytics.service;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;
import java.util.Map;

@Service
@Profile("prod")
public class ChatbotServiceProd implements ChatbotService {

    @Override
    public Map<String, Object> processarMensagem(String texto) {
        return Map.of("resposta", "Funcionalidade em implementação", "escalarParaHumano", true);
    }
}
```

- [ ] **Step 4: Create BiServiceProd**

```java
package com.aureus.platform.analytics.service;

import com.aureus.platform.analytics.repository.MetricaRepository;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;
import java.util.Map;

@Service
@Profile("prod")
public class BiServiceProd implements BiService {

    private final MetricaRepository metricaRepository;

    public BiServiceProd(MetricaRepository metricaRepository) {
        this.metricaRepository = metricaRepository;
    }

    @Override
    public Map<String, Object> obterKpis() {
        return Map.of("totalMetricas", metricaRepository.count(), "transacoesHoje", 0, "volumePixHoje", 0, "status", "prod-placeholder");
    }

    @Override
    public Map<String, Object> obterDashboard() {
        return Map.of("kpis", Map.of("metricas", metricaRepository.count()), "alertas", 0, "status", "prod-placeholder");
    }
}
```

- [ ] **Step 5: Compile**

```bash
mvn -pl aureus-analytics -am compile
```

Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/MlFraudServiceProd.java backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/CreditScoreServiceProd.java backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/ChatbotServiceProd.java backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/service/BiServiceProd.java
git commit -m "refactor(analytics): add prod placeholder service implementations"
```

---

### Task G5-4: Refactor controllers to use interfaces

**Files:**
- Replace: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/controller/MlStubController.java` → renamed to `MlController.java`
- Replace: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/controller/ChatbotStubController.java` → renamed to `ChatbotController.java`
- Replace: `backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/controller/BiStubController.java` → renamed to `BiController.java`
- Delete: old stub controller files
- Modify: `backend/aureus-analytics/src/test/java/com/aureus/platform/analytics/integration/AnalyticsFlowIntegrationTest.java`

- [ ] **Step 1: Create MlController.java** (replacing MlStubController)

```java
package com.aureus.platform.analytics.controller;

import com.aureus.platform.analytics.service.FraudService;
import com.aureus.platform.analytics.service.CreditScoreService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.Map;

@RestController
@RequestMapping("/api/analytics/ml")
@Tag(name = "ML", description = "Detecção de fraude e score de crédito")
public class MlController {

    private final FraudService fraudService;
    private final CreditScoreService creditScoreService;

    public MlController(FraudService fraudService, CreditScoreService creditScoreService) {
        this.fraudService = fraudService;
        this.creditScoreService = creditScoreService;
    }

    @PostMapping("/fraude/avaliar")
    @Operation(summary = "Avaliar transação para fraude")
    public ResponseEntity<Map<String, Object>> avaliarFraude(@RequestBody Map<String, Object> transacao) {
        return ResponseEntity.ok(fraudService.avaliarFraude(transacao));
    }

    @GetMapping("/credito/score")
    @Operation(summary = "Score de crédito do cliente")
    public ResponseEntity<Map<String, Object>> scoreCredito(@RequestParam String clienteId) {
        return ResponseEntity.ok(creditScoreService.obterScore(clienteId));
    }
}
```

- [ ] **Step 2: Create ChatbotController.java** (replacing ChatbotStubController)

```java
package com.aureus.platform.analytics.controller;

import com.aureus.platform.analytics.service.ChatbotService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.Map;

@RestController
@RequestMapping("/api/analytics/chatbot")
@Tag(name = "Chatbot", description = "Assistente virtual")
public class ChatbotController {

    private final ChatbotService chatbotService;

    public ChatbotController(ChatbotService chatbotService) {
        this.chatbotService = chatbotService;
    }

    @PostMapping("/mensagem")
    @Operation(summary = "Enviar mensagem ao chatbot")
    public ResponseEntity<Map<String, Object>> enviar(@RequestBody Map<String, String> body) {
        String texto = body != null ? body.get("texto") : null;
        return ResponseEntity.ok(chatbotService.processarMensagem(texto));
    }
}
```

- [ ] **Step 3: Create BiController.java** (replacing BiStubController)

```java
package com.aureus.platform.analytics.controller;

import com.aureus.platform.analytics.service.BiService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.Map;

@RestController
@RequestMapping("/api/analytics/bi")
@Tag(name = "Business Intelligence", description = "KPIs e dashboards")
public class BiController {

    private final BiService biService;

    public BiController(BiService biService) {
        this.biService = biService;
    }

    @GetMapping("/kpis")
    @Operation(summary = "KPIs consolidados")
    public ResponseEntity<Map<String, Object>> kpis() {
        return ResponseEntity.ok(biService.obterKpis());
    }

    @GetMapping("/dashboard")
    @Operation(summary = "Resumo dashboard")
    public ResponseEntity<Map<String, Object>> dashboard() {
        return ResponseEntity.ok(biService.obterDashboard());
    }
}
```

- [ ] **Step 4: Remove old stub controllers**

```bash
rm backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/controller/MlStubController.java
rm backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/controller/ChatbotStubController.java
rm backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/controller/BiStubController.java
```

- [ ] **Step 5: Update AnalyticsFlowIntegrationTest.java**

Read the existing file, update the 3 test methods to reference the new controller paths (which are the same since `@RequestMapping` paths unchanged):

- `shouldEvaluateFraudViaMlStub()` → rename to `shouldEvaluateFraud()`, update reference
- `shouldGetCreditScoreViaMlStub()` → rename to `shouldGetCreditScore()`
- `shouldSendMessageToChatbotStub()` → rename to `shouldSendMessageToChatbot()`
- Update `@DisplayName` to remove "(stub)" mentions
- Update any `MlStubController` references to `MlController`, etc.

- [ ] **Step 6: Compile and run tests**

```bash
mvn -pl aureus-analytics -am compile
mvn test -pl aureus-analytics -am -Dtest=AnalyticsFlowIntegrationTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD SUCCESS, TESTS PASS

- [ ] **Step 7: Commit**

```bash
git add backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/controller/MlController.java backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/controller/ChatbotController.java backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/controller/BiController.java backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/integration/AnalyticsFlowIntegrationTest.java && git rm backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/controller/MlStubController.java backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/controller/ChatbotStubController.java backend/aureus-analytics/src/main/java/com/aureus/platform/analytics/controller/BiStubController.java && git commit -m "refactor(analytics): replace stub controllers with interface-driven controllers"
```