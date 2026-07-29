# Admin Document Validation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan.

**Goal:** Add per-document validation (approve/reject) with observation for onboarding documents.

**Architecture:** Backend adds one simple endpoint per controller (PF/PJ), frontend adds validation dialog to DocumentList component.

**Tech Stack:** Spring Boot 4.1 (Java 25), React Admin 4 + MUI 5

## Global Constraints

- Backend: existing patterns (Lombok `@Slf4j`, constructor injection, `TenantContext.getTenantId()`)
- Frontend: functional components, `export const`, Portuguese labels, `fetchUtils` + `getResourceUrl` from `config/resources.js`
- Endpoint body as JSON (not form params)

---

### Task 1: Backend — Document Validation Endpoint (PF + PJ)

**Files:**
- Modify: `apps/backend/aurix-onboarding/src/main/java/com/aurix/platform/onboarding/controller/ControllerPF.java`
- Modify: `apps/backend/aurix-onboarding/src/main/java/com/aurix/platform/onboarding/controller/ControllerPJ.java`
- Modify: `apps/backend/aurix-onboarding/src/main/java/com/aurix/platform/onboarding/service/OnboardingPFService.java`
- Modify: `apps/backend/aurix-onboarding/src/main/java/com/aurix/platform/onboarding/service/OnboardingPJService.java`
- Modify: `apps/backend/aurix-onboarding/src/main/java/com/aurix/platform/onboarding/repository/DocumentoOnboardingRepository.java`

- [ ] **Step 1: Add `validarDocumento` method to OnboardingPFService**

Read existing `OnboardingPFService.java`. Add method:

```java
public void validarDocumento(Long solicitacaoId, Long documentoId, boolean validado, String observacao) {
    String tenantId = TenantContext.getTenantId();
    SolicitacaoOnboarding onboarding = solicitacaoOnboardingRepository.findByTenantIdAndId(tenantId, solicitacaoId)
        .orElseThrow(() -> new IllegalArgumentException("Solicitacao nao encontrada"));
    DocumentoOnboarding doc = documentoRepository.findById(documentoId)
        .orElseThrow(() -> new IllegalArgumentException("Documento nao encontrado"));
    if (!doc.getSolicitacao().getId().equals(solicitacaoId)) {
        throw new IllegalArgumentException("Documento nao pertence a esta solicitacao");
    }
    doc.setValidado(validado);
    doc.setObservacaoValidacao(observacao);
    documentoRepository.save(doc);
}
```

- [ ] **Step 2: Add endpoint to ControllerPF**

Read existing `ControllerPF.java`. Add:

```java
@PostMapping("/solicitacoes/{id}/documentos/{documentoId}/validar")
@Operation(summary = "Validar ou rejeitar um documento da solicitacao PF")
public ResponseEntity<Void> validarDocumento(@PathVariable Long id, @PathVariable Long documentoId, @RequestBody Map<String, Object> body) {
    boolean validado = Boolean.TRUE.equals(body.get("validado"));
    String observacao = (String) body.getOrDefault("observacao", null);
    onboardingPFService.validarDocumento(id, documentoId, validado, observacao);
    return ResponseEntity.noContent().build();
}
```

- [ ] **Step 3: Add `validarDocumento` to OnboardingPJService**

Same logic as PF but in `OnboardingPJService.java`:

```java
public void validarDocumento(Long solicitacaoId, Long documentoId, boolean validado, String observacao) {
    String tenantId = TenantContext.getTenantId();
    SolicitacaoOnboarding onboarding = solicitacaoOnboardingRepository.findByTenantIdAndId(tenantId, solicitacaoId)
        .orElseThrow(() -> new IllegalArgumentException("Solicitacao nao encontrada"));
    DocumentoOnboarding doc = documentoRepository.findById(documentoId)
        .orElseThrow(() -> new IllegalArgumentException("Documento nao encontrado"));
    if (!doc.getSolicitacao().getId().equals(solicitacaoId)) {
        throw new IllegalArgumentException("Documento nao pertence a esta solicitacao");
    }
    doc.setValidado(validado);
    doc.setObservacaoValidacao(observacao);
    documentoRepository.save(doc);
}
```

- [ ] **Step 4: Add endpoint to ControllerPJ**

```java
@PostMapping("/{id}/documentos/{documentoId}/validar")
@Operation(summary = "Validar ou rejeitar um documento da solicitacao PJ")
public ResponseEntity<Void> validarDocumento(@PathVariable Long id, @PathVariable Long documentoId, @RequestBody Map<String, Object> body) {
    boolean validado = Boolean.TRUE.equals(body.get("validado"));
    String observacao = (String) body.getOrDefault("observacao", null);
    onboardingPJService.validarDocumento(id, documentoId, validado, observacao);
    return ResponseEntity.noContent().build();
}
```

- [ ] **Step 5: Verify compilation**

Run: `mvn compile -pl aurix-onboarding -am -DskipTests` from `apps/backend/`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add apps/backend/aurix-onboarding/src/main/java/com/aurix/platform/onboarding/controller/ControllerPF.java apps/backend/aurix-onboarding/src/main/java/com/aurix/platform/onboarding/controller/ControllerPJ.java apps/backend/aurix-onboarding/src/main/java/com/aurix/platform/onboarding/service/OnboardingPFService.java apps/backend/aurix-onboarding/src/main/java/com/aurix/platform/onboarding/service/OnboardingPJService.java
git commit -m "feat(onboarding): add document validation endpoint (PF + PJ)"
```

---

### Task 2: Frontend — Document Validation Dialog

**Files:**
- Modify: `apps/frontend/aurix-admin/src/components/DocumentList.js`

- [ ] **Step 1: Add validation dialog to DocumentList**

Read existing `DocumentList.js`. The component currently has:
- Upload form at top
- Table with rows showing: ID, Tipo, Arquivo, Validado (chip)
- `onRefresh` prop for refresh

Add a new column "Ações" with a button "Validar" per row (disabled if doc already validated). Add state for dialog control.

Keep all existing code. Add after the `Chip` column:

```js
import React, { useState } from 'react';
// ... existing imports ...
import {
  // ... existing ...
  IconButton, Tooltip, Dialog, DialogTitle, DialogContent, DialogActions,
  Button, Radio, RadioGroup, FormControlLabel, FormControl, FormLabel,
} from '@mui/material';
import { CheckCircle, GppMaybe } from '@mui/icons-material';
```

Inside the component, add state:
```js
const [validarDialog, setValidarDialog] = useState({ open: false, doc: null });
const [validarAcao, setValidarAcao] = useState('aprovar');
const [validarObs, setValidarObs] = useState('');
const [validarLoading, setValidarLoading] = useState(false);
```

Add after the upload section and before the table:
```js
const handleValidarClick = (doc) => {
  setValidarDialog({ open: true, doc });
  setValidarAcao('aprovar');
  setValidarObs('');
};

const handleValidarConfirm = async () => {
  setValidarLoading(true);
  try {
    const token = localStorage.getItem('token');
    const headers = new Headers({
      'Content-Type': 'application/json',
      'Authorization': token ? `Bearer ${token}` : '',
    });
    const url = `${getResourceUrl(resourceName, solicitacaoId)}/documentos/${validarDialog.doc.id}/validar`;
    await fetchUtils.fetchJson(url, {
      method: 'POST',
      headers,
      body: JSON.stringify({
        validado: validarAcao === 'aprovar',
        observacao: validarObs || null,
      }),
    });
    setValidarDialog({ open: false, doc: null });
    if (onRefresh) onRefresh();
  } catch (e) {
    console.error('Validation failed', e);
  } finally {
    setValidarLoading(false);
  }
};
```

Add table header cell `<TableCell>Ações</TableCell>` and body cell:
```js
<TableCell>
  <Tooltip title={doc.validado ? 'Documento já validado' : 'Validar documento'}>
    <span>
      <IconButton
        size="small"
        color="primary"
        disabled={doc.validado}
        onClick={() => handleValidarClick(doc)}
      >
        {doc.validado ? <CheckCircle /> : <GppMaybe />}
      </IconButton>
    </span>
  </Tooltip>
</TableCell>
```

Add the dialog (after the table/empty state, before the closing `</Box>`):
```js
<Dialog open={validarDialog.open} onClose={() => setValidarDialog({ open: false, doc: null })} maxWidth="sm" fullWidth>
  <DialogTitle>Validar Documento</DialogTitle>
  <DialogContent>
    {validarDialog.doc && (
      <>
        <Typography variant="body2" gutterBottom>
          <strong>Tipo:</strong> {validarDialog.doc.tipoDocumento}
        </Typography>
        <Typography variant="body2" gutterBottom>
          <strong>Arquivo:</strong> {validarDialog.doc.nomeArquivo}
        </Typography>
        <FormControl sx={{ mt: 2, mb: 2 }}>
          <FormLabel>Decisão</FormLabel>
          <RadioGroup row value={validarAcao} onChange={(e) => setValidarAcao(e.target.value)}>
            <FormControlLabel value="aprovar" control={<Radio />} label="Aprovar" />
            <FormControlLabel value="rejeitar" control={<Radio />} label="Rejeitar" />
          </RadioGroup>
        </FormControl>
        <TextField
          label="Observação"
          fullWidth
          multiline
          rows={3}
          value={validarObs}
          onChange={(e) => setValidarObs(e.target.value)}
        />
      </>
    )}
  </DialogContent>
  <DialogActions>
    <Button onClick={() => setValidarDialog({ open: false, doc: null })}>Cancelar</Button>
    <Button variant="contained" onClick={handleValidarConfirm} disabled={validarLoading}>
      {validarLoading ? 'Salvando...' : 'Confirmar'}
    </Button>
  </DialogActions>
</Dialog>
```

- [ ] **Step 2: Verify lint**

Run: `npm run lint --workspace=aurix-admin` from `apps/frontend/`
Expected: 0 errors, only pre-existing warnings

- [ ] **Step 3: Commit**

```bash
git add apps/frontend/aurix-admin/src/components/DocumentList.js
git commit -m "feat(admin): add document validation dialog to DocumentList"
```
