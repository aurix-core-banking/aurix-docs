# Frontend — Onboarding PJ (Admin)

**Date:** 2026-07-03
**Status:** Approved

## Goal

Add PJ onboarding management to aurix-admin (back office), reusing existing patterns and adding reusable workflow components.

## Architecture

```
src/
├── components/
│   ├── WorkflowActions.js     # Action buttons conditional on status
│   ├── StatusTimeline.js      # Visual timeline of onboarding progress
│   ├── DocumentList.js        # Document table with upload/validation status
│   └── SocioList.js           # Participant table with add/remove
├── pages/
│   └── SolicitacoesPJ/
│       ├── index.js           # Barrel exports
│       ├── SolicitacaoPJList.js   # List with filters
│       └── SolicitacaoPJShow.js   # Detail with tabs + actions
└── config/
    └── resources.js           # + solicitacoes_pj resource mapping
```

## Resource Mapping

| Resource | Base | Path | Full URL |
|----------|------|------|----------|
| `solicitacoes_conta` | `/api/onboarding/onboarding` | `solicitacoes` | PF solicitations (existing) |
| `solicitacoes_pj` | `/api/onboarding/contas` | `pj` | PJ solicitations (new) |

New BASE entry:
```
onboarding_pj: '/api/onboarding/contas'
```

New RESOURCE_PATH entry:
```
solicitacoes_pj: { base: BASE.onboarding_pj, path: 'pj' }
```

This resolves to `http://localhost:8080/api/onboarding/contas/pj` for list and `http://localhost:8080/api/onboarding/contas/pj/{id}` for getOne.

## Components

### StatusTimeline

- **Props:** `historico: Array<{ acao, usuarioAnalista, dataAcao }>`, `statusAtual: string`
- **Render:** MUI Timeline (`@mui/lab`) with color-coded entries:
  - Concluído = green dot + check icon
  - Atual = blue dot
  - Pendente = grey dot (disabled)
  - Erro = red dot
- **Status mapping:** Maps `StatusOnboarding` enum values to human-readable labels

### DocumentList

- **Props:** `documentos: Array<{ id, tipoDocumento, nomeArquivo, validado }>`, `solicitacaoId: number`, `onUpload: (formData) => void`
- **Render:** MUI Table
- **Upload:** Paperclip button per row? or upload zone at bottom? → Single upload zone at top
- **Upload implementation:** `POST /api/onboarding/contas/pj/{id}/documentos` with `{ tipoDocumento, conteudo }`

### SocioList

- **Props:** `socios: Array<{ id, tipo, cpf, nome, email, telefone, percentualParticipacao }>`, `solicitacaoId: number`, `onRemove: (participanteId) => void`
- **Render:** MUI Table with remove button
- **Add modal:** Dialog with form fields: tipo (select: SOCIO/ADMINISTRADOR/REPRESENTANTE/PROCURADOR/BENEFICIARIO_FINAL), cpf, nome, email, telefone, percentualParticipacao
- **POST:** `POST /api/onboarding/contas/pj/{id}/socios`
- **DELETE:** `DELETE /api/onboarding/contas/pj/{id}/socios/{participanteId}`

### WorkflowActions

- **Props:** `solicitacaoId: number`, `statusAtual: string`, `onAction: (action: string, data?: object) => void`
- **Logic:** Conditional button rendering based on `statusAtual`:

| Status | Available Actions |
|--------|-------------------|
| `EM_PREENCHIMENTO` | Validar CNPJ |
| `CNPJ_CONSULTADO` | Aprovar AML, Rejeitar |
| `SOCIOS_VALIDADOS` | Aprovar AML, Rejeitar |
| `DOCUMENTOS_ANALISADOS` | Aprovar AML, Rejeitar |
| `AML_APROVADO` | Aprovar Compliance, Rejeitar |
| `COMPLIANCE_APROVADO` | Solicitar Assinatura, Rejeitar |
| `EM_ASSINATURA` | Confirmar Assinatura, Rejeitar |
| `RECEBIDA`, `DOCUMENTOS_PENDENTES`, `EM_ANALISE_KYC` | Rejeitar |

When multiple actions possible, show all applicable.

- **Implementations:**
  - `validar-cnpj` → `POST /.../{id}/validar-cnpj`
  - `aml-aprovar` → `POST /.../{id}/aml-aprovar`
  - `compliance-aprovar` → `POST /.../{id}/compliance-aprovar`
  - `assinatura-solicitar` → `POST /.../{id}/assinatura-solicitar` (requires tipoAssinatura input)
  - `assinatura-confirmar` → `POST /.../{id}/assinatura-confirmar`
  - `aprovar` → `POST /.../{id}/aprovar?usuarioAnalista=X&observacao=Y`
  - `rejeitar` → `POST /.../{id}/rejeitar?usuarioAnalista=X&observacao=Y` (requires observacao input)

- **UX:** Each action button shows a confirmation dialog before executing. On success → refresh show page data.

## Pages

### SolicitacaoPJList

```
Resource: solicitacoes_pj
Filters: status (multi-select), cnpj (text search)
Columns: ID, CNPJ, Razão Social, Status (colored chip), Criado em, ShowButton
Sort: Default DESC by id
Extras: ExportButton
```

### SolicitacaoPJShow

**Layout:** `<TabbedShowLayout>` with 5 tabs:

1. **Detalhes Tab:**
   - Fields: id, cnpj, razaoSocial, nomeFantasia, email, telefone, status, clienteIdCriado, contaIdCriada, dataCriacao, dataAtualizacao
   - Simple fields, no sub-records

2. **Socios Tab:**
   - `<SocioList>` component
   - "Adicionar Sócio" button at top
   - Remove button per row

3. **Documentos Tab:**
   - `<DocumentList>` component
   - Upload zone at top

4. **Histórico Tab:**
   - `<StatusTimeline>` component
   - Read-only log

5. **Ações Tab:**
   - `<WorkflowActions>` component
   - Observação text field + Rejeitar button
   - Shows current status prominently

## Data Flow

```
User clicks action → Confirmation dialog → POST to backend API
  → Success: refresh show page (refetch getOne)
  → Error: show error notification
```

Custom actions bypass react-admin's dataProvider and call the API directly via `fetchUtils.fetchJson` (like the dataProvider does) for POST endpoints that aren't standard CRUD.

## Files to Create/Modify

1. **CREATE** `src/components/StatusTimeline.js`
2. **CREATE** `src/components/DocumentList.js`
3. **CREATE** `src/components/SocioList.js`
4. **CREATE** `src/components/WorkflowActions.js`
5. **CREATE** `src/pages/SolicitacoesPJ/index.js`
6. **CREATE** `src/pages/SolicitacoesPJ/SolicitacaoPJList.js`
7. **CREATE** `src/pages/SolicitacoesPJ/SolicitacaoPJShow.js`
8. **MODIFY** `src/config/resources.js` — add `solicitacoes_pj` entry
9. **MODIFY** `src/App.js` — add `<Resource name="solicitacoes_pj">`

## Testing

- Components are simple enough that visual verification in the browser suffices
- No existing test framework for aurix-admin components (no Jest tests in admin)
- Manual testing: list loads, show tabs render, actions trigger correct POSTs

## Out of Scope

- Create/Edit pages (PJ onboarding is created by the client in aurix-web)
- PF onboarding enhancements (existing read-only List+Show stays as-is)
- aurix-web or aurix-mobile changes
