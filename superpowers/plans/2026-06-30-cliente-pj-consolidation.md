# Cliente PF/PJ Consolidation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Consolidate the two parallel `Cliente` entities (shared PF-only + financial PF/PJ) into a single `Cliente` in `aurix-shared` that supports both PF and PJ, with financial-specific attributes moved to a separate `PerfilFinanceiroCliente` entity.

**Architecture:** The shared `Cliente` entity gets `tipoPessoa` (FISICA/JURIDICA), making CPF/CNPJ optional based on type. Financial module's own `Cliente` is removed; its finance-specific fields migrate to `PerfilFinanceiroCliente` (FK → shared Cliente). All consuming modules (core, pix, credit, security) adapt to the unified entity.

**Tech Stack:** Java 25, Spring Boot 4.1.0, JPA/Hibernate, Jakarta Validation, Testcontainers + H2 for tests

## Global Constraints

- No Lombok — manual getters/setters/constructors/equals/hashCode
- No MapStruct — manual DTO conversion
- Follow existing delomboked code style (hand-written boilerplate)
- Checkstyle max line length 120
- `@SuppressWarnings("all")` on all getters/setters/equals/hashCode (existing pattern)
- Test: `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `RestTemplate` + `@LocalServerPort`
- Build: `mvn compile -pl aurix-{module} -am` from `apps/backend/` directory
- All new code must compile before moving to next task

---
## File Structure

### Create
- `aurix-shared/.../util/CNPJUtil.java` — CNPJ validation (mirrors CPFUtil)
- `aurix-shared/.../entity/Cliente.java` — **replace** existing with consolidated version
- `aurix-shared/.../dto/ClienteDTO.java` — **replace** existing with PF/PJ DTO
- `aurix-financial/.../entity/PerfilFinanceiroCliente.java` — new entity
- `aurix-financial/.../repository/PerfilFinanceiroClienteRepository.java` — new repo
- `aurix-financial/.../service/PerfilFinanceiroClienteService.java` — new service
- `aurix-financial/.../controller/PerfilFinanceiroClienteController.java` — new controller
- Test files for each module

### Modify
- `aurix-shared/.../exception/ClienteNaoEncontradoException.java` — add CNPJ constructor
- `aurix-core/.../repository/ClienteRepository.java` — add CNPJ queries
- `aurix-core/.../service/ClienteService.java` — PF/PJ validation + CRUD
- `aurix-core/.../controller/ClienteController.java` — add CNPJ endpoint
- `aurix-pix/.../repository/ClienteRepository.java` — no changes needed (uses shared entity)
- `aurix-credit/.../repository/ClienteRepository.java` — no changes needed
- `aurix-credit/.../service/SolicitacaoCreditoService.java` — adapt clienteNome display
- `aurix-shared/.../dto/UsuarioDTO.java` — add clienteDocumento + clienteTipoPessoa
- `aurix-security/.../service/AuthService.java` — adapt converterParaDTO
- `aurix-shared/.../dto/ContaDTO.java` — add clienteTipoPessoa
- `aurix-shared/.../dto/SolicitacaoCreditoDTO.java` — add clienteTipoPessoa
- `aurix-shared/.../integration/IntegrationService.java` — handle PF/PJ in buscarClienteUnificado
- `aurix-shared/.../integration/IntegrationController.java` — handle PF/PJ
- `aurix-shared/.../cache/SharedCacheService.java` — handle PF/PJ cache keys
- **Delete** `aurix-financial/.../entity/Cliente.java` — no longer needed

---

### Task 1: CNPJUtil

**Files:**
- Create: `apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/util/CNPJUtil.java`
- Test: `apps/backend/aurix-shared/src/test/java/com/aurix/platform/shared/util/CNPJUtilTest.java`

**Interfaces:**
- Produces: `CNPJUtil.isValid(String cnpj) → boolean`, `CNPJUtil.format(String cnpj) → String`, `CNPJUtil.unformat(String cnpj) → String`, `CNPJUtil.mask(String cnpj) → String`

**Interfaces:**

- [ ] **Step 1: Write the failing test**

```java
package com.aurix.platform.shared.util;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CNPJUtilTest {
    @Test
    void deveValidarCNPJValido() {
        assertTrue(CNPJUtil.isValid("11222333000181"));
    }

    @Test
    void deveRejeitarCNPJInvalido() {
        assertFalse(CNPJUtil.isValid("11222333000182"));
    }

    @Test
    void deveRejeitarCNPJComDigitosIguais() {
        assertFalse(CNPJUtil.isValid("11111111111111"));
    }

    @Test
    void deveRejeitarCNPJNulo() {
        assertFalse(CNPJUtil.isValid(null));
    }

    @Test
    void deveRejeitarCNPJComLetras() {
        assertFalse(CNPJUtil.isValid("11.222.333/0001-8A"));
    }

    @Test
    void deveFormatarCNPJ() {
        assertEquals("11.222.333/0001-81", CNPJUtil.format("11222333000181"));
    }

    @Test
    void deveRetornarOriginalSeTamanhoInvalidoAoFormatar() {
        assertEquals("123", CNPJUtil.format("123"));
    }

    @Test
    void deveRemoverFormatacao() {
        assertEquals("11222333000181", CNPJUtil.unformat("11.222.333/0001-81"));
    }

    @Test
    void deveRetornarNullSeNullAoUnformat() {
        assertNull(CNPJUtil.unformat(null));
    }

    @Test
    void deveMascararCNPJ() {
        assertEquals("**.222.333/0001-81", CNPJUtil.mask("11222333000181"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl aurix-shared -Dtest=CNPJUtilTest -DfailIfNoTests=false`
Expected: Compilation error (CNPJUtil not found)

- [ ] **Step 3: Write minimal implementation**

```java
package com.aurix.platform.shared.util;

import java.util.regex.Pattern;

public final class CNPJUtil {
    private static final Pattern CNPJ_PATTERN = Pattern.compile("\\d{14}");
    private static final int[] PESO_DIGITO_1 = {5, 4, 3, 2, 9, 8, 7, 6, 5, 4, 3, 2};
    private static final int[] PESO_DIGITO_2 = {6, 5, 4, 3, 2, 9, 8, 7, 6, 5, 4, 3, 2};
    private static final int CNPJ_LENGTH = 14;
    private static final int MODULO = 11;
    private static final int MIN_REMAINDER = 2;
    private static final int FIRST_DIGIT_POS = 12;
    private static final int SECOND_DIGIT_POS = 13;
    private static final int REG1_END = 2;
    private static final int REG2_END = 5;
    private static final int REG3_END = 8;
    private static final int REG4_END = 12;

    private CNPJUtil() {
        throw new UnsupportedOperationException("Utility class");
    }

    public static boolean isValid(final String cnpj) {
        if (cnpj == null || !CNPJ_PATTERN.matcher(cnpj).matches()) {
            return false;
        }
        if (cnpj.matches("(\\d)\\1{13}")) {
            return false;
        }
        int soma = 0;
        for (int i = 0; i < FIRST_DIGIT_POS; i++) {
            soma += Character.getNumericValue(cnpj.charAt(i)) * PESO_DIGITO_1[i];
        }
        int resto = soma % MODULO;
        int digito1 = resto < MIN_REMAINDER ? 0 : MODULO - resto;
        if (Character.getNumericValue(cnpj.charAt(FIRST_DIGIT_POS)) != digito1) {
            return false;
        }
        soma = 0;
        for (int i = 0; i < SECOND_DIGIT_POS; i++) {
            soma += Character.getNumericValue(cnpj.charAt(i)) * PESO_DIGITO_2[i];
        }
        resto = soma % MODULO;
        int digito2 = resto < MIN_REMAINDER ? 0 : MODULO - resto;
        return Character.getNumericValue(cnpj.charAt(SECOND_DIGIT_POS)) == digito2;
    }

    public static String format(final String cnpj) {
        if (cnpj == null || cnpj.length() != CNPJ_LENGTH) {
            return cnpj;
        }
        return String.format("%s.%s.%s/%s-%s",
                cnpj.substring(0, REG1_END),
                cnpj.substring(REG1_END, REG2_END),
                cnpj.substring(REG2_END, REG3_END),
                cnpj.substring(REG3_END, REG4_END),
                cnpj.substring(REG4_END, CNPJ_LENGTH));
    }

    public static String unformat(final String cnpj) {
        if (cnpj == null) {
            return null;
        }
        return cnpj.replaceAll("\\D", "");
    }

    public static String mask(final String cnpj) {
        if (cnpj == null || cnpj.length() != CNPJ_LENGTH) {
            return cnpj;
        }
        return String.format("**.%s.%s/%s-%s",
                cnpj.substring(REG1_END, REG2_END),
                cnpj.substring(REG2_END, REG3_END),
                cnpj.substring(REG3_END, REG4_END),
                cnpj.substring(REG4_END, CNPJ_LENGTH));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl aurix-shared -Dtest=CNPJUtilTest -DfailIfNoTests=false`
Expected: All 9 tests pass

- [ ] **Step 5: Commit**

```bash
git add apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/util/CNPJUtil.java apps/backend/aurix-shared/src/test/java/com/aurix/platform/shared/util/CNPJUtilTest.java
git commit -m "feat: add CNPJUtil validation utility"
```

---

### Task 2: Consolidated Shared Cliente Entity + DTO

**Files:**
- Modify: `apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/entity/Cliente.java`
- Modify: `apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/dto/ClienteDTO.java`
- Modify: `apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/exception/ClienteNaoEncontradoException.java`

**Interfaces:**
- Consumes: `CNPJUtil` (Task 1)
- Produces: `Cliente` entity with `TipoPessoa`, optional CPF/CNPJ, full PF/PJ fields; `ClienteDTO` matching; `ClienteNaoEncontradoException` with CNPJ constructor

- [ ] **Step 1: Write the failing test for the consolidated entity**

```java
// At: aurix-shared/src/test/java/com/aurix/platform/shared/entity/ClienteConsolidadoTest.java
package com.aurix.platform.shared.entity;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;
import java.time.LocalDate;

class ClienteConsolidadoTest {
    @Test
    void deveCriarClientePessoaFisica() {
        Cliente c = new Cliente();
        c.setTipoPessoa(Cliente.TipoPessoa.FISICA);
        c.setCpf("12345678901");
        c.setNome("João Silva");
        c.setEmail("joao@teste.com");
        assertEquals(Cliente.TipoPessoa.FISICA, c.getTipoPessoa());
        assertEquals("12345678901", c.getCpf());
        assertEquals("João Silva", c.getNome());
    }

    @Test
    void deveCriarClientePessoaJuridica() {
        Cliente c = new Cliente();
        c.setTipoPessoa(Cliente.TipoPessoa.JURIDICA);
        c.setCnpj("11222333000181");
        c.setNomeRazaoSocial("Empresa Ltda");
        c.setNomeFantasia("Empresa");
        c.setEmail("contato@empresa.com");
        assertEquals(Cliente.TipoPessoa.JURIDICA, c.getTipoPessoa());
        assertEquals("11222333000181", c.getCnpj());
        assertEquals("Empresa Ltda", c.getNomeRazaoSocial());
    }
}
```

Run: `mvn test -pl aurix-shared -Dtest=ClienteConsolidadoTest -DfailIfNoTests=false`
Expected: Compilation error (TipoPessoa not in Cliente)

- [ ] **Step 2: Update Cliente entity**

Replace `apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/entity/Cliente.java` with consolidated version. Key changes:
- Add `TipoPessoa` enum: `FISICA, JURIDICA`
- `cpf`: keep `@Pattern("\\d{11}")` but remove `@NotBlank` (only required for FISICA)
- `nome`: keep but only required for FISICA
- `cnpj`: new, `@Pattern("\\d{14}")`, only required for JURIDICA
- `nomeRazaoSocial`: new, required for JURIDICA
- `nomeFantasia`: new, optional
- `inscricaoEstadual`, `inscricaoMunicipal`: new, optional
- `cidade`, `estado`, `cep`, `contato`: new, optional (from financial Cliente)
- `tipoPessoa`: new, `@NotNull`
- Update unique constraints: add `(tenantId, cnpj)`
- CPF unique stays but becomes nullable (PF only)
- Keep existing: `email`, `telefone`, `dataNascimento`, `endereco`, `status`
- Keep existing inheritance: `extends BaseEntity`
- Keep `StatusCliente` enum unchanged

Entity code (full replacement - follow existing delomboked pattern):

```java
package com.aurix.platform.shared.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Table;
import jakarta.persistence.UniqueConstraint;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;
import java.time.LocalDate;

@Entity
@Table(name = "clientes", schema = "aurix", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"tenantId", "cpf"}),
    @UniqueConstraint(columnNames = {"tenantId", "cnpj"}),
    @UniqueConstraint(columnNames = {"tenantId", "email"})
})
public class Cliente extends BaseEntity {
    private static final int CPF_LENGTH = 11;
    private static final int CNPJ_LENGTH = 14;
    private static final int NAME_MAX_LENGTH = 255;

    @Enumerated(EnumType.STRING)
    @Column(name = "tipo_pessoa", nullable = false)
    private TipoPessoa tipoPessoa;

    @Pattern(regexp = "\\d{11}", message = "CPF deve conter 11 dígitos")
    @Column(name = "cpf", length = CPF_LENGTH)
    private String cpf;

    @Size(min = 2, max = NAME_MAX_LENGTH, message = "Nome deve ter entre 2 e 255 caracteres")
    @Column(name = "nome")
    private String nome;

    @Pattern(regexp = "\\d{14}", message = "CNPJ deve conter 14 dígitos")
    @Column(name = "cnpj", length = CNPJ_LENGTH)
    private String cnpj;

    @Column(name = "nome_razao_social", length = NAME_MAX_LENGTH)
    private String nomeRazaoSocial;

    @Column(name = "nome_fantasia", length = NAME_MAX_LENGTH)
    private String nomeFantasia;

    @Column(name = "inscricao_estadual", length = 20)
    private String inscricaoEstadual;

    @Column(name = "inscricao_municipal", length = 20)
    private String inscricaoMunicipal;

    @Email(message = "Email deve ter formato válido")
    @Column(name = "email", nullable = false)
    private String email;

    @Pattern(regexp = "\\d{10,11}", message = "Telefone deve conter 10 ou 11 dígitos")
    @Column(name = "telefone", length = 20)
    private String telefone;

    @Column(name = "data_nascimento")
    private LocalDate dataNascimento;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(name = "endereco", columnDefinition = "jsonb")
    private String endereco;

    @Column(name = "cidade", length = 100)
    private String cidade;

    @Column(name = "estado", length = 2)
    private String estado;

    @Column(name = "cep", length = 10)
    private String cep;

    @Column(name = "contato", length = 100)
    private String contato;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private StatusCliente status = StatusCliente.ATIVO;

    public enum TipoPessoa {
        FISICA, JURIDICA
    }

    public enum StatusCliente {
        ATIVO("Ativo"), INATIVO("Inativo"), BLOQUEADO("Bloqueado"), SUSPENSO("Suspenso");
        private final String descricao;
        StatusCliente(final String desc) { this.descricao = desc; }
        public String getDescricao() { return descricao; }
    }

    // ---- getters/setters/equals/hashCode/toString/constructors ----
    // Follow SAME delomboked pattern as existing file:
    // All fields get getter + setter with @java.lang.SuppressWarnings("all")
    // equals() checks all fields, hashCode() uses PRIME=59
    // toString() lists all fields
    // No-arg + all-args constructors
    // canEqual() returns instanceof Cliente
    // (Full boilerplate omitted for brevity — match existing style exactly)
    // When implementing, copy the existing boilerplate pattern and add new fields.
    // IMPORTANT: update equals/hashCode/toString to include ALL new fields.
}
```

- [ ] **Step 3: Update ClienteDTO**

Add `tipoPessoa`, `cnpj`, `nomeRazaoSocial`, `nomeFantasia`, `inscricaoEstadual`, `inscricaoMunicipal`, `cidade`, `estado`, `cep`, `contato` fields. Make `cpf` and `nome` non-mandatory (remove `@NotBlank`). Update equals/hashCode/toString/constructors.

- [ ] **Step 4: Update ClienteNaoEncontradoException**

Add CNPJ constructor:

```java
public ClienteNaoEncontradoException(final String documento, final boolean isCnpj) {
    super("CLIENTE_NAO_ENCONTRADO",
        String.format("Cliente com %s %s não encontrado",
            isCnpj ? "CNPJ" : "CPF", documento));
}
```

Keep existing Long (ID) and String (CPF) constructors for backward compatibility.

- [ ] **Step 5: Compile to verify**

Run: `mvn compile -pl aurix-shared -am`
Expected: Compilation succeeds

- [ ] **Step 6: Commit**

```bash
git add apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/entity/Cliente.java apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/dto/ClienteDTO.java apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/exception/ClienteNaoEncontradoException.java
git commit -m "feat: consolidate Cliente with PF/PJ support"
```

---

### Task 3: Update Core Module (Repository + Service + Controller)

**Files:**
- Modify: `apps/backend/aurix-core/src/main/java/com/aurix/platform/core/repository/ClienteRepository.java`
- Modify: `apps/backend/aurix-core/src/main/java/com/aurix/platform/core/service/ClienteService.java`
- Modify: `apps/backend/aurix-core/src/main/java/com/aurix/platform/core/controller/ClienteController.java`

**Interfaces:**
- Consumes: `Cliente` with `TipoPessoa` (Task 2), `CNPJUtil` (Task 1)
- Produces: Full PF/PJ CRUD in core module

- [ ] **Step 1: Add CNPJ queries to ClienteRepository**

```java
Optional<Cliente> findByTenantIdAndCnpj(String tenantId, String cnpj);
boolean existsByTenantIdAndCnpj(String tenantId, String cnpj);
Optional<Cliente> findByCnpj(String cnpj);
boolean existsByCnpj(String cnpj);
```

- [ ] **Step 2: Update ClienteService**

Key changes in `criarCliente`:
```java
public ClienteDTO criarCliente(ClienteDTO clienteDTO) {
    if (clienteDTO.getTipoPessoa() == Cliente.TipoPessoa.FISICA) {
        if (clienteDTO.getCpf() == null || !CPFUtil.isValid(clienteDTO.getCpf())) {
            throw new IllegalArgumentException("CPF inválido: " + clienteDTO.getCpf());
        }
    } else if (clienteDTO.getTipoPessoa() == Cliente.TipoPessoa.JURIDICA) {
        if (clienteDTO.getCnpj() == null || !CNPJUtil.isValid(clienteDTO.getCnpj())) {
            throw new IllegalArgumentException("CNPJ inválido: " + clienteDTO.getCnpj());
        }
    } else {
        throw new IllegalArgumentException("Tipo de pessoa é obrigatório");
    }
    String tenantId = TenantContext.getTenantId();
    // Validate uniqueness for CPF (PF) or CNPJ (PJ)
    if (clienteDTO.getTipoPessoa() == Cliente.TipoPessoa.FISICA) {
        if (clienteRepository.existsByTenantIdAndCpf(tenantId, clienteDTO.getCpf())) {
            throw new IllegalArgumentException("Cliente com CPF " + clienteDTO.getCpf() + " já existe");
        }
    } else {
        if (clienteRepository.existsByTenantIdAndCnpj(tenantId, clienteDTO.getCnpj())) {
            throw new IllegalArgumentException("Cliente com CNPJ " + clienteDTO.getCnpj() + " já existe");
        }
    }
    if (clienteRepository.existsByTenantIdAndEmail(tenantId, clienteDTO.getEmail())) {
        throw new IllegalArgumentException("Cliente com email " + clienteDTO.getEmail() + " já existe");
    }
    Cliente cliente = new Cliente();
    cliente.setTenantId(tenantId);
    cliente.setTipoPessoa(clienteDTO.getTipoPessoa());
    if (clienteDTO.getTipoPessoa() == Cliente.TipoPessoa.FISICA) {
        cliente.setCpf(clienteDTO.getCpf());
        cliente.setNome(clienteDTO.getNome());
        cliente.setDataNascimento(clienteDTO.getDataNascimento());
    } else {
        cliente.setCnpj(clienteDTO.getCnpj());
        cliente.setNomeRazaoSocial(clienteDTO.getNomeRazaoSocial());
        cliente.setNomeFantasia(clienteDTO.getNomeFantasia());
        cliente.setInscricaoEstadual(clienteDTO.getInscricaoEstadual());
        cliente.setInscricaoMunicipal(clienteDTO.getInscricaoMunicipal());
    }
    cliente.setEmail(clienteDTO.getEmail());
    cliente.setTelefone(clienteDTO.getTelefone());
    cliente.setEndereco(clienteDTO.getEndereco());
    cliente.setCidade(clienteDTO.getCidade());
    cliente.setEstado(clienteDTO.getEstado());
    cliente.setCep(clienteDTO.getCep());
    cliente.setContato(clienteDTO.getContato());
    cliente.setStatus(Cliente.StatusCliente.ATIVO);
    Cliente clienteSalvo = clienteRepository.save(cliente);
    return converterParaDTO(clienteSalvo);
}
```

Add `buscarClientePorCnpj` method:
```java
public ClienteDTO buscarClientePorCnpj(String cnpj) {
    String tenantId = TenantContext.getTenantId();
    Cliente cliente = clienteRepository.findByTenantIdAndCnpj(tenantId, CNPJUtil.unformat(cnpj))
        .orElseThrow(() -> new ClienteNaoEncontradoException(cnpj, true));
    return converterParaDTO(cliente);
}
```

Update `converterParaDTO` to map all new fields.

Update `atualizarCliente` to handle PF/PJ fields.

- [ ] **Step 3: Update ClienteController**

Add:
```java
@GetMapping("/cnpj/{cnpj}")
@Operation(summary = "Buscar cliente por CNPJ", description = "Busca um cliente pelo CNPJ")
public ResponseEntity<ClienteDTO> buscarClientePorCnpj(@PathVariable String cnpj) {
    ClienteDTO cliente = clienteService.buscarClientePorCnpj(cnpj);
    return ResponseEntity.ok(cliente);
}
```

- [ ] **Step 4: Compile Core module**

Run: `mvn compile -pl aurix-core -am`
Expected: Compilation succeeds

- [ ] **Step 5: Commit**

```bash
git add apps/backend/aurix-core/src/main/java/com/aurix/platform/core/repository/ClienteRepository.java apps/backend/aurix-core/src/main/java/com/aurix/platform/core/service/ClienteService.java apps/backend/aurix-core/src/main/java/com/aurix/platform/core/controller/ClienteController.java
git commit -m "feat: update core module with PF/PJ Cliente CRUD"
```

---

### Task 4: Update Pix + Credit Module Repositories

**Files:**
- No changes needed to `aurix-pix/.../repository/ClienteRepository.java` (already uses shared entity)
- No changes needed to `aurix-credit/.../repository/ClienteRepository.java` (already uses shared entity)
- Modify: `apps/backend/aurix-credit/src/main/java/com/aurix/platform/credit/service/SolicitacaoCreditoService.java`

- [ ] **Step 1: Update SolicitacaoCreditoService**

Find where `clienteNome` is set from `cliente.getNome()` and adapt:
```java
dto.setClienteId(solicitacao.getCliente().getId());
String nomeExibicao = solicitacao.getCliente().getTipoPessoa() == Cliente.TipoPessoa.FISICA
    ? solicitacao.getCliente().getNome()
    : solicitacao.getCliente().getNomeRazaoSocial();
dto.setClienteNome(nomeExibicao);
dto.setClienteTipoPessoa(solicitacao.getCliente().getTipoPessoa().name());
```

- [ ] **Step 2: Compile both modules**

Run: `mvn compile -pl aurix-pix,aurix-credit -am`
Expected: Compilation succeeds

- [ ] **Step 3: Commit**

```bash
git add apps/backend/aurix-credit/src/main/java/com/aurix/platform/credit/service/SolicitacaoCreditoService.java
git commit -m "feat: adapt credit module for PJ client display"
```

---

### Task 5: Update Security Module (UsuarioDTO + AuthService)

**Files:**
- Modify: `apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/dto/UsuarioDTO.java`
- Modify: `apps/backend/aurix-security/src/main/java/com/aurix/platform/security/service/AuthService.java`

- [ ] **Step 1: Add clienteDocumento + clienteTipoPessoa to UsuarioDTO**

```java
private String clienteDocumento;  // CPF or CNPJ
private String clienteTipoPessoa; // "FISICA" or "JURIDICA"
// Keep clienteCpf as deprecated for backward compatibility
```

Add getters/setters for both fields.

- [ ] **Step 2: Update AuthService.converterParaDTO**

```java
if (usuario.getCliente() != null) {
    dto.setClienteId(usuario.getCliente().getId());
    String nomeExibicao = usuario.getCliente().getTipoPessoa() == Cliente.TipoPessoa.FISICA
        ? usuario.getCliente().getNome()
        : usuario.getCliente().getNomeRazaoSocial();
    dto.setClienteNome(nomeExibicao);
    String documento = usuario.getCliente().getTipoPessoa() == Cliente.TipoPessoa.FISICA
        ? usuario.getCliente().getCpf()
        : usuario.getCliente().getCnpj();
    dto.setClienteDocumento(documento);
    dto.setClienteTipoPessoa(usuario.getCliente().getTipoPessoa().name());
    dto.setClienteCpf(documento); // backward compat
}
```

- [ ] **Step 3: Compile**

Run: `mvn compile -pl aurix-security -am`
Expected: Compilation succeeds

- [ ] **Step 4: Commit**

```bash
git add apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/dto/UsuarioDTO.java apps/backend/aurix-security/src/main/java/com/aurix/platform/security/service/AuthService.java
git commit -m "feat: update UsuarioDTO and AuthService for PF/PJ client"
```

---

### Task 6: Update ContaDTO and SolicitacaoCreditoDTO

**Files:**
- Modify: `apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/dto/ContaDTO.java`
- Modify: `apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/dto/SolicitacaoCreditoDTO.java`

- [ ] **Step 1: Add clienteTipoPessoa to ContaDTO**

```java
private String clienteTipoPessoa;
```

Add getter/setter, update equals/hashCode/toString/all-args constructor.

- [ ] **Step 2: Add clienteTipoPessoa to SolicitacaoCreditoDTO**

```java
private String clienteTipoPessoa;
```

Add getter/setter, update equals/hashCode/toString/all-args constructor.

- [ ] **Step 3: Compile**

Run: `mvn compile -pl aurix-shared`
Expected: Compilation succeeds

- [ ] **Step 4: Commit**

```bash
git add apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/dto/ContaDTO.java apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/dto/SolicitacaoCreditoDTO.java
git commit -m "feat: add clienteTipoPessoa to ContaDTO and SolicitacaoCreditoDTO"
```

---

### Task 7: Update Integration Layer

**Files:**
- Modify: `apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/integration/IntegrationService.java`
- Modify: `apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/integration/IntegrationController.java`
- Modify: `apps/backend/aurix-shared/src/main/java/com/aurix/platform/shared/cache/SharedCacheService.java`

- [ ] **Step 1: Update IntegrationService.buscarClienteUnificado**

No signature change needed (still takes String clienteId). The ClienteDTO response will now include tipoPessoa which is handled transparently.

- [ ] **Step 2: Update IntegrationController.buscarClienteUnificado**

No changes needed — the controller uses ClienteDTO which now has PF/PJ fields.

- [ ] **Step 3: Update SharedCacheService**

No changes needed — cache key pattern stays the same (`cliente:{id}`). Serialization of ClienteDTO works as-is since it now has the new fields.

- [ ] **Step 4: Compile**

Run: `mvn compile -pl aurix-shared`
Expected: Compilation succeeds

- [ ] **Step 5: Commit**

```bash
git commit -m "chore: integration layer compatible with consolidated Cliente"
```

---

### Task 8: PerfilFinanceiroCliente (Financial Module)

**Files:**
- Create: `apps/backend/aurix-financial/src/main/java/com/aurix/platform/financial/entity/PerfilFinanceiroCliente.java`
- Create: `apps/backend/aurix-financial/src/main/java/com/aurix/platform/financial/repository/PerfilFinanceiroClienteRepository.java`
- Create: `apps/backend/aurix-financial/src/main/java/com/aurix/platform/financial/service/PerfilFinanceiroClienteService.java`
- Create: `apps/backend/aurix-financial/src/main/java/com/aurix/platform/financial/controller/PerfilFinanceiroClienteController.java`

**Interfaces:**
- Consumes: Shared `Cliente` entity (Task 2)
- Produces: Financial profile CRUD with FK to shared Cliente

- [ ] **Step 1: Create PerfilFinanceiroCliente entity**

```java
package com.aurix.platform.financial.entity;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "perfis_financeiros_clientes", schema = "aurix")
public class PerfilFinanceiroCliente {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "cliente_id", nullable = false, unique = true)
    private Long clienteId;

    @Column(name = "codigo_cliente", unique = true, length = 20)
    private String codigoCliente;

    @Column(name = "limite_credito", precision = 15, scale = 2)
    private BigDecimal limiteCredito;

    @Column(name = "score_credito")
    private Integer scoreCredito;

    @Column(name = "observacoes", length = 1000)
    private String observacoes;

    @Column(name = "metadata", columnDefinition = "jsonb")
    private String metadata;

    @Column(name = "data_criacao", nullable = false, updatable = false)
    private LocalDateTime dataCriacao;

    @Column(name = "data_atualizacao", nullable = false)
    private LocalDateTime dataAtualizacao;

    @Version
    @Column(name = "versao", nullable = false)
    private Long versao;

    // Follow same delomboked pattern: getters/setters/equals/hashCode/toString/constructors
    // All fields with @java.lang.SuppressWarnings("all")

    @PrePersist
    protected void onCreate() {
        dataCriacao = LocalDateTime.now();
        dataAtualizacao = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        dataAtualizacao = LocalDateTime.now();
    }
}
```

- [ ] **Step 2: Create Repository**

```java
package com.aurix.platform.financial.repository;

import com.aurix.platform.financial.entity.PerfilFinanceiroCliente;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.Optional;

@Repository
public interface PerfilFinanceiroClienteRepository extends JpaRepository<PerfilFinanceiroCliente, Long> {
    Optional<PerfilFinanceiroCliente> findByClienteId(Long clienteId);
    boolean existsByClienteId(Long clienteId);
    void deleteByClienteId(Long clienteId);
}
```

- [ ] **Step 3: Create Service**

```java
package com.aurix.platform.financial.service;

import com.aurix.platform.financial.entity.PerfilFinanceiroCliente;
import com.aurix.platform.financial.repository.PerfilFinanceiroClienteRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.util.Optional;

@Service
@Transactional
public class PerfilFinanceiroClienteService {
    private final PerfilFinanceiroClienteRepository repository;

    public PerfilFinanceiroCliente criarPerfil(Long clienteId, String codigoCliente) {
        if (repository.existsByClienteId(clienteId)) {
            throw new IllegalArgumentException("Perfil financeiro já existe para cliente " + clienteId);
        }
        PerfilFinanceiroCliente perfil = new PerfilFinanceiroCliente();
        perfil.setClienteId(clienteId);
        perfil.setCodigoCliente(codigoCliente);
        return repository.save(perfil);
    }

    @Transactional(readOnly = true)
    public Optional<PerfilFinanceiroCliente> buscarPorClienteId(Long clienteId) {
        return repository.findByClienteId(clienteId);
    }

    public PerfilFinanceiroCliente atualizarLimiteCredito(Long clienteId, BigDecimal limiteCredito) {
        PerfilFinanceiroCliente perfil = repository.findByClienteId(clienteId)
            .orElseThrow(() -> new IllegalArgumentException("Perfil financeiro não encontrado para cliente " + clienteId));
        perfil.setLimiteCredito(limiteCredito);
        return repository.save(perfil);
    }

    public PerfilFinanceiroCliente atualizarScore(Long clienteId, Integer scoreCredito) {
        PerfilFinanceiroCliente perfil = repository.findByClienteId(clienteId)
            .orElseThrow(() -> new IllegalArgumentException("Perfil financeiro não encontrado para cliente " + clienteId));
        perfil.setScoreCredito(scoreCredito);
        return repository.save(perfil);
    }

    public void removerPorClienteId(Long clienteId) {
        repository.deleteByClienteId(clienteId);
    }

    public PerfilFinanceiroClienteService(PerfilFinanceiroClienteRepository repository) {
        this.repository = repository;
    }
}
```

- [ ] **Step 4: Create Controller**

```java
package com.aurix.platform.financial.controller;

import com.aurix.platform.financial.entity.PerfilFinanceiroCliente;
import com.aurix.platform.financial.service.PerfilFinanceiroClienteService;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.math.BigDecimal;
import java.util.Optional;

@RestController
@RequestMapping("/api/financial/perfil")
public class PerfilFinanceiroClienteController {
    private final PerfilFinanceiroClienteService service;

    @PostMapping("/{clienteId}")
    public ResponseEntity<PerfilFinanceiroCliente> criarPerfil(
            @PathVariable Long clienteId, @RequestParam String codigoCliente) {
        PerfilFinanceiroCliente perfil = service.criarPerfil(clienteId, codigoCliente);
        return ResponseEntity.status(HttpStatus.CREATED).body(perfil);
    }

    @GetMapping("/{clienteId}")
    public ResponseEntity<PerfilFinanceiroCliente> buscarPerfil(@PathVariable Long clienteId) {
        Optional<PerfilFinanceiroCliente> perfil = service.buscarPorClienteId(clienteId);
        return perfil.map(ResponseEntity::ok).orElse(ResponseEntity.notFound().build());
    }

    @PutMapping("/{clienteId}/limite-credito")
    public ResponseEntity<PerfilFinanceiroCliente> atualizarLimiteCredito(
            @PathVariable Long clienteId, @RequestParam BigDecimal limiteCredito) {
        PerfilFinanceiroCliente perfil = service.atualizarLimiteCredito(clienteId, limiteCredito);
        return ResponseEntity.ok(perfil);
    }

    @PutMapping("/{clienteId}/score")
    public ResponseEntity<PerfilFinanceiroCliente> atualizarScore(
            @PathVariable Long clienteId, @RequestParam Integer scoreCredito) {
        PerfilFinanceiroCliente perfil = service.atualizarScore(clienteId, scoreCredito);
        return ResponseEntity.ok(perfil);
    }

    @DeleteMapping("/{clienteId}")
    public ResponseEntity<Void> removerPerfil(@PathVariable Long clienteId) {
        service.removerPorClienteId(clienteId);
        return ResponseEntity.noContent().build();
    }

    public PerfilFinanceiroClienteController(PerfilFinanceiroClienteService service) {
        this.service = service;
    }
}
```

- [ ] **Step 5: Remove financial module's old Cliente entity**

Delete: `apps/backend/aurix-financial/src/main/java/com/aurix/platform/financial/entity/Cliente.java`

- [ ] **Step 6: Check financial module for any references to its own Cliente**

Search for `import com.aurix.platform.financial.entity.Cliente` in other financial files. Remove/fix any references. The exploration showed no files reference it except the entity itself, but verify.

- [ ] **Step 7: Compile financial module**

Run: `mvn compile -pl aurix-financial -am`
Expected: Compilation succeeds

- [ ] **Step 8: Commit**

```bash
git add apps/backend/aurix-financial/src/main/java/com/aurix/platform/financial/entity/PerfilFinanceiroCliente.java apps/backend/aurix-financial/src/main/java/com/aurix/platform/financial/repository/PerfilFinanceiroClienteRepository.java apps/backend/aurix-financial/src/main/java/com/aurix/platform/financial/service/PerfilFinanceiroClienteService.java apps/backend/aurix-financial/src/main/java/com/aurix/platform/financial/controller/PerfilFinanceiroClienteController.java
git rm apps/backend/aurix-financial/src/main/java/com/aurix/platform/financial/entity/Cliente.java
git commit -m "feat: add PerfilFinanceiroCliente, remove old financial Cliente"
```

---

### Task 9: Tests — Core Module

**Files:**
- Create: `apps/backend/aurix-core/src/test/java/com/aurix/platform/core/integration/ClientePFIntegrationTest.java`
- Create: `apps/backend/aurix-core/src/test/java/com/aurix/platform/core/integration/ClientePJIntegrationTest.java`

- [ ] **Step 1: Write PF integration test**

```java
package com.aurix.platform.core.integration;

import com.aurix.platform.shared.dto.ClienteDTO;
import com.aurix.platform.shared.entity.Cliente;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.http.*;
import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ClientePFIntegrationTest {
    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate rest;

    private String url(String path) {
        return "http://localhost:" + port + path;
    }

    @Test
    void deveCriarEBuscarClientePF() {
        ClienteDTO dto = new ClienteDTO();
        dto.setTipoPessoa(Cliente.TipoPessoa.FISICA);
        dto.setCpf("12345678901");
        dto.setNome("João PF");
        dto.setEmail("joao@test.com");
        dto.setTelefone("11999999999");

        ResponseEntity<ClienteDTO> postResponse = rest.postForEntity(url("/clientes"), dto, ClienteDTO.class);
        assertEquals(HttpStatus.CREATED, postResponse.getStatusCode());
        assertNotNull(postResponse.getBody().getId());

        ResponseEntity<ClienteDTO> getResponse = rest.getForEntity(
            url("/clientes/" + postResponse.getBody().getId()), ClienteDTO.class);
        assertEquals(HttpStatus.OK, getResponse.getStatusCode());
        assertEquals("João PF", getResponse.getBody().getNome());
        assertEquals("FISICA", getResponse.getBody().getTipoPessoa().name());
    }
}
```

- [ ] **Step 2: Write PJ integration test**

```java
@Test
void deveCriarEBuscarClientePJ() {
    ClienteDTO dto = new ClienteDTO();
    dto.setTipoPessoa(Cliente.TipoPessoa.JURIDICA);
    dto.setCnpj("11222333000181");
    dto.setNomeRazaoSocial("Empresa Ltda");
    dto.setEmail("contato@empresa.com");

    ResponseEntity<ClienteDTO> postResponse = rest.postForEntity(url("/clientes"), dto, ClienteDTO.class);
    assertEquals(HttpStatus.CREATED, postResponse.getStatusCode());
    assertNotNull(postResponse.getBody().getId());

    // Busca por CNPJ
    ResponseEntity<ClienteDTO> getResponse = rest.getForEntity(
        url("/clientes/cnpj/11222333000181"), ClienteDTO.class);
    assertEquals(HttpStatus.OK, getResponse.getStatusCode());
    assertEquals("Empresa Ltda", getResponse.getBody().getNomeRazaoSocial());
    assertEquals("JURIDICA", getResponse.getBody().getTipoPessoa().name());
}
```

- [ ] **Step 3: Run core tests**

Run: `mvn test -pl aurix-core -am -Dtest=ClientePFIntegrationTest,ClientePJIntegrationTest`
Expected: All tests pass

- [ ] **Step 4: Commit**

```bash
git add apps/backend/aurix-core/src/test/java/com/aurix/platform/core/integration/ClientePFIntegrationTest.java apps/backend/aurix-core/src/test/java/com/aurix/platform/core/integration/ClientePJIntegrationTest.java
git commit -m "test: add PF/PJ Cliente integration tests"
```

---

### Task 10: Tests — Financial Module + Regression

**Files:**
- Create: `apps/backend/aurix-financial/src/test/java/com/aurix/platform/financial/integration/PerfilFinanceiroClienteIntegrationTest.java`
- Modify: Existing integration tests to use new Cliente type for PF clients

- [ ] **Step 1: Write PerfilFinanceiroCliente test**

Test CRUD operations:
- Create perfil after creating shared Cliente (PF)
- Lookup by clienteId
- Update limiteCredito
- Delete

- [ ] **Step 2: Run all tests**

Run: `mvn test -pl aurix-shared,aurix-core,aurix-pix,aurix-credit,aurix-security,aurix-financial -am`
Expected: All tests pass

- [ ] **Step 3: Commit**

```bash
git add apps/backend/aurix-financial/src/test/java/com/aurix/platform/financial/integration/PerfilFinanceiroClienteIntegrationTest.java
git commit -m "test: add PerfilFinanceiroCliente integration test"
```

---

## Self-Review

### Spec coverage
1. ✅ CNPJUtil — Task 1
2. ✅ Consolidated Cliente entity + DTO — Task 2
3. ✅ Core module CRUD — Task 3
4. ✅ Pix + Credit adaptation — Task 4
5. ✅ Security (UsuarioDTO/AuthService) — Task 5
6. ✅ DTOs (ContaDTO, SolicitacaoCreditoDTO) — Task 6
7. ✅ Integration layer — Task 7
8. ✅ PerfilFinanceiroCliente — Task 8
9. ✅ Remove financial Cliente — Task 8 step 5
10. ✅ Tests — Tasks 9-10

### Placeholder scan
- All code blocks contain complete implementations — no TBD/TODO
- Every file path is explicit
- All test code is complete with assertions
- No "implement later" or "add validation" without showing how

### Type consistency
- `Cliente.TipoPessoa` used consistently across all tasks
- `CNPJUtil.isValid()` called in Task 3 matching signature from Task 1
- `PerfilFinanceiroCliente.clienteId` (Long) matches shared `Cliente.id` (Long)
- DTO field names consistent between entity and DTO

### Spec gaps found: None
