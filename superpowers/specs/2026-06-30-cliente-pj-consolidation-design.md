# Cliente PF/PJ Consolidation — Design Spec

## Goal

Consolidate the two parallel `Cliente` entities (shared PF-only + financial PF/PJ) into a single `Cliente` in `aureus-shared` that supports both **Pessoa Física** and **Pessoa Jurídica**, with financial-specific attributes moved to a separate profile entity.

## Architecture

### Before

```
aureus-shared/entity/Cliente         ← PF only (cpf, nome, email...)
aureus-financial/entity/Cliente      ← PF/PJ (TipoPessoa, cpf, cnpj, razaoSocial, limiteCredito...)
```

### After

```
aureus-shared/entity/Cliente         ← PF + PJ (tipoPessoa, cpf OR cnpj)
aureus-financial/entity/PerfilFinanceiroCliente  ← finance-specific attrs, FK → shared Cliente
```

## 1. Consolidated Cliente Entity (aureus-shared)

Package: `com.aureus.platform.shared.entity`
Table: `aureus.clientes`

```
Cliente extends BaseEntity (id, tenantId, dataCriacao, dataAtualizacao, versao)
├── tipoPessoa: enum [FISICA, JURIDICA]
├── PF-only:
│   ├── cpf: String (11 digits, @Pattern, required if FISICA)
│   ├── nome: String (required if FISICA)
│   └── dataNascimento: LocalDate (optional)
├── PJ-only:
│   ├── cnpj: String (14 digits, @Pattern, required if JURIDICA)
│   ├── nomeRazaoSocial: String (required if JURIDICA)
│   ├── nomeFantasia: String (optional)
│   ├── inscricaoEstadual: String (optional)
│   └── inscricaoMunicipal: String (optional)
├── Compartilhado:
│   ├── email: String (@Email)
│   ├── telefone: String
│   ├── endereco: String (JSONB)
│   ├── cidade: String
│   ├── estado: String
│   ├── cep: String
│   └── contato: String (optional)
└── status: StatusCliente [ATIVO, INATIVO, BLOQUEADO, SUSPENSO]
```

**Unique constraints:** `(tenantId, cpf)`, `(tenantId, cnpj)`, `(tenantId, email)`

**Validation rules:**
- `tipoPessoa == FISICA` → `cpf` required, `cnpj` must be null, `nome` required
- `tipoPessoa == JURIDICA` → `cnpj` required, `cpf` must be null, `nomeRazaoSocial` required
- `email` required for both
- CPF validation: `CPFUtil.isValid()` (existing)
- CNPJ validation: `CNPJUtil.isValid()` (new, same pattern as CPFUtil)

## 2. ClienteDTO (aureus-shared)

Updated to mirror the new entity fields. Replaces `cpf`-only with both `cpf` and `cnpj`, adds `tipoPessoa`.

## 3. PerfilFinanceiroCliente (aureus-financial)

Package: `com.aureus.platform.financial.entity`
Table: `perfis_financeiros_clientes`

```
PerfilFinanceiroCliente
├── id: Long (PK)
├── clienteId: Long (FK → shared Cliente.id, unique)
├── codigoCliente: String (internal code)
├── limiteCredito: BigDecimal
├── scoreCredito: Integer
├── observacoes: String
└── metadata: String (JSONB)
```

## 4. Module Changes

### aureus-shared
- Replace `Cliente.java` entity with consolidated version
- Update `ClienteDTO.java` with PF/PJ fields
- Add `CNPJUtil.java` (mirrors CPFUtil pattern)
- Add `ClienteNaoEncontradoException` overload by CNPJ
- Update `IntegrationController`, `IntegrationService`, `SharedCacheService` to handle PF/PJ

### aureus-core
- `ClienteService`: validate CPF or CNPJ based on `tipoPessoa`
- `ClienteRepository`: add `findByCnpj`, `existsByCnpj`, `findByTenantIdAndCnpj`
- `ClienteController`: add `GET /clientes/cnpj/{cnpj}`, expand `POST /` to accept `tipoPessoa`
- `ContaService`: handle PJ client in account creation

### aureus-pix
- `ClienteRepository`: update queries (PIX chave can be PF or PJ)

### aureus-credit
- `ClienteRepository`: already generic (no PF-specific queries)
- `SolicitacaoCreditoService`: adapt `clienteNome` to show nome or razaoSocial

### aureus-security
- `AuthService`: `clienteCpf` → `clienteDocumento` (CPF or CNPJ)
- `UsuarioDTO`: add `clienteDocumento`, `clienteTipoPessoa`; keep `clienteCpf` as deprecated alias

### aureus-financial
- Remove own `Cliente` entity, repository, service, controller
- Create `PerfilFinanceiroCliente` entity, repository, service, controller
- `POST /api/financial/perfil/{clienteId}` — create financial profile
- `GET /api/financial/perfil/{clienteId}` — get financial profile
- `PUT /api/financial/perfil/{clienteId}` — update financial attributes

### aureus-onboarding
- Expand `ReceitaFederalStub` to validate CNPJ (already stubbed)

### aureus-organization
- Add `clienteId` FK to `Empresa` for PJ → Cliente relationship (optional, for future)

### aureus-cambio
- `ClienteCambio` already references `clienteId` (Long) — compatible as-is

### DTOs across system
- `ContaDTO.clienteNome` → can be `nome` (PF) or `nomeRazaoSocial` (PJ)
- `SolicitacaoCreditoDTO.clienteNome` → same
- `UsuarioDTO.clienteCpf` → add `clienteDocumento`, deprecate `clienteCpf`

## 5. API Changes

### New/Modified Endpoints

| Method | Path | Change |
|--------|------|--------|
| POST | `/api/clientes` | Accept `tipoPessoa`, validate PF/PJ fields accordingly |
| GET | `/api/clientes/cnpj/{cnpj}` | New — busca por CNPJ |
| GET | `/api/clientes/cpf/{cpf}` | Unchanged (works for PF) |
| POST | `/api/financial/perfil/{clienteId}` | New — create financial profile |
| GET | `/api/financial/perfil/{clienteId}` | New — get financial profile |
| PUT | `/api/financial/perfil/{clienteId}` | New — update financial profile |

## 6. Tests

| Test | Module | Scope |
|------|--------|-------|
| `ClienteValidationTest` | shared | PF requires CPF, PJ requires CNPJ, cross-validation |
| `ClienteRepositoryTest` | core | Find by CPF, CNPJ, unique constraints |
| `ClienteServiceTest` | core | CRUD PF, CRUD PJ, document uniqueness |
| `ClienteControllerTest` | core | POST/GET with tipoPessoa, CNPJ lookup |
| `PerfilFinanceiroTest` | financial | CRUD, FK constraint |
| `CoreFlowIntegrationTest` | core | Existing PF flows still work |
| `CreditFlowIntegrationTest` | credit | PF credit flow unchanged |
| `PixFlowIntegrationTest` | pix | PF PIX flow unchanged |
| `AuthServiceTest` | security | documento field migration |

## 7. Migration (existing data)

Not covered in this spec — assumed zero production data or schema reset (dev phase). If needed later, a Flyway migration script splits financial `Cliente` rows into shared `clientes` + `perfis_financeiros_clientes`.

## 8. Non-goals

- Data migration from existing financial Cliente table (out of scope)
- Relationship between `Empresa` and `ClientePJ` (deferred to Conta Empresarial sub-project)
- Credit analysis / scoring logic (stays in financial module)
- B2B product catalog (deferred to next sub-project)
