# Onboarding Admin Dashboard + Bulk Actions + Mobile Upload Design

> **Goal:** Add onboarding metrics cards/chart to the existing admin Dashboard, bulk approve/reject actions on List pages, and document upload step to mobile onboarding.

## Section 1 — Admin Dashboard: Onboarding Metrics

### Location
Integrated into `Dashboard.js` — new section below existing stat cards, alongside existing charts.

### Stat Cards (new row, 4 columns)
| Card | Data Source | Calculation |
|------|-------------|-------------|
| Solicitações PF Pendentes | `GET /contas/pf/solicitacoes?status=RECEBIDA,DOCUMENTOS_PENDENTES,EM_ANALISE_KYC` | Count of results |
| Solicitações PJ Pendentes | `GET /contas/pj?status=RECEBIDA,EM_PREENCHIMENTO,...,EM_ASSINATURA` | Count of results |
| Taxa Aprovação PF | `GET /contas/pf/solicitacoes` (all) | `(APROVADA+CONTA_CRIADA) / (total - CANCELADA) * 100` |
| Taxa Aprovação PJ | `GET /contas/pj` (all) | Same calculation |

### Chart: Onboarding Funnel
Horizontal BarChart showing count per status. Two series (PF + PJ) side by side.

Statuses: Recebida → Documentos Pendentes → Em Análise KYC → KYC Aprovado → Aprovada → Conta Criada

### Backend
No new backend endpoint needed — existing `GET .../solicitacoes` and `GET .../pj` already support `?status=` filter and return all records. The `Dashboard.js` will use `useGetList` with `filter` params (but React Admin's `useGetList` with `perPage=1` currently only gets totals via `content-range` header). Since the existing endpoints don't return `content-range`, we need a lightweight count approach.

**Alternative:** Use `useGetList` with `perPage=0` or a reasonable limit, or use `fetchUtils` directly for counting.

**Simpler approach:** Fetch full list once (max 1000 per page) and compute counts client-side. Since onboarding lists are unlikely to exceed a few hundred records, this is acceptable.

## Section 2 — Admin Bulk Approve/Reject

### Target Pages
- `SolicitacaoContaList` (PF onboarding)
- `SolicitacaoPJList` (PJ onboarding)

### Behavior
- Checkbox selection in Datagrid (React Admin built-in)
- When N >= 1 selected, show toolbar with "Aprovar Selecionados" and "Rejeitar Selecionados"
- Confirmation dialog before executing
- Sequentially calls `POST /solicitacoes/{id}/aprovar?usuarioAnalista=admin` (or `/rejeitar`) for each ID
- Shows success/error snackbar per item
- Refreshes list on completion

### Rejection Specific
- Dialog includes `observacao` text field (multiline, required)
- Passes `&observacao=<encoded>` to the reject endpoint

## Section 3 — Mobile Document Upload (Base64)

### Overview
Two new screens that let the user capture/select photos via camera or gallery, convert to base64, and send to the existing `POST .../{id}/documentos` endpoint.

### PF Flow
`TipoSelector → FormPF → StepDocumentosPF → SuccessScreen`

### PJ Flow
`StepEmpresa → StepSocios → StepDocumentosPJ → SuccessScreen`

### StepDocumentosPF
- Title: "Envio de Documentos"
- Subtitle: "Anexe os documentos necessários para análise"
- Document list (required):
  - Documento de Identidade (RG/CNH)
  - CPF
  - Comprovante de Endereço
  - Comprovante de Renda
- Each item: document type label + capture button + image preview (if captured)
- "Capturar" button → opens `react-native-image-picker` camera
- "Galeria" button → opens gallery picker
- Image preview with "Remover" button
- "Enviar Documentos" button at bottom (disabled until all required docs captured)
- Upload progress per document
- On success → `SuccessScreen`
- Receives `solicitacaoId` via route params (passed from `FormPF` on successful submission)

### StepDocumentosPJ
- Title: "Envio de Documentos"
- Two sections:

**Seção 1: Documentos da Empresa**
- Contrato Social
- CNPJ (Cartão CNPJ)
- Balanço Patrimonial

**Seção 2: Documentos dos Sócios**
- For each partner from StepSocios: Nome do Sócio + sub-items:
  - Documento de Identidade
  - CPF

- Same capture/preview mechanism as PF
- "Enviar Todos" button at bottom
- Receives `solicitacaoId` + `socios` (list of partner names) via route params

### OnboardingService Change
Add method:
```js
uploadDocumento(solicitacaoId, tipo, tipoPessoa, base64, nomeArquivo)
```
- POST to `/onboarding/contas/pf/solicitacoes/{id}/documentos` (for PF) or `/onboarding/contas/pj/{id}/documentos` (for PJ)
- Body: `{ tipoDocumento, nomeArquivo, urlStorage: base64 }`
- `tipoPessoa` param to determine PF vs PJ URL

### Image Handling
- Use `react-native-image-picker` with options: `{ mediaType: 'photo', includeBase64: true, maxWidth: 1024, maxHeight: 1024, quality: 0.7 }`
- Store captured images as state: `{ [tipoDocumento]: { base64, uri, nomeArquivo } }`
- Generate `nomeArquivo` as `${tipo}_${timestamp}.jpg`

### Dependencies
- Add `react-native-image-picker` to package.json
- Update iOS Podfile for camera permissions (if iOS)
- Android camera permissions already handled in App.js

### Navigation Changes
- Add routes in `AuthNavigator.js`:
  - `StepDocumentosPF` → `StepDocumentosPF` screen
  - `StepDocumentosPJ` → `StepDocumentosPJ` screen
- Modify `FormPF.js`: after successful `criarSolicitacaoPF()`, navigate to `StepDocumentosPF` instead of `SuccessScreen`
- Modify `StepSocios.js`: after successful `criarSolicitacaoPJ()`, navigate to `StepDocumentosPJ` instead of `SuccessScreen`

## Data Flow

```
Mobile App                         Backend
    |                                 |
    |— POST /contas/pf/solicitacoes —>|  (FormPF)
    |<— { id, status } ————————-------|
    |                                 |
    |— POST .../{id}/documentos —---->|  (StepDocumentosPF, per document)
    |   { tipoDocumento, nomeArquivo, |
    |     urlStorage: base64 }        |
    |<— 204 No Content ——————---------|
    |                                 |
    |— NAVIGATE SuccessScreen —------>|
```
