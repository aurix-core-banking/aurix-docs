# Conta Empresarial — Design Spec

## Summary

Add `EMPRESARIAL` as a new `TipoConta` value to the existing `Conta` entity, enabling business accounts for PJ clients with minimal code changes. Reuses the existing `Conta` entity, `ContaService`, `ContaController`, and `dadosExtras` (JSONB) for business metadata.

## Scope

- New `TipoConta.EMPRESARIAL` enum value
- Validation: only `TipoPessoa.JURIDICA` clients can open EMPRESARIAL accounts
- Populate `clienteTipoPessoa` on `ContaDTO` responses
- API spec update (`aureus-core.yaml`)
- Integration tests

## Out of Scope

- Multi-signer authority
- Dedicated business fee/tariff rules
- New module or entity
- Spending limits (use `limiteCredito` + `dadosExtras`)

## Changes

### 1. Conta.TipoConta enum

File: `backend/aureus-core/src/main/java/com/aureus/platform/core/entity/Conta.java`

Add `EMPRESARIAL("Conta Empresarial")` to the existing enum. The enum is embedded inside the Conta entity class.

### 2. ContaService.criarConta()

File: `backend/aureus-core/src/main/java/com/aureus/platform/core/service/ContaService.java`

- After loading the Cliente, validate: if `contaDTO.getTipoConta() == EMPRESARIAL`, cliente must be `TipoPessoa.JURIDICA`
- Populate `clienteTipoPessoa` on the response DTO: `dto.setClienteTipoPessoa(cliente.getTipoPessoa().name())`
- No other logic changes — the existing Conta entity handles EMPRESARIAL like any other tipoConta

### 3. API Spec

File: `aureus-api-specs/aureus-core.yaml`

- Add `EMPRESARIAL` to the `tipoConta` enum definition

### 4. Tests

File: `backend/aureus-core/src/test/java/.../integration/ClientePJIntegrationTest.java`

- Add test: create PJ client → create EMPRESARIAL account → verify success + tipoConta
- Add test: create PF client → attempt EMPRESARIAL account → verify rejection (400)
- All 79 existing onboarding tests must still pass

## Implementation Plan

1. Add EMPRESARIAL to TipoConta enum
2. Update ContaService validation + populate clienteTipoPessoa
3. Update API spec
4. Add integration tests
5. Verify all tests pass
