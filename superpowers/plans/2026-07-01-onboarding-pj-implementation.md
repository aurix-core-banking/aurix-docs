# Onboarding PJ Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Expand `aureus-onboarding` module with PJ onboarding flow — separate entity/controller/service, reusing shared core (Documento, HistoricoAprovacao, PEP, integrações stub).

**Architecture:** Three-layer split: SolicitacaoOnboarding (protocolo comum) + SolicitacaoPF (refatorado do existente) + SolicitacaoPJ (novo com Empresa + Participantes). Núcleo compartilhado: DocumentoService, WorkflowEngine, CoreApiClient.

**Tech Stack:** Java 25, Spring Boot 4.1.0, Spring Data JPA, H2 (test), PostgreSQL (prod), builder pattern (delombok).

## Global Constraints

- No Lombok — existing code uses builder pattern via delombok (`SolicitacaoConta.builder()`)
- Follow existing patterns: `@CreationTimestamp`/`@UpdateTimestamp` on entities, builder pattern, `SolicitacaoContaResponse.from(entity)` for DTO conversion
- Schema: `aureus`
- New tables: `solicitacoes_pj`, `empresas`, `participantes`, `solicitacoes_pf`
- Existing `solicitacoes_conta` → rename to `solicitacoes_onboarding`
- Test pattern: `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `@MockitoBean` for `CoreApiClient`
- Keep backward compat: existing PF endpoints continue at `/onboarding/contas/pf`
- New PJ endpoints: `/onboarding/contas/pj`

---

### Task 1: Refactor SolicitacaoConta → SolicitacaoOnboarding + SolicitacaoPF

**Files:**
- Create: `entity/SolicitacaoOnboarding.java` (núcleo comum)
- Create: `entity/SolicitacaoPF.java` (dados específicos PF)
- Create: `entity/TipoPessoa.java` (enum: FISICA, JURIDICA)
- Delete: `entity/SolicitacaoConta.java`
- Modify: `entity/DocumentoOnboarding.java` (change `@ManyToOne SolicitacaoConta` → `SolicitacaoOnboarding`)
- Modify: `entity/HistoricoAprovacao.java` (same FK change)
- Modify: `repository/SolicitacaoContaRepository.java` → rename to `SolicitacaoOnboardingRepository.java`
- Create: `repository/SolicitacaoPFRepository.java`
- Modify: `dto/SolicitacaoContaRequest.java` → add `tipoPessoa`, keep PF fields
- Modify: `dto/SolicitacaoContaResponse.java` → adapt to new entity structure

**Entity: SolicitacaoOnboarding.java**
```java
@Entity
@Table(name = "solicitacoes_onboarding", schema = "aureus")
public class SolicitacaoOnboarding {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(name = "tenant_id", length = 64)
    private String tenantId;
    @Enumerated(EnumType.STRING)
    @Column(name = "tipo_pessoa", nullable = false, length = 10)
    private TipoPessoa tipoPessoa;
    @Column(name = "status", nullable = false, length = 30)
    private String status;  // workflow-specific status string
    @Column(name = "canal", length = 20)
    private String canal;
    @Column(name = "produto", length = 50)
    private String produto;
    @OneToMany(mappedBy = "solicitacao", cascade = ALL, orphanRemoval = true)
    private List<DocumentoOnboarding> documentos;
    @OneToMany(mappedBy = "solicitacao", cascade = ALL, orphanRemoval = true)
    @OrderBy("dataAcao DESC")
    private List<HistoricoAprovacao> historico;
    @CreationTimestamp @Column(name = "data_criacao", nullable = false, updatable = false)
    private LocalDateTime dataCriacao;
    @UpdateTimestamp @Column(name = "data_atualizacao", nullable = false)
    private LocalDateTime dataAtualizacao;
    // builder, getters, setters, equals, hashCode
}
```

**Entity: SolicitacaoPF.java**
```java
@Entity
@Table(name = "solicitacoes_pf", schema = "aureus")
public class SolicitacaoPF {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(name = "solicitacao_id", nullable = false, unique = true)
    private Long solicitacaoId;  // FK → SolicitacaoOnboarding.id
    @NotBlank @Pattern(regexp = "\\d{11}")
    @Column(name = "cpf", nullable = false, length = 11)
    private String cpf;
    @NotBlank @Column(name = "nome", nullable = false, length = 255)
    private String nome;
    @Column(name = "data_nascimento")
    private LocalDate dataNascimento;
    @Column(name = "ocupacao", length = 100)
    private String ocupacao;
    @Column(name = "renda_declarada", precision = 15, scale = 2)
    private BigDecimal rendaDeclarada;
    @Column(name = "pep")
    private Boolean pep;
    @Column(name = "score_bureau")
    private Integer scoreBureau;
    @Column(name = "resultado_kyc", length = 50)
    private String resultadoKyc;
    @Column(name = "conta_limitada_ate_kyc")
    private Boolean contaLimitadaAteKyc;
    // builder, getters, setters
}
```

**TipoPessoa.java:**
```java
public enum TipoPessoa { FISICA, JURIDICA }
```

**Update DocumentoOnboarding:** Change `@ManyToOne` type from `SolicitacaoConta` to `SolicitacaoOnboarding`.

**Update HistoricoAprovacao:** Same FK type change.

**Migration DTO:** `SolicitacaoContaRequest` keeps PF fields + adds `tipoPessoa` (default FISICA).

**Compile verify:** `mvn compile -pl aureus-onboarding -am`

**Commit:** `feat(onboarding): split SolicitacaoConta into shared SolicitacaoOnboarding + SolicitacaoPF`

---

### Task 2: Refactor PF Service + Controller + Tests

**Files:**
- Modify: `service/OnboardingService.java` → adapt to new entity structure, rename to `OnboardingPFService.java`
- Modify: `controller/OnboardingController.java` → refactor to `ControllerPF.java` at `/onboarding/contas/pf`
- Modify: `client/CoreApiClient.java` → update method signatures (accept SolicitacaoOnboarding + SolicitacaoPF)
- Modify: `test/.../OnboardingFlowIntegrationTest.java` → adapt tests to new structure
- Create: `service/WorkflowEngine.java` (shared interface + PF implementation)

**OnboardingPFService:** Same logic but operates on `SolicitacaoOnboarding` + `SolicitacaoPF` entities. Create both in `solicitarAberturaConta`. On approve, call `CoreApiClient.criarClientePFeConta`.

**ControllerPF:** Keep all existing endpoints at `/onboarding/contas/pf`. Use `OnboardingPFService`.

**WorkflowEngine (shared):**
```java
public interface WorkflowEngine {
    String getTipo();
    List<String> getTransicoesValidas(String statusAtual, String novoStatus);
    boolean transicaoValida(String statusAtual, String novoStatus);
    String getStatusInicial();
}
```

Create a simple implementation `WorkflowPF` with the existing status machine.

**Test:** All 14 existing tests must still pass after refactoring.

**Commit:** `refactor(onboarding): adapt PF service/controller to new entity structure`

---

### Task 3: Expand CoreApiClient for PJ

**Files:**
- Modify: `client/CoreApiClient.java`

Add new method:
```java
public CriarClienteContaResult criarClientePJeConta(
    String tenantId, String cnpj, String razaoSocial,
    String email, String telefone, String endereco,
    boolean contaLimitada)
```

This calls:
```
POST /api/core/clientes → {tipoPessoa: JURIDICA, cnpj, nomeRazaoSocial: razaoSocial, email, telefone}
POST /api/core/contas → {clienteId, tipoConta: CORRENTE, saldo: 0, limiteCredito: 0}
```

The Cliente entity already supports tipoPessoa=JURIDICA with cnpj+nomeRazaoSocial (from Sub-project 1).

**Commit:** `feat(onboarding): add PJ support to CoreApiClient`

---

### Task 4: Create PJ Entities

**Files:**
- Create: `entity/SolicitacaoPJ.java`
- Create: `entity/Empresa.java`
- Create: `entity/Participante.java`
- Create: `entity/PorteEmpresa.java` (enum: MEI, ME, EPP, DEMAIS)
- Create: `entity/SituacaoCNPJ.java` (enum: ATIVA, INAPTA, BAIXADA, SUSPENSA, NULA)
- Create: `entity/TipoParticipante.java` (enum: SOCIO, ADMINISTRADOR, REPRESENTANTE, PROCURADOR, BENEFICIARIO_FINAL)
- Create: `repository/SolicitacaoPJRepository.java`
- Create: `repository/EmpresaRepository.java`
- Create: `repository/ParticipanteRepository.java`

**SolicitacaoPJ** (per spec: cnpj, razaoSocial, nomeFantasia, naturezaJuridica, porte, capitalSocial, dataConstituicao, inscricaoEstadual, inscricaoMunicipal, faturamentoMensal, numeroFuncionarios, clienteIdCriado, contaIdCriada, observacoesAnalista). FK: solicitacaoId → SolicitacaoOnboarding.id (unique).

**Empresa** (per spec: cnpj, razaoSocial, nomeFantasia, cnaePrincipal, cnaeSecundarios JSONB, endereco JSONB, situacaoCadastral, dataSituacao, regimeTributario, dadosAbertos JSONB). FK: solicitacaoId → SolicitacaoOnboarding.id (unique).

**Participante** (per spec: solicitacaoId, tipo, cpf, nome, email, telefone, dataNascimento, nacionalidade, qualificacao, percentualParticipacao, documentos @OneToMany, validado).

All entities use builder pattern (delombok) and `@CreationTimestamp`/`@UpdateTimestamp`.

**Commit:** `feat(onboarding): add PJ entities (SolicitacaoPJ, Empresa, Participante)`

---

### Task 5: Create OnboardingPJService + WorkflowPJ

**Files:**
- Create: `service/OnboardingPJService.java`
- Create: `service/WorkflowPJ.java`

**WorkflowPJ** implements `WorkflowEngine`:
- tipo: "PJ"
- status inicial: "EM_PREENCHIMENTO"
- transições: EM_PREENCHIMENTO → CNPJ_CONSULTADO → SOCIOS_VALIDADOS → DOCUMENTOS_ANALISADOS → AML_APROVADO → COMPLIANCE_APROVADO → EM_ASSINATURA → CONTRATO_ASSINADO → CONTA_CRIADA
- Qualquer status → REJEITADA

**OnboardingPJService:**
- `iniciarOnboarding(SolicitacaoPJRequest)` — cria SolicitacaoOnboarding(tipoPessoa=JURIDICA, status=EM_PREENCHIMENTO) + SolicitacaoPJ
- `consultarCNPJ(solicitacaoId)` — chama ReceitaFederalService.consultarCnpj, cria/atualiza Empresa, transiciona para CNPJ_CONSULTADO. Se CNPJ inválido/inativo → REJEITADA.
- `adicionarParticipante(solicitacaoId, dadosParticipante)` — adiciona participante, transiciona para SOCIOS_VALIDADOS (se ao menos 1 sócio adicionado)
- `adicionarDocumento(solicitacaoId, tipo, nome, url)` — reusa DocumentoService
- `aprovar(solicitacaoId, usuario, obs)` — cria Cliente PJ via CoreApiClient, cria Conta, transiciona para CONTA_CRIADA
- `rejeitar(solicitacaoId, usuario, obs)` — transiciona para REJEITADA

**Status transitions use WorkflowPJ** to validate.

**Tests:** `OnboardingPJServiceTest` — mock repositories, test each transition and CRUD operation.

**Commit:** `feat(onboarding): add OnboardingPJService + WorkflowPJ`

---

### Task 6: Create ControllerPJ + DTOs

**Files:**
- Create: `controller/ControllerPJ.java`
- Create: `dto/SolicitacaoPJRequest.java`
- Create: `dto/SolicitacaoPJResponse.java`
- Create: `dto/ParticipanteRequest.java`

**ControllerPJ:** `@RestController @RequestMapping("/api/onboarding/contas/pj")`

| Method | Path | Description |
|--------|------|-------------|
| POST | `/` | Iniciar onboarding PJ |
| GET | `/{id}` | Consultar status |
| GET | `/` | Listar (back office) |
| POST | `/{id}/socios` | Adicionar sócio |
| DELETE | `/{id}/socios/{participanteId}` | Remover |
| POST | `/{id}/documentos` | Adicionar documento |
| POST | `/{id}/validar-cnpj` | Consultar Receita |
| POST | `/{id}/aprovar` | Aprovar |
| POST | `/{id}/rejeitar` | Rejeitar |
| GET | `/{id}/socios` | Listar participantes |

**DTOs:** Simple Java records or classes with constructor/accessors.

**Tests:** `ControllerPJTest` — MockMvc test for each endpoint.

**Commit:** `feat(onboarding): add ControllerPJ + DTOs for PJ onboarding flow`

---

### Task 7: Integration Tests + Regression

**Files:**
- Create: `test/.../integration/OnboardingPJFlowIntegrationTest.java`
- Modify: `test/.../integration/OnboardingFlowIntegrationTest.java` (adapt to new PF endpoints)

**OnboardingPJFlowIntegrationTest:**
- `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `@MockitoBean` for `CoreApiClient`
- Tests: iniciar onboarding PJ, consultar status, validar CNPJ (stub), adicionar sócio, adicionar documento, aprovar, verificar cliente criado no core

**Regression:** Update existing PF integration test to use new endpoints path `/onboarding/contas/pf`.

```bash
mvn test -pl aureus-onboarding -am -DfailIfNoTests=false
```

**Commit:** `test(onboarding): add PJ flow integration test + regression`
