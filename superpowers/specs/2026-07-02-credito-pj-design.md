# Crédito PJ — Design Spec

## Summary

Extend the existing `aurix-credit` module to support business credit (Crédito PJ) by adding CNPJ bureau scoring, PJ-specific decision rules, PJ eligibility rules in the catalog module, and Limite Rotativo via the business account's `limiteCredito`. Reuses the existing `SolicitacaoCredito` entity and workflow for loan-type products (Capital de Giro).

## Architecture

- **Existing `SolicitacaoCredito` entity** handles PJ credit applications via `cliente` (ManyToOne → Cliente, supports both PF/PJ) + `dadosAdicionais` (JSONB) for PJ metadata
- **New `CreditBureauService.consultarScoreCNPJ()`** for CNPJ-based scoring
- **Extended `DecisaoCreditoService`** with PJ-specific decision rules (score thresholds, faturamento check)
- **Extended `ElegibilidadeService`** with PJ field extractors (FATURAMENTO, CNAE, TEMPO_EXISTENCIA, PORTE, CAPITAL_SOCIAL)
- **Extended `ConfigCredito`** with PJ-specific configuration fields
- **Limite Rotativo** uses `Conta.limiteCredito` (already exists) with a new approval flow

## Scope

### 1. Product Types

- Add `LIMITE_ROTATIVO` to `ProdutoCredito.TipoCredito` enum
- `CAPITAL_GIRO` already exists

### 2. CreditBureauService — CNPJ scoring

New method:
```java
ScoreCNPJResult consultarScoreCNPJ(String cnpj);

record ScoreCNPJResult(int score, BigDecimal faturamentoEstimado,
    RiscoCNPJ risco, String mensagem) {
    enum RiscoCNPJ { BAIXO, MEDIO, ALTO }
}
```

Stub implementation: score = 750, faturamento = R$ 500k, risco = MEDIO for valid CNPJs.

### 3. DecisaoCreditoService — PJ rules

When `cliente.tipoPessoa == JURIDICA`:
- Call `consultarScoreCNPJ()` instead of `consultarScore()`
- Score thresholds: approve ≥ 500, reject ≤ 300, refer 301-499
- Capacity check: `faturamentoEstimado >= valorSolicitado * 0.3`
- Check `cliente.faturamentoMensal` as additional capacity signal
- Store PJ risk analysis in `analiseRisco` JSONB field

### 3a. Cliente entity — add PJ financial fields

The credit module needs access to PJ financial data during decision/eligibility. Rather than coupling the credit module to the onboarding module's entities, add key financial fields to `Cliente` in `aurix-shared`:

- `faturamentoMensal` (BigDecimal)
- `capitalSocial` (BigDecimal)
- `cnaePrincipal` (String)
- `porte` (String — values: MEI, ME, EPP, DEMAIS)
- `dataConstituicao` (LocalDate)
- `tempoConstituicaoMeses` (Integer, computed on save)

These are nullable and only populated for PJ clients. The `OnboardingPJService` copies these from `SolicitacaoPJ`/`Empresa` when creating the cliente via `CoreApiClient.criarClientePJeConta()`, which sends the data to the core API's ClienteController.

### 4. ElegibilidadeService — PJ field extractors

| Campo | Source |
|-------|--------|
| `FATURAMENTO_MENSAL` | `Cliente.faturamentoMensal` |
| `CAPITAL_SOCIAL` | `Cliente.capitalSocial` |
| `CNAE_PRINCIPAL` | `Cliente.cnaePrincipal` |
| `TEMPO_EXISTENCIA_MESES` | `Cliente.dataConstituicao` |
| `PORTE_EMPRESA` | `Cliente.porte` |

### 5. ConfigCredito — PJ fields

Add to `ConfigCredito` entity:
- `faturamentoMinimo` (BigDecimal)
- `tempoConstituicaoMinimoMeses` (Integer)
- `cnaePermitidos` (String, JSON array stored as TEXT)

### 6. Limite Rotativo flow

- Product type `LIMITE_ROTATIVO` creates a solicitação like any other credit
- After approval, `POST /solicitacoes/{id}/liberar-limite` endpoint:
  - Validates the solicitação is APROVADA with tipo = LIMITE_ROTATIVO
  - Calls `ContaService.atualizarLimiteCredito(contaId, valorAprovado)` via admin API
  - Updates solicitação status to LIBERADO
  - The business account's `limiteCredito` is updated

### 7. Tests

- Integration test: create PJ client → create CAPITAL_GIRO solicitação → analyze → approve → verify
- Integration test: create LIMITE_ROTATIVO → approve → liberar limite → verify limiteCredito updated
- Unit tests for PJ decision rules
- All existing credit tests must pass

### 3b. Onboarding → Cliente financial data sync

Update `CoreApiClient.criarClientePJeConta()` to send PJ financial data to the core API:

```
POST /api/core/clientes → {
  tipoPessoa: JURIDICA, cnpj, nomeRazaoSocial, email, telefone, endereco,
  faturamentoMensal, capitalSocial, cnaePrincipal, porte, dataConstituicao
}
```

Update `ClienteController` + `ClienteService.criarCliente()` in `aurix-core` to accept and persist these new fields.

### 3c. OnboardingPJService update

After `consultarCNPJ()` creates the `Empresa` entity, and before calling `CoreApiClient.criarClientePJeConta()`, populate `Cliente`'s financial fields from `SolicitacaoPJ` data.

## Out of Scope (Phase 2)

- Multi-level credit committee/review
- Collateral/guarantee tracking for PJ
- Balance sheet analysis (beyond faturamento)
- CNAE-based risk rating
- Repayment schedule entity for general credit
- Performance/BACEN reporting
