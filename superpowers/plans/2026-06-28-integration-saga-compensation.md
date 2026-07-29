# Integration Saga Compensation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement compensating transactions for the `IntegrationService` saga chain — sync endpoints, error propagation, idempotency, and compensation.

**Architecture:** Orchestrated Saga. `IntegrationService` coordinates sync to Financial module, throws on failure, and calls Financial's compensation endpoints to rollback. Read-only methods (lookups, dashboard, pricing) remain unchanged.

**Tech Stack:** Java 25, Spring Boot 4.1.0, RestTemplate, JPA/H2 (tests)

## Global Constraints

- No Lombok — manual constructors/getters/setters
- Tests: `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `RestTemplate` + `@LocalServerPort` + H2 + `@TestConfiguration` for security
- Follow existing patterns in `aurix-financial` and `aurix-shared`
- Uncalled methods (`enviarDadosControladoria`, `calcularImpostos`, `registrarImpostoContabil`) remain unchanged (YAGNI)
- Read-only methods (lookups, dashboard, pricing) remain unchanged (no side effects = no compensation needed)

---
## File Map

### Create in `aurix-financial`:
- `src/main/java/.../controller/SyncController.java`
- `src/main/java/.../service/SyncService.java`
- `src/main/java/.../entity/ContaSincronizada.java`
- `src/main/java/.../entity/TransacaoSincronizada.java`
- `src/main/java/.../repository/ContaSincronizadaRepository.java`
- `src/main/java/.../repository/TransacaoSincronizadaRepository.java`
- `src/test/java/.../integration/SyncIntegrationTest.java`

### Modify in `aurix-shared`:
- `src/main/java/.../integration/IntegrationService.java` — saga: propagate errors, idempotency, compensation

### Create in `aurix-shared`:
- `src/test/java/.../integration/IntegrationServiceSagaTest.java`

---

### Task 1: Create sync entities + repositories in aurix-financial

**Files:**
- Create: `backend/aurix-financial/src/main/java/com/aurix/platform/financial/entity/ContaSincronizada.java`
- Create: `backend/aurix-financial/src/main/java/com/aurix/platform/financial/entity/TransacaoSincronizada.java`
- Create: `backend/aurix-financial/src/main/java/com/aurix/platform/financial/repository/ContaSincronizadaRepository.java`
- Create: `backend/aurix-financial/src/main/java/com/aurix/platform/financial/repository/TransacaoSincronizadaRepository.java`

- [ ] **Step 1: Create ContaSincronizada.java**

```java
package com.aurix.platform.financial.entity;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "contas_sincronizadas", schema = "aurix")
public class ContaSincronizada {

    public enum StatusSync {
        ATIVO, DESINCronizado
    }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "conta_id", nullable = false, unique = true, length = 50)
    private String contaId;

    @Column(name = "cliente_id", nullable = false, length = 50)
    private String clienteId;

    @Column(name = "saldo_inicial", nullable = false, precision = 18, scale = 2)
    private BigDecimal saldoInicial;

    @Column(name = "data_criacao", nullable = false)
    private LocalDateTime dataCriacao;

    @Column(name = "data_sincronizacao", nullable = false)
    private LocalDateTime dataSincronizacao;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 30)
    private StatusSync status;

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getContaId() { return contaId; }
    public void setContaId(String contaId) { this.contaId = contaId; }
    public String getClienteId() { return clienteId; }
    public void setClienteId(String clienteId) { this.clienteId = clienteId; }
    public BigDecimal getSaldoInicial() { return saldoInicial; }
    public void setSaldoInicial(BigDecimal saldoInicial) { this.saldoInicial = saldoInicial; }
    public LocalDateTime getDataCriacao() { return dataCriacao; }
    public void setDataCriacao(LocalDateTime dataCriacao) { this.dataCriacao = dataCriacao; }
    public LocalDateTime getDataSincronizacao() { return dataSincronizacao; }
    public void setDataSincronizacao(LocalDateTime dataSincronizacao) { this.dataSincronizacao = dataSincronizacao; }
    public StatusSync getStatus() { return status; }
    public void setStatus(StatusSync status) { this.status = status; }
}
```

- [ ] **Step 2: Create TransacaoSincronizada.java**

```java
package com.aurix.platform.financial.entity;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "transacoes_sincronizadas", schema = "aurix")
public class TransacaoSincronizada {

    public enum StatusSync {
        REGISTRADA, ESTORNADA
    }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "transacao_id", nullable = false, unique = true, length = 50)
    private String transacaoId;

    @Column(name = "conta_id", nullable = false, length = 50)
    private String contaId;

    @Column(name = "valor", nullable = false, precision = 18, scale = 2)
    private BigDecimal valor;

    @Column(name = "tipo", nullable = false, length = 30)
    private String tipo;

    @Column(name = "data_transacao", nullable = false)
    private LocalDateTime dataTransacao;

    @Column(name = "data_sincronizacao", nullable = false)
    private LocalDateTime dataSincronizacao;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 30)
    private StatusSync status;

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getTransacaoId() { return transacaoId; }
    public void setTransacaoId(String transacaoId) { this.transacaoId = transacaoId; }
    public String getContaId() { return contaId; }
    public void setContaId(String contaId) { this.contaId = contaId; }
    public BigDecimal getValor() { return valor; }
    public void setValor(BigDecimal valor) { this.valor = valor; }
    public String getTipo() { return tipo; }
    public void setTipo(String tipo) { this.tipo = tipo; }
    public LocalDateTime getDataTransacao() { return dataTransacao; }
    public void setDataTransacao(LocalDateTime dataTransacao) { this.dataTransacao = dataTransacao; }
    public LocalDateTime getDataSincronizacao() { return dataSincronizacao; }
    public void setDataSincronizacao(LocalDateTime dataSincronizacao) { this.dataSincronizacao = dataSincronizacao; }
    public StatusSync getStatus() { return status; }
    public void setStatus(StatusSync status) { this.status = status; }
}
```

- [ ] **Step 3: Create ContaSincronizadaRepository.java**

```java
package com.aurix.platform.financial.repository;

import com.aurix.platform.financial.entity.ContaSincronizada;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface ContaSincronizadaRepository extends JpaRepository<ContaSincronizada, Long> {
    Optional<ContaSincronizada> findByContaId(String contaId);
    boolean existsByContaId(String contaId);
    void deleteByContaId(String contaId);
}
```

- [ ] **Step 4: Create TransacaoSincronizadaRepository.java**

```java
package com.aurix.platform.financial.repository;

import com.aurix.platform.financial.entity.TransacaoSincronizada;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface TransacaoSincronizadaRepository extends JpaRepository<TransacaoSincronizada, Long> {
    Optional<TransacaoSincronizada> findByTransacaoId(String transacaoId);
    boolean existsByTransacaoId(String transacaoId);
    void deleteByTransacaoId(String transacaoId);
}
```

- [ ] **Step 5: Compile**

```bash
mvn compile -pl aurix-financial -am -q
```

- [ ] **Step 6: Commit**

```bash
git add backend/aurix-financial/src/main/java/com/aurix/platform/financial/entity/ContaSincronizada.java \
       backend/aurix-financial/src/main/java/com/aurix/platform/financial/entity/TransacaoSincronizada.java \
       backend/aurix-financial/src/main/java/com/aurix/platform/financial/repository/ContaSincronizadaRepository.java \
       backend/aurix-financial/src/main/java/com/aurix/platform/financial/repository/TransacaoSincronizadaRepository.java
git commit -m "feat(financial): add sync entities for saga compensation"
```

---

### Task 2: Create SyncService + SyncController in aurix-financial

**Files:**
- Create: `SyncService.java`
- Create: `SyncController.java`

- [ ] **Step 1: Create SyncService.java**

```java
package com.aurix.platform.financial.service;

import com.aurix.platform.financial.entity.ContaSincronizada;
import com.aurix.platform.financial.entity.TransacaoSincronizada;
import com.aurix.platform.financial.repository.ContaSincronizadaRepository;
import com.aurix.platform.financial.repository.TransacaoSincronizadaRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Optional;

@Service
@Transactional
public class SyncService {
    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(SyncService.class);
    private final ContaSincronizadaRepository contaRepo;
    private final TransacaoSincronizadaRepository transacaoRepo;

    public SyncService(ContaSincronizadaRepository contaRepo, TransacaoSincronizadaRepository transacaoRepo) {
        this.contaRepo = contaRepo;
        this.transacaoRepo = transacaoRepo;
    }

    public ContaSincronizada sincronizarConta(String contaId, String clienteId, BigDecimal saldoInicial, LocalDateTime dataCriacao) {
        Optional<ContaSincronizada> existing = contaRepo.findByContaId(contaId);
        if (existing.isPresent()) {
            ContaSincronizada c = existing.get();
            if (c.getStatus() != ContaSincronizada.StatusSync.DESINCronizado) {
                log.info("Conta {} ja sincronizada, atualizando", contaId);
                c.setSaldoInicial(saldoInicial);
                c.setDataSincronizacao(LocalDateTime.now());
                return contaRepo.save(c);
            }
            contaRepo.delete(c);
            contaRepo.flush();
        }
        ContaSincronizada conta = new ContaSincronizada();
        conta.setContaId(contaId);
        conta.setClienteId(clienteId);
        conta.setSaldoInicial(saldoInicial);
        conta.setDataCriacao(dataCriacao);
        conta.setDataSincronizacao(LocalDateTime.now());
        conta.setStatus(ContaSincronizada.StatusSync.ATIVO);
        return contaRepo.save(conta);
    }

    public void desyncConta(String contaId) {
        Optional<ContaSincronizada> existing = contaRepo.findByContaId(contaId);
        if (existing.isPresent()) {
            ContaSincronizada c = existing.get();
            c.setStatus(ContaSincronizada.StatusSync.DESINCronizado);
            contaRepo.save(c);
            log.info("Conta {} desincronizada (compensation)", contaId);
        } else {
            log.warn("Conta {} nao encontrada para desync", contaId);
        }
    }

    public TransacaoSincronizada sincronizarTransacao(String transacaoId, String contaId, BigDecimal valor, String tipo, LocalDateTime dataTransacao) {
        Optional<TransacaoSincronizada> existing = transacaoRepo.findByTransacaoId(transacaoId);
        if (existing.isPresent()) {
            TransacaoSincronizada t = existing.get();
            if (t.getStatus() != TransacaoSincronizada.StatusSync.ESTORNADA) {
                log.info("Transacao {} ja sincronizada, atualizando", transacaoId);
                t.setValor(valor);
                t.setDataSincronizacao(LocalDateTime.now());
                return transacaoRepo.save(t);
            }
            transacaoRepo.delete(t);
            transacaoRepo.flush();
        }
        TransacaoSincronizada txn = new TransacaoSincronizada();
        txn.setTransacaoId(transacaoId);
        txn.setContaId(contaId);
        txn.setValor(valor);
        txn.setTipo(tipo);
        txn.setDataTransacao(dataTransacao);
        txn.setDataSincronizacao(LocalDateTime.now());
        txn.setStatus(TransacaoSincronizada.StatusSync.REGISTRADA);
        return transacaoRepo.save(txn);
    }

    public void desyncTransacao(String transacaoId) {
        Optional<TransacaoSincronizada> existing = transacaoRepo.findByTransacaoId(transacaoId);
        if (existing.isPresent()) {
            TransacaoSincronizada t = existing.get();
            t.setStatus(TransacaoSincronizada.StatusSync.ESTORNADA);
            transacaoRepo.save(t);
            log.info("Transacao {} estornada (compensation)", transacaoId);
        } else {
            log.warn("Transacao {} nao encontrada para desync", transacaoId);
        }
    }
}
```

- [ ] **Step 2: Create SyncController.java**

```java
package com.aurix.platform.financial.controller;

import com.aurix.platform.financial.service.SyncService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Map;

@RestController
@RequestMapping("/sincronizar")
@Tag(name = "Sync", description = "Sincronizacao Core -> Financial")
public class SyncController {
    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(SyncController.class);
    private final SyncService syncService;

    public SyncController(SyncService syncService) {
        this.syncService = syncService;
    }

    @PostMapping("/contas")
    @Operation(summary = "Sincronizar conta do Core")
    public ResponseEntity<Map<String, Object>> sincronizarConta(@RequestBody Map<String, Object> payload) {
        try {
            String contaId = (String) payload.get("contaId");
            String clienteId = (String) payload.get("clienteId");
            BigDecimal saldoInicial = new BigDecimal(payload.get("saldoInicial").toString());
            LocalDateTime dataCriacao = payload.get("dataCriacao") != null
                ? LocalDateTime.parse(payload.get("dataCriacao").toString().substring(0, 19))
                : LocalDateTime.now();
            var result = syncService.sincronizarConta(contaId, clienteId, saldoInicial, dataCriacao);
            return ResponseEntity.ok(Map.of(
                "id", result.getId(), "contaId", result.getContaId(),
                "status", result.getStatus().name(),
                "dataSincronizacao", result.getDataSincronizacao().toString()
            ));
        } catch (Exception e) {
            log.error("Erro ao sincronizar conta: {}", e.getMessage());
            return ResponseEntity.internalServerError().body(Map.of("erro", e.getMessage()));
        }
    }

    @DeleteMapping("/contas/{contaId}")
    @Operation(summary = "Desincronizar conta (compensation)")
    public ResponseEntity<Map<String, Object>> desyncConta(@PathVariable String contaId) {
        try {
            syncService.desyncConta(contaId);
            return ResponseEntity.ok(Map.of("status", "compensated", "contaId", contaId));
        } catch (Exception e) {
            return ResponseEntity.internalServerError().body(Map.of("erro", e.getMessage()));
        }
    }

    @PostMapping("/transacoes")
    @Operation(summary = "Sincronizar transacao do Core")
    public ResponseEntity<Map<String, Object>> sincronizarTransacao(@RequestBody Map<String, Object> payload) {
        try {
            String transacaoId = (String) payload.get("transacaoId");
            String contaId = (String) payload.get("contaId");
            BigDecimal valor = new BigDecimal(payload.get("valor").toString());
            String tipo = (String) payload.get("tipo");
            LocalDateTime dataTransacao = payload.get("dataTransacao") != null
                ? LocalDateTime.parse(payload.get("dataTransacao").toString().substring(0, 19))
                : LocalDateTime.now();
            var result = syncService.sincronizarTransacao(transacaoId, contaId, valor, tipo, dataTransacao);
            return ResponseEntity.ok(Map.of(
                "id", result.getId(), "transacaoId", result.getTransacaoId(),
                "status", result.getStatus().name(),
                "dataSincronizacao", result.getDataSincronizacao().toString()
            ));
        } catch (Exception e) {
            log.error("Erro ao sincronizar transacao: {}", e.getMessage());
            return ResponseEntity.internalServerError().body(Map.of("erro", e.getMessage()));
        }
    }

    @DeleteMapping("/transacoes/{transacaoId}")
    @Operation(summary = "Desincronizar transacao (compensation)")
    public ResponseEntity<Map<String, Object>> desyncTransacao(@PathVariable String transacaoId) {
        try {
            syncService.desyncTransacao(transacaoId);
            return ResponseEntity.ok(Map.of("status", "compensated", "transacaoId", transacaoId));
        } catch (Exception e) {
            return ResponseEntity.internalServerError().body(Map.of("erro", e.getMessage()));
        }
    }
}
```

- [ ] **Step 3: Compile**

```bash
mvn compile -pl aurix-financial -am -q
```

- [ ] **Step 4: Commit**

```bash
git add backend/aurix-financial/src/main/java/com/aurix/platform/financial/service/SyncService.java \
       backend/aurix-financial/src/main/java/com/aurix/platform/financial/controller/SyncController.java
git commit -m "feat(financial): sync service + controller with compensation endpoints"
```

---

### Task 3: Integration tests for SyncController in aurix-financial

**Files:**
- Create: `backend/aurix-financial/src/test/java/com/aurix/platform/financial/integration/SyncIntegrationTest.java`

- [ ] **Step 1: Write SyncIntegrationTest.java**

```java
package com.aurix.platform.financial.integration;

import com.aurix.platform.financial.AurixFinancialApplication;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.context.annotation.Bean;
import org.springframework.http.*;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.web.client.NoOpResponseErrorHandler;
import org.springframework.web.client.RestTemplate;
import java.time.LocalDateTime;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
    classes = {AurixFinancialApplication.class, SyncIntegrationTest.TestSecurityConfig.class})
@ActiveProfiles("test")
class SyncIntegrationTest {

    @TestConfiguration
    static class TestSecurityConfig {
        @Bean
        public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
            http.csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth.anyRequest().permitAll());
            return http.build();
        }
    }

    @LocalServerPort
    private int port;

    private final RestTemplate rest = new RestTemplate();
    private String baseUrl;

    @BeforeEach
    void setUp() {
        rest.setErrorHandler(new NoOpResponseErrorHandler());
        baseUrl = "http://localhost:" + port + "/sincronizar";
    }

    @Test
    void shouldSyncAndDesyncConta() {
        var payload = Map.of("contaId", "CONTA-001", "clienteId", "CLI-001",
            "saldoInicial", 1000.00, "dataCriacao", LocalDateTime.now().toString());
        ResponseEntity<Map> sync = rest.postForEntity(baseUrl + "/contas", payload, Map.class);
        assertEquals(HttpStatus.OK, sync.getStatusCode());
        assertEquals("CONTA-001", sync.getBody().get("contaId"));
        assertEquals("ATIVO", sync.getBody().get("status"));

        ResponseEntity<Map> desync = rest.exchange(baseUrl + "/contas/CONTA-001",
            HttpMethod.DELETE, null, Map.class);
        assertEquals(HttpStatus.OK, desync.getStatusCode());
        assertEquals("compensated", desync.getBody().get("status"));
    }

    @Test
    void shouldSyncAndDesyncTransacao() {
        var payload = Map.of("transacaoId", "TXN-001", "contaId", "CONTA-001",
            "valor", 500.00, "tipo", "DEBITO", "dataTransacao", LocalDateTime.now().toString());
        ResponseEntity<Map> sync = rest.postForEntity(baseUrl + "/transacoes", payload, Map.class);
        assertEquals(HttpStatus.OK, sync.getStatusCode());
        assertEquals("TXN-001", sync.getBody().get("transacaoId"));
        assertEquals("REGISTRADA", sync.getBody().get("status"));

        ResponseEntity<Map> desync = rest.exchange(baseUrl + "/transacoes/TXN-001",
            HttpMethod.DELETE, null, Map.class);
        assertEquals(HttpStatus.OK, desync.getStatusCode());
        assertEquals("compensated", desync.getBody().get("status"));
    }

    @Test
    void shouldBeIdempotentOnRepeatedSync() {
        var payload = Map.of("contaId", "CONTA-002", "clienteId", "CLI-001",
            "saldoInicial", 2000.00, "dataCriacao", LocalDateTime.now().toString());
        ResponseEntity<Map> first = rest.postForEntity(baseUrl + "/contas", payload, Map.class);
        assertEquals(HttpStatus.OK, first.getStatusCode());
        ResponseEntity<Map> second = rest.postForEntity(baseUrl + "/contas", payload, Map.class);
        assertEquals(HttpStatus.OK, second.getStatusCode());
        assertEquals(first.getBody().get("contaId"), second.getBody().get("contaId"));
    }

    @Test
    void shouldReturnErrorOnInvalidPayload() {
        var resp = rest.postForEntity(baseUrl + "/contas", Map.of("contaId", "CONTA-003"), Map.class);
        assertEquals(HttpStatus.INTERNAL_SERVER_ERROR, resp.getStatusCode());
    }

    @Test
    void shouldDesyncNonExistentContaGracefully() {
        ResponseEntity<Map> resp = rest.exchange(baseUrl + "/contas/NAO-EXISTE",
            HttpMethod.DELETE, null, Map.class);
        assertEquals(HttpStatus.OK, resp.getStatusCode());
        assertEquals("compensated", resp.getBody().get("status"));
    }
}
```

- [ ] **Step 2: Run tests**

```bash
mvn test -pl aurix-financial -am -Dtest=SyncIntegrationTest
```
Expected: 5/5 PASS

- [ ] **Step 3: Run all financial tests to verify no regressions**

```bash
mvn test -pl aurix-financial -am
```
Expected: FinancialFlowIntegrationTest (6) + SyncIntegrationTest (5) = 11 PASS

- [ ] **Step 4: Commit**

```bash
git add backend/aurix-financial/src/test/java/com/aurix/platform/financial/integration/SyncIntegrationTest.java
git commit -m "test(financial): sync + compensation integration tests"
```

---

### Task 4: Refactor IntegrationService in aurix-shared

**Files:**
- Modify: `backend/aurix-shared/src/main/java/com/aurix/platform/shared/integration/IntegrationService.java`

Changes to `sincronizarContaComFinanceiro`:
1. Change URL from `/api/financial/contas/sincronizar` to `/api/financial/sincronizar/contas`
2. Add `X-Idempotency-Key` header
3. Remove log-and-swallow — throw `RuntimeException` on failure
4. Caught exception triggers `compensarSyncConta` then rethrows

Changes to `sincronizarTransacaoComFinanceiro`: same 4 changes.

New private methods: `compensarSyncConta`, `compensarSyncTransacao`, `buildHeaders`.

- [ ] **Step 1: Edit `sincronizarContaComFinanceiro`**

Replace the method body (lines 67-87):

```java
    public void sincronizarContaComFinanceiro(final String contaId, String clienteId, BigDecimal saldoInicial) {
        log.info("SAGA: sincronizando conta {} com modulo financeiro", contaId);
        Map<String, Object> payload = new HashMap<>();
        payload.put("contaId", contaId);
        payload.put("clienteId", clienteId);
        payload.put("saldoInicial", saldoInicial);
        payload.put("dataCriacao", LocalDateTime.now());
        try {
            HttpEntity<Map<String, Object>> entity = new HttpEntity<>(payload, buildHeaders(contaId));
            ResponseEntity<Map> response = restTemplate.exchange(
                financialUrl + "/api/financial/sincronizar/contas", HttpMethod.POST, entity, Map.class);
            if (!response.getStatusCode().is2xxSuccessful()) {
                throw new RuntimeException("Falha ao sincronizar conta " + contaId + ": HTTP " + response.getStatusCode());
            }
            log.info("SAGA: conta {} sincronizada com sucesso", contaId);
        } catch (Exception e) {
            log.error("SAGA: erro ao sincronizar conta {}, executando compensacao: {}", contaId, e.getMessage());
            compensarSyncConta(contaId);
            throw new RuntimeException("Saga falhou ao sincronizar conta " + contaId, e);
        }
    }
```

- [ ] **Step 2: Edit `sincronizarTransacaoComFinanceiro`**

Replace method body (lines 92-113):

```java
    public void sincronizarTransacaoComFinanceiro(final String transacaoId, String contaId, BigDecimal valor, String tipo) {
        log.info("SAGA: sincronizando transacao {} com modulo financeiro", transacaoId);
        Map<String, Object> payload = new HashMap<>();
        payload.put("transacaoId", transacaoId);
        payload.put("contaId", contaId);
        payload.put("valor", valor);
        payload.put("tipo", tipo);
        payload.put("dataTransacao", LocalDateTime.now());
        try {
            HttpEntity<Map<String, Object>> entity = new HttpEntity<>(payload, buildHeaders(transacaoId));
            ResponseEntity<Map> response = restTemplate.exchange(
                financialUrl + "/api/financial/sincronizar/transacoes", HttpMethod.POST, entity, Map.class);
            if (!response.getStatusCode().is2xxSuccessful()) {
                throw new RuntimeException("Falha ao sincronizar transacao " + transacaoId + ": HTTP " + response.getStatusCode());
            }
            log.info("SAGA: transacao {} sincronizada com sucesso", transacaoId);
        } catch (Exception e) {
            log.error("SAGA: erro ao sincronizar transacao {}, executando compensacao: {}", transacaoId, e.getMessage());
            compensarSyncTransacao(transacaoId);
            throw new RuntimeException("Saga falhou ao sincronizar transacao " + transacaoId, e);
        }
    }
```

- [ ] **Step 3: Add compensation methods + buildHeaders**

After the last saga method (before "INTEGRAÇÃO FINANCIAL -> CONTROLLER" comment, around line 114), add:

```java
    private void compensarSyncConta(String contaId) {
        try {
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);
            headers.set("X-Compensation", "true");
            restTemplate.exchange(financialUrl + "/api/financial/sincronizar/contas/" + contaId,
                HttpMethod.DELETE, new HttpEntity<>(headers), Map.class);
            log.info("SAGA: compensacao da conta {} executada", contaId);
        } catch (Exception e) {
            log.error("SAGA: compensacao da conta {} tambem falhou: {}", contaId, e.getMessage());
        }
    }

    private void compensarSyncTransacao(String transacaoId) {
        try {
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);
            headers.set("X-Compensation", "true");
            restTemplate.exchange(financialUrl + "/api/financial/sincronizar/transacoes/" + transacaoId,
                HttpMethod.DELETE, new HttpEntity<>(headers), Map.class);
            log.info("SAGA: compensacao da transacao {} executada", transacaoId);
        } catch (Exception e) {
            log.error("SAGA: compensacao da transacao {} tambem falhou: {}", transacaoId, e.getMessage());
        }
    }

    private HttpHeaders buildHeaders(String idempotencyKey) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("X-Idempotency-Key", "saga-" + idempotencyKey + "-" + System.currentTimeMillis());
        return headers;
    }
```

- [ ] **Step 4: Compile**

```bash
mvn compile -pl aurix-shared -am -q
```

- [ ] **Step 5: Commit**

```bash
git add backend/aurix-shared/src/main/java/com/aurix/platform/shared/integration/IntegrationService.java
git commit -m "feat(shared): saga compensation + idempotency in IntegrationService"
```

---

### Task 5: Saga orchestration tests in aurix-shared

**Files:**
- Create: `backend/aurix-shared/src/test/java/com/aurix/platform/shared/integration/IntegrationServiceSagaTest.java`

- [ ] **Step 1: Write IntegrationServiceSagaTest.java**

```java
package com.aurix.platform.shared.integration;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.test.web.client.MockRestServiceServer;
import org.springframework.web.client.RestTemplate;
import java.math.BigDecimal;
import static org.springframework.test.web.client.match.MockRestRequestMatchers.*;
import static org.springframework.test.web.client.response.MockRestResponseCreators.*;

class IntegrationServiceSagaTest {

    private final RestTemplate restTemplate = new RestTemplate();
    private IntegrationService service;
    private MockRestServiceServer mockServer;

    @BeforeEach
    void setUp() {
        service = new IntegrationService(restTemplate);
        mockServer = MockRestServiceServer.bindTo(restTemplate).build();
    }

    @Test
    void shouldSyncContaAndCallDesyncOnFailure() {
        mockServer.expect(requestTo("http://localhost:8081/api/financial/sincronizar/contas"))
            .andExpect(method(HttpMethod.POST))
            .andExpect(headerExists("X-Idempotency-Key"))
            .andRespond(withStatus(HttpStatus.INTERNAL_SERVER_ERROR));

        mockServer.expect(requestTo("http://localhost:8081/api/financial/sincronizar/contas/CONTA-001"))
            .andExpect(method(HttpMethod.DELETE))
            .andExpect(header("X-Compensation", "true"))
            .andRespond(withStatus(HttpStatus.OK));

        org.junit.jupiter.api.Assertions.assertThrows(RuntimeException.class,
            () -> service.sincronizarContaComFinanceiro("CONTA-001", "CLI-001", BigDecimal.valueOf(1000)));

        mockServer.verify();
    }

    @Test
    void shouldSyncTransacaoAndCallDesyncOnFailure() {
        mockServer.expect(requestTo("http://localhost:8081/api/financial/sincronizar/transacoes"))
            .andExpect(method(HttpMethod.POST))
            .andExpect(headerExists("X-Idempotency-Key"))
            .andRespond(withStatus(HttpStatus.INTERNAL_SERVER_ERROR));

        mockServer.expect(requestTo("http://localhost:8081/api/financial/sincronizar/transacoes/TXN-001"))
            .andExpect(method(HttpMethod.DELETE))
            .andExpect(header("X-Compensation", "true"))
            .andRespond(withStatus(HttpStatus.OK));

        org.junit.jupiter.api.Assertions.assertThrows(RuntimeException.class,
            () -> service.sincronizarTransacaoComFinanceiro("TXN-001", "CONTA-001", BigDecimal.valueOf(500), "DEBITO"));

        mockServer.verify();
    }

    @Test
    void shouldNotCallDesyncOnSuccessfulSync() {
        mockServer.expect(requestTo("http://localhost:8081/api/financial/sincronizar/contas"))
            .andExpect(method(HttpMethod.POST))
            .andRespond(withStatus(HttpStatus.OK));

        service.sincronizarContaComFinanceiro("CONTA-002", "CLI-001", BigDecimal.valueOf(2000));
        mockServer.verify();
    }

    @Test
    void shouldHandleCompensationFailureGracefully() {
        mockServer.expect(requestTo("http://localhost:8081/api/financial/sincronizar/contas"))
            .andExpect(method(HttpMethod.POST))
            .andRespond(withStatus(HttpStatus.INTERNAL_SERVER_ERROR));

        mockServer.expect(requestTo("http://localhost:8081/api/financial/sincronizar/contas/CONTA-003"))
            .andExpect(method(HttpMethod.DELETE))
            .andRespond(withStatus(HttpStatus.INTERNAL_SERVER_ERROR));

        org.junit.jupiter.api.Assertions.assertThrows(RuntimeException.class,
            () -> service.sincronizarContaComFinanceiro("CONTA-003", "CLI-001", BigDecimal.valueOf(3000)));

        mockServer.verify();
    }
}
```

- [ ] **Step 2: Run tests**

```bash
mvn test -pl aurix-shared -am -Dtest=IntegrationServiceSagaTest
```
Expected: 4/4 PASS

- [ ] **Step 3: Run ALL shared module tests to verify no regressions**

```bash
mvn test -pl aurix-shared -am
```

- [ ] **Step 4: Commit**

```bash
git add backend/aurix-shared/src/test/java/com/aurix/platform/shared/integration/IntegrationServiceSagaTest.java
git commit -m "test(shared): saga orchestration tests for compensation + idempotency"
```

---

### Task 6: Full build verification

- [ ] **Run full compile** — 37 modules, all modifications must compile

```bash
cd /mnt/c/Users/wende/Projects/aurix-platform/backend
mvn compile -q 2>&1 | tail -20
```
Expected: BUILD SUCCESS

- [ ] **Run financial module tests**

```bash
mvn test -pl aurix-financial -am
```
Expected: 11 PASS (6 original + 5 new)

- [ ] **Run shared module tests**

```bash
mvn test -pl aurix-shared -am
```
Expected: existing tests + 4 new saga tests = all PASS

- [ ] **Commit any final fixes**

```bash
git add -A
git diff --cached --stat
```
Verify only intended files are staged, then:
```bash
git commit -m "chore: final adjustments after saga compensation implementation"
```
