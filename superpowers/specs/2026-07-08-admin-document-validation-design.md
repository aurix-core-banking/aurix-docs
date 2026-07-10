# Admin Document Validation Feature Design

> **Goal:** Allow back-office analysts to individually validate or reject uploaded onboarding documents with an observation comment.

## Backend

### New endpoint (both PF and PJ)

`POST /contas/pf/solicitacoes/{id}/documentos/{documentoId}/validar`

**Request body:**
```json
{
  "validado": true,
  "observacao": "Documento confere com registro original"
}
```

**Behavior:**
- Finds `DocumentoOnboarding` by `documentoId`
- Validates the document belongs to the given solicitation
- Sets `documento.validado = request.validado`
- Sets `documento.observacaoValidacao = request.observacao`
- Saves
- Returns 204 No Content

**PJ equivalent:** `POST /contas/pj/{id}/documentos/{documentoId}/validar` (same logic)

### Service method

`OnboardingPFService.validarDocumento(Long solicitacaoId, Long documentoId, boolean validado, String observacao)`

Simple: lookup doc, verify it belongs to the solicitation, update fields, save.

PJ service gets the same method.

### Controller changes

Both `ControllerPF` and `ControllerPJ` get one new endpoint each.

## Frontend

### DocumentList component changes

Each row in the document table currently shows: ID, Tipo, Arquivo, Validado (chip).

Add a new column with a "Validar" icon button that opens a dialog:

**Dialog:**
- Title: "Validar Documento"
- Shows: document type, file name
- Radio/switch: "Aprovar" / "Rejeitar"
- TextField: "Observação" (multiline, 3 rows, optional)
- Buttons: Cancelar, Confirmar
- On confirm: POST to validation endpoint
- On success: snackbar "Documento validado com sucesso", refresh list

The button should be disabled if `doc.validado` is already `true` (already validated).

### API call

Uses `fetchUtils.fetchJson` and `getResourceUrl` (for base URL building).

The URL pattern: `${getResourceUrl('solicitacoes_conta', solicitacaoId)}/documentos/${documentoId}/validar`

## Data Flow

```
Admin (DocumentList)                  Backend
    |                                     |
    | POST .../documentos/{id}/validar -->|
    | { validado, observacao }            |
    |<-- 204 No Content                   |
    |                                     |
    | refresh() --> get solicitation      |
    |<-- documentos atualizados           |
```
