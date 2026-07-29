# Onboarding Admin Dashboard + Bulk Actions + Mobile Upload Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add onboarding metrics to the admin Dashboard, bulk approve/reject actions to List pages, and document upload step to mobile onboarding.

**Architecture:** Three independent subsystems — (1) frontend-only Dashboard additions using existing API, (2) frontend-only bulk action components on two List pages, (3) two new mobile screens + service method + navigation wiring.

**Tech Stack:** React Admin 4 + MUI 5 (admin), React Native 0.73 + react-native-image-picker (mobile)

## Global Constraints

- Admin: functional components, `export const`, no default exports, Portuguese labels, `date-fns` with `ptBR`
- Mobile: React Native 0.73, no Expo, plain JavaScript (no TypeScript), singleton class services, `StyleSheet.create`, `Colors.js` constants
- Admin data fetching: use `fetchUtils` + `getActionUrl`/`getResourceUrl` from `config/resources.js` for custom API calls (not `useGetList` for computed metrics)
- Mobile document upload: base64 in `urlStorage` field, endpoint accepts JSON (not MultipartFile)
- Commit messages follow conventional commits pattern

---

## File Inventory

### Create:
- `frontend/aurix-admin/src/components/BulkApproveReject.js` — bulk approve/reject toolbar
- `frontend/aurix-mobile/src/pages/onboarding/StepDocumentosPF.js` — PF document upload screen
- `frontend/aurix-mobile/src/pages/onboarding/StepDocumentosPJ.js` — PJ document upload screen

### Modify:
- `frontend/aurix-admin/src/pages/Dashboard.js` — add onboarding stat cards + funnel chart
- `frontend/aurix-admin/src/pages/SolicitacoesConta/SolicitacaoContaList.js` — add bulkActionButtons
- `frontend/aurix-admin/src/pages/SolicitacoesPJ/SolicitacaoPJList.js` — add bulkActionButtons
- `frontend/aurix-mobile/src/services/onboardingService.js` — add `uploadDocumento` method
- `frontend/aurix-mobile/src/navigation/AuthNavigator.js` — add `StepDocumentosPF`, `StepDocumentosPJ` routes
- `frontend/aurix-mobile/src/pages/onboarding/FormPF.js` — navigate to StepDocumentosPF instead of SuccessScreen
- `frontend/aurix-mobile/src/pages/onboarding/StepSocios.js` — navigate to StepDocumentosPJ instead of SuccessScreen
- `frontend/aurix-mobile/package.json` — add `react-native-image-picker` dependency

---

### Task 1: Admin Dashboard — Onboarding Metrics Section

**Files:**
- Modify: `frontend/aurix-admin/src/pages/Dashboard.js:40-end`

**Interfaces:**
- Consumes: existing `fetchUtils` + `getResourceUrl` from `config/resources.js`
- Produces: onboarding metrics section rendered in Dashboard

- [ ] **Step 1: Add onboarding stat cards**

Read the existing Dashboard and `resources.js` config. Import `fetchUtils`, `getResourceUrl`, `Box`, `Grid`, `Card`, `CardContent`, `Typography`, `CircularProgress` from MUI. Keep all existing code, add a new section after the first stat row and before the charts row.

Add state variables:
```js
const [pfCount, setPfCount] = React.useState(null);
const [pjCount, setPjCount] = React.useState(null);
const [pfStatusCounts, setPfStatusCounts] = React.useState({});
const [pjStatusCounts, setPjStatusCounts] = React.useState({});
const [loadingMetrics, setLoadingMetrics] = React.useState(true);
```

Add a `useEffect` that fetches PF and PJ onboardings:
```js
const token = localStorage.getItem('token');
const headers = new Headers({ 'Authorization': token ? `Bearer ${token}` : '', 'Content-Type': 'application/json' });

const fetchPf = fetchUtils.fetchJson(getResourceUrl('solicitacoes_conta'), { headers })
  .then(({ json }) => {
    const data = Array.isArray(json) ? json : (json.content || []);
    setPfCount(data.length);
    const counts = {};
    data.forEach(item => { counts[item.status] = (counts[item.status] || 0) + 1; });
    setPfStatusCounts(counts);
  });

const fetchPj = fetchUtils.fetchJson(getResourceUrl('solicitacoes_pj'), { headers })
  .then(({ json }) => {
    const data = Array.isArray(json) ? json : (json.content || []);
    setPjCount(data.length);
    const counts = {};
    data.forEach(item => { counts[item.status] = (counts[item.status] || 0) + 1; });
    setPjStatusCounts(counts);
  });

Promise.all([fetchPf, fetchPj]).finally(() => setLoadingMetrics(false));
```

Add onboarding stat row after line 121 (`</Grid>` closing the first stat row):
```js
<Typography variant="h5" gutterBottom sx={{ mt: 2 }}>
  Onboarding
</Typography>

<Grid container spacing={3} sx={{ mb: 3 }}>
  <Grid item xs={12} sm={6} md={3}>
    <StatCard
      title="Solicitações PF Pendentes"
      value={loadingMetrics ? '-' : ((pfStatusCounts['RECEBIDA'] || 0) + (pfStatusCounts['DOCUMENTOS_PENDENTES'] || 0) + (pfStatusCounts['EM_ANALISE_KYC'] || 0))}
      icon="📋"
      color="#1976d2"
    />
  </Grid>
  <Grid item xs={12} sm={6} md={3}>
    <StatCard
      title="Solicitações PJ Pendentes"
      value={loadingMetrics ? '-' : ((pjStatusCounts['RECEBIDA'] || 0) + (pjStatusCounts['EM_PREENCHIMENTO'] || 0) + (pjStatusCounts['CNPJ_CONSULTADO'] || 0) + (pjStatusCounts['SOCIOS_VALIDADOS'] || 0) + (pjStatusCounts['DOCUMENTOS_PENDENTES'] || 0) + (pjStatusCounts['EM_ANALISE_KYC'] || 0) + (pjStatusCounts['DOCUMENTOS_ANALISADOS'] || 0) + (pjStatusCounts['AML_APROVADO'] || 0) + (pjStatusCounts['COMPLIANCE_APROVADO'] || 0) + (pjStatusCounts['EM_ASSINATURA'] || 0) + (pjStatusCounts['CONTRATO_ASSINADO'] || 0))}
      icon="🏢"
      color="#ed6c02"
    />
  </Grid>
  <Grid item xs={12} sm={6} md={3}>
    <StatCard
      title="Taxa Aprovação PF"
      value={loadingMetrics ? '-' : (() => {
        const aprovadas = (pfStatusCounts['APROVADA'] || 0) + (pfStatusCounts['CONTA_CRIADA'] || 0);
        const total = pfCount - (pfStatusCounts['CANCELADA'] || 0);
        return total > 0 ? `${((aprovadas / total) * 100).toFixed(1)}%` : '0%';
      })()}
      icon="✅"
      color="#2e7d32"
    />
  </Grid>
  <Grid item xs={12} sm={6} md={3}>
    <StatCard
      title="Taxa Aprovação PJ"
      value={loadingMetrics ? '-' : (() => {
        const aprovadas = (pjStatusCounts['APROVADA'] || 0) + (pjStatusCounts['CONTA_CRIADA'] || 0);
        const total = pjCount - (pjStatusCounts['CANCELADA'] || 0);
        return total > 0 ? `${((aprovadas / total) * 100).toFixed(1)}%` : '0%';
      })()}
      icon="✅"
      color="#2e7d32"
    />
  </Grid>
</Grid>
```

- [ ] **Step 2: Add Onboarding Funnel chart**

Add after the onboarding stat row (before the existing `</Box>` closing tag), inside the bottom chart row grid:

```js
<Grid item xs={12}>
  <Card>
    <CardHeader title="Funil de Onboarding" />
    <CardContent>
      <ResponsiveContainer width="100%" height={300}>
        <BarChart
          data={[
            { status: 'Recebida', PF: pfStatusCounts['RECEBIDA'] || 0, PJ: pjStatusCounts['RECEBIDA'] || 0 },
            { status: 'Docs Pendentes', PF: pfStatusCounts['DOCUMENTOS_PENDENTES'] || 0, PJ: pjStatusCounts['DOCUMENTOS_PENDENTES'] || 0 },
            { status: 'Em Análise KYC', PF: pfStatusCounts['EM_ANALISE_KYC'] || 0, PJ: pjStatusCounts['EM_ANALISE_KYC'] || 0 },
            { status: 'KYC Aprovado', PF: pfStatusCounts['KYC_APROVADO'] || 0, PJ: pjStatusCounts['KYC_APROVADO'] || 0 },
            { status: 'Aprovada', PF: pfStatusCounts['APROVADA'] || 0, PJ: pjStatusCounts['APROVADA'] || 0 },
            { status: 'Conta Criada', PF: pfStatusCounts['CONTA_CRIADA'] || 0, PJ: pjStatusCounts['CONTA_CRIADA'] || 0 },
          ]}
          margin={{ top: 5, right: 30, left: 20, bottom: 5 }}
        >
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis dataKey="status" />
          <YAxis />
          <Tooltip />
          <Legend />
          <Bar dataKey="PF" fill="#1976d2" />
          <Bar dataKey="PJ" fill="#ed6c02" />
        </BarChart>
      </ResponsiveContainer>
    </CardContent>
  </Card>
</Grid>
```

- [ ] **Step 3: Verify Dashboard renders without errors**

Run: `npm run lint` from `frontend/`
Expected: no new lint errors (pre-existing warnings OK)

- [ ] **Step 4: Commit**

```bash
git add frontend/aurix-admin/src/pages/Dashboard.js
git commit -m "feat(admin): add onboarding metrics cards and funnel chart to Dashboard"
```

---

### Task 2: Admin Bulk Approve/Reject Component

**Files:**
- Create: `frontend/aurix-admin/src/components/BulkApproveReject.js`
- Modify: `frontend/aurix-admin/src/pages/SolicitacoesConta/SolicitacaoContaList.js`
- Modify: `frontend/aurix-admin/src/pages/SolicitacoesPJ/SolicitacaoPJList.js`

**Interfaces:**
- Consumes: `fetchUtils`, `getActionUrl` from `config/resources.js`, selectedIds from Datagrid
- Produces: bulk approve/reject toolbar wired into both List pages

- [ ] **Step 1: Create BulkApproveReject component**

File `frontend/aurix-admin/src/components/BulkApproveReject.js`:

```js
import React, { useState } from 'react';
import {
  Box, Button, Dialog, DialogTitle, DialogContent, DialogContentText,
  DialogActions, TextField, Snackbar, Alert,
} from '@mui/material';
import { CheckCircle, Cancel } from '@mui/icons-material';
import { fetchUtils } from 'react-admin';
import { getActionUrl } from '../config/resources';

export const BulkApproveReject = ({ selectedIds, resourceName, onRefresh }) => {
  const [confirmOpen, setConfirmOpen] = useState(false);
  const [action, setAction] = useState(null);
  const [observacao, setObservacao] = useState('');
  const [processing, setProcessing] = useState(false);
  const [snackbar, setSnackbar] = useState({ open: false, message: '', severity: 'success' });

  const handleClick = (tipo) => {
    setAction(tipo);
    setObservacao('');
    setConfirmOpen(true);
  };

  const handleConfirm = async () => {
    setProcessing(true);
    const token = localStorage.getItem('token');
    const headers = new Headers({
      'Content-Type': 'application/json',
      'Authorization': token ? `Bearer ${token}` : '',
    });
    let success = 0;
    let errors = 0;
    for (const id of selectedIds) {
      try {
        let url;
        if (action === 'rejeitar') {
          url = `${getActionUrl(resourceName, id, 'rejeitar')}?usuarioAnalista=admin&observacao=${encodeURIComponent(observacao)}`;
        } else {
          url = `${getActionUrl(resourceName, id, 'aprovar')}?usuarioAnalista=admin&observacao=${encodeURIComponent(observacao)}`;
        }
        await fetchUtils.fetchJson(url, { method: 'POST', headers });
        success++;
      } catch (e) {
        errors++;
      }
    }
    setProcessing(false);
    setConfirmOpen(false);
    const message = errors > 0
      ? `${success} processada(s), ${errors} erro(s)`
      : `${success} solicitação(ões) ${action === 'aprovar' ? 'aprovada(s)' : 'rejeitada(s)'} com sucesso`;
    setSnackbar({ open: true, message, severity: errors > 0 && success === 0 ? 'error' : 'success' });
    if (onRefresh) onRefresh();
  };

  return (
    <>
      {selectedIds.length > 0 && (
        <Box sx={{ display: 'flex', gap: 1, alignItems: 'center', ml: 2 }}>
          <Button
            size="small"
            variant="contained"
            color="success"
            startIcon={<CheckCircle />}
            onClick={() => handleClick('aprovar')}
          >
            Aprovar ({selectedIds.length})
          </Button>
          <Button
            size="small"
            variant="outlined"
            color="error"
            startIcon={<Cancel />}
            onClick={() => handleClick('rejeitar')}
          >
            Rejeitar ({selectedIds.length})
          </Button>
        </Box>
      )}

      <Dialog open={confirmOpen} onClose={() => setConfirmOpen(false)} maxWidth="sm" fullWidth>
        <DialogTitle>
          {action === 'aprovar' ? 'Aprovar Solicitações' : 'Rejeitar Solicitações'}
        </DialogTitle>
        <DialogContent>
          <DialogContentText>
            {action === 'aprovar'
              ? `Deseja aprovar ${selectedIds.length} solicitação(ões)? Os clientes e contas serão criados automaticamente.`
              : `Deseja rejeitar ${selectedIds.length} solicitação(ões)? Informe o motivo abaixo.`}
          </DialogContentText>
          <TextField
            autoFocus
            margin="dense"
            label="Observação (opcional)"
            fullWidth
            multiline
            rows={3}
            value={observacao}
            onChange={(e) => setObservacao(e.target.value)}
          />
        </DialogContent>
        <DialogActions>
          <Button onClick={() => setConfirmOpen(false)}>Cancelar</Button>
          <Button variant="contained" onClick={handleConfirm} disabled={processing}>
            {processing ? 'Processando...' : 'Confirmar'}
          </Button>
        </DialogActions>
      </Dialog>

      <Snackbar
        open={snackbar.open}
        autoHideDuration={6000}
        onClose={() => setSnackbar({ ...snackbar, open: false })}
      >
        <Alert severity={snackbar.severity} onClose={() => setSnackbar({ ...snackbar, open: false })}>
          {snackbar.message}
        </Alert>
      </Snackbar>
    </>
  );
};
```

- [ ] **Step 2: Wire into SolicitacaoContaList**

Modify `SolicitacaoContaList.js` — import `BulkApproveReject`, add `bulkActionButtons` prop to `Datagrid`:

Add import:
```js
import { BulkApproveReject } from '../../components/BulkApproveReject';
```

Modify Datagrid:
```js
<Datagrid bulkActionButtons={<BulkApproveReject resourceName="solicitacoes_conta" />}>
```

- [ ] **Step 3: Wire into SolicitacaoPJList**

Same changes in `SolicitacaoPJList.js`:
```js
import { BulkApproveReject } from '../../components/BulkApproveReject';
// ...
<Datagrid bulkActionButtons={<BulkApproveReject resourceName="solicitacoes_pj" />}>
```

- [ ] **Step 4: Verify lint**

Run: `npm run lint` from `frontend/`
Expected: no new errors

- [ ] **Step 5: Commit**

```bash
git add frontend/aurix-admin/src/components/BulkApproveReject.js frontend/aurix-admin/src/pages/SolicitacoesConta/SolicitacaoContaList.js frontend/aurix-admin/src/pages/SolicitacoesPJ/SolicitacaoPJList.js
git commit -m "feat(admin): add bulk approve/reject actions to onboarding lists"
```

---

### Task 3: Mobile — Add uploadDocumento Service Method

**Files:**
- Modify: `frontend/aurix-mobile/src/services/onboardingService.js`
- Modify: `frontend/aurix-mobile/package.json`

**Interfaces:**
- Consumes: existing axios instance from onboardingService
- Produces: `uploadDocumento(solicitacaoId, tipoDocumento, nomeArquivo, urlStorage, tipoPessoa)` method

- [ ] **Step 1: Add uploadDocumento method**

Read existing `onboardingService.js`. Add:

```js
  uploadDocumento(solicitacaoId, tipoDocumento, nomeArquivo, urlStorage, tipoPessoa) {
    const basePath = tipoPessoa === 'PF'
      ? '/onboarding/contas/pf/solicitacoes'
      : '/onboarding/contas/pj';
    return api.post(`${basePath}/${solicitacaoId}/documentos`, {
      tipoDocumento,
      nomeArquivo,
      urlStorage,
    });
  }
```

- [ ] **Step 2: Add react-native-image-picker to package.json**

Read `package.json`. Add to `dependencies`:

```json
"react-native-image-picker": "^7.1.0",
```

- [ ] **Step 3: Commit**

```bash
git add frontend/aurix-mobile/src/services/onboardingService.js frontend/aurix-mobile/package.json
git commit -m "feat(mobile): add uploadDocumento method and react-native-image-picker dep"
```

---

### Task 4: Mobile — StepDocumentosPF Screen

**Files:**
- Create: `frontend/aurix-mobile/src/pages/onboarding/StepDocumentosPF.js`
- Modify: `frontend/aurix-mobile/src/navigation/AuthNavigator.js`
- Modify: `frontend/aurix-mobile/src/pages/onboarding/FormPF.js`

**Interfaces:**
- Consumes: `onboardingService.uploadDocumento()`, `solicitacaoId` from route params
- Produces: document upload UI for PF flow

- [ ] **Step 1: Create StepDocumentosPF component**

File `frontend/aurix-mobile/src/pages/onboarding/StepDocumentosPF.js`:

```js
import React, { useState, useCallback } from 'react';
import {
  View, Text, StyleSheet, TouchableOpacity, ScrollView,
  Image, Alert, ActivityIndicator,
} from 'react-native';
import { launchCamera, launchImageLibrary } from 'react-native-image-picker';
import { Colors } from '../../constants/Colors';
import { onboardingService } from '../../services/onboardingService';

const DOCS_REQUIRED = [
  { key: 'RG', label: 'Documento de Identidade (RG/CNH)' },
  { key: 'CPF', label: 'CPF' },
  { key: 'COMPROVANTE_ENDERECO', label: 'Comprovante de Endereço' },
  { key: 'COMPROVANTE_RENDA', label: 'Comprovante de Renda' },
];

export const StepDocumentosPF = ({ route, navigation }) => {
  const { solicitacaoId } = route.params;
  const [documents, setDocuments] = useState({});
  const [uploading, setUploading] = useState({});
  const [submitting, setSubmitting] = useState(false);

  const pickImage = useCallback(async (docKey, useCamera) => {
    const options = {
      mediaType: 'photo',
      includeBase64: true,
      maxWidth: 1024,
      maxHeight: 1024,
      quality: 0.7,
    };
    const result = useCamera
      ? await launchCamera(options)
      : await launchImageLibrary(options);
    if (result.didCancel || result.errorCode) return;
    const asset = result.assets[0];
    setDocuments((prev) => ({
      ...prev,
      [docKey]: {
        base64: asset.base64,
        uri: asset.uri,
        fileName: `${docKey}_${Date.now()}.jpg`,
      },
    }));
  }, []);

  const removeDocument = (docKey) => {
    setDocuments((prev) => {
      const next = { ...prev };
      delete next[docKey];
      return next;
    });
  };

  const handleSubmit = async () => {
    setSubmitting(true);
    const docKeys = Object.keys(documents);
    let success = 0;
    for (const key of docKeys) {
      setUploading((prev) => ({ ...prev, [key]: true }));
      try {
        await onboardingService.uploadDocumento(
          solicitacaoId,
          key,
          documents[key].fileName,
          documents[key].base64,
          'PF'
        );
        success++;
      } catch (e) {
        Alert.alert('Erro', `Falha ao enviar ${key}: ${e.message}`);
      }
      setUploading((prev) => ({ ...prev, [key]: false }));
    }
    setSubmitting(false);
    if (success > 0) {
      navigation.replace('SuccessScreen', { protocolo: `DOCS-${solicitacaoId}` });
    }
  };

  const allUploaded = DOCS_REQUIRED.every((d) => documents[d.key]);

  return (
    <ScrollView style={styles.container} contentContainerStyle={styles.content}>
      <Text style={styles.title}>Envio de Documentos</Text>
      <Text style={styles.subtitle}>Anexe os documentos necessários para análise</Text>

      {DOCS_REQUIRED.map((doc) => (
        <View key={doc.key} style={styles.docItem}>
          <Text style={styles.docLabel}>{doc.label}</Text>
          {documents[doc.key] ? (
            <View style={styles.previewContainer}>
              <Image source={{ uri: documents[doc.key].uri }} style={styles.preview} />
              <View style={styles.previewActions}>
                <TouchableOpacity onPress={() => pickImage(doc.key, true)} style={styles.smallBtn}>
                  <Text style={styles.smallBtnText}>Recapturar</Text>
                </TouchableOpacity>
                <TouchableOpacity onPress={() => removeDocument(doc.key)} style={styles.removeBtn}>
                  <Text style={styles.removeBtnText}>Remover</Text>
                </TouchableOpacity>
              </View>
              {uploading[doc.key] && <ActivityIndicator size="small" color={Colors.primary} />}
            </View>
          ) : (
            <View style={styles.actions}>
              <TouchableOpacity onPress={() => pickImage(doc.key, true)} style={styles.captureBtn}>
                <Text style={styles.btnText}>📷 Capturar</Text>
              </TouchableOpacity>
              <TouchableOpacity onPress={() => pickImage(doc.key, false)} style={styles.galleryBtn}>
                <Text style={styles.btnText}>🖼 Galeria</Text>
              </TouchableOpacity>
            </View>
          )}
        </View>
      ))}

      <TouchableOpacity
        style={[styles.submitBtn, (!allUploaded || submitting) && styles.disabledBtn]}
        onPress={handleSubmit}
        disabled={!allUploaded || submitting}
      >
        {submitting ? (
          <ActivityIndicator color="#fff" />
        ) : (
          <Text style={styles.submitBtnText}>Enviar Documentos</Text>
        )}
      </TouchableOpacity>
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: Colors.background },
  content: { padding: 20, paddingBottom: 40 },
  title: { fontSize: 24, fontWeight: 'bold', color: Colors.text, marginBottom: 8 },
  subtitle: { fontSize: 14, color: Colors.gray, marginBottom: 24 },
  docItem: { backgroundColor: Colors.surface, borderRadius: 8, padding: 16, marginBottom: 16, borderWidth: 1, borderColor: Colors.border },
  docLabel: { fontSize: 15, fontWeight: '600', color: Colors.text, marginBottom: 12 },
  actions: { flexDirection: 'row', gap: 12 },
  captureBtn: { flex: 1, backgroundColor: Colors.primary, padding: 12, borderRadius: 8, alignItems: 'center' },
  galleryBtn: { flex: 1, backgroundColor: Colors.gray, padding: 12, borderRadius: 8, alignItems: 'center' },
  btnText: { color: '#fff', fontWeight: '600', fontSize: 14 },
  previewContainer: { alignItems: 'center' },
  preview: { width: '100%', height: 160, borderRadius: 8, marginBottom: 8, resizeMode: 'cover' },
  previewActions: { flexDirection: 'row', gap: 12 },
  smallBtn: { backgroundColor: Colors.primary, padding: 8, borderRadius: 6 },
  smallBtnText: { color: '#fff', fontSize: 13 },
  removeBtn: { backgroundColor: Colors.error, padding: 8, borderRadius: 6 },
  removeBtnText: { color: '#fff', fontSize: 13 },
  submitBtn: { backgroundColor: Colors.success, padding: 16, borderRadius: 8, alignItems: 'center', marginTop: 16 },
  disabledBtn: { opacity: 0.5 },
  submitBtnText: { color: '#fff', fontWeight: 'bold', fontSize: 16 },
});
```

- [ ] **Step 2: Add StepDocumentosPF route to AuthNavigator**

Read `AuthNavigator.js`. Add import and route.

Import:
```js
import { StepDocumentosPF } from '../pages/onboarding/StepDocumentosPF';
```

Add after the `FormPF` screen definition:
```js
<AuthStack.Screen
  name="StepDocumentosPF"
  component={StepDocumentosPF}
  options={{ headerTitle: 'Envio de Documentos' }}
/>
```

- [ ] **Step 3: Modify FormPF to navigate to StepDocumentosPF**

In `FormPF.js`, find the `handleSubmit` function where it navigates to `SuccessScreen` after successful submission. Change the success navigation to:

```js
navigation.replace('StepDocumentosPF', { solicitacaoId: result.id });
```

- [ ] **Step 4: Commit**

```bash
git add frontend/aurix-mobile/src/pages/onboarding/StepDocumentosPF.js frontend/aurix-mobile/src/navigation/AuthNavigator.js frontend/aurix-mobile/src/pages/onboarding/FormPF.js
git commit -m "feat(mobile): add PF document upload screen"
```

---

### Task 5: Mobile — StepDocumentosPJ Screen

**Files:**
- Create: `frontend/aurix-mobile/src/pages/onboarding/StepDocumentosPJ.js`
- Modify: `frontend/aurix-mobile/src/navigation/AuthNavigator.js`
- Modify: `frontend/aurix-mobile/src/pages/onboarding/StepSocios.js`

**Interfaces:**
- Consumes: `onboardingService.uploadDocumento()`, `solicitacaoId` + `socios` from route params
- Produces: document upload UI for PJ flow (company docs + partner docs)

- [ ] **Step 1: Create StepDocumentosPJ component**

File `frontend/aurix-mobile/src/pages/onboarding/StepDocumentosPJ.js`:

```js
import React, { useState, useCallback } from 'react';
import {
  View, Text, StyleSheet, TouchableOpacity, ScrollView,
  Image, Alert, ActivityIndicator, SectionList,
} from 'react-native';
import { launchCamera, launchImageLibrary } from 'react-native-image-picker';
import { Colors } from '../../constants/Colors';
import { onboardingService } from '../../services/onboardingService';

const COMPANY_DOCS = [
  { key: 'CONTRATO_SOCIAL', label: 'Contrato Social' },
  { key: 'CNPJ', label: 'Cartão CNPJ' },
  { key: 'BALANCO_PATRIMONIAL', label: 'Balanço Patrimonial' },
];

const PARTNER_DOCS = [
  { key: 'IDENTIDADE_SOCIO', label: 'Documento de Identidade' },
  { key: 'CPF', label: 'CPF' },
];

export const StepDocumentosPJ = ({ route, navigation }) => {
  const { solicitacaoId, socios } = route.params;
  const [documents, setDocuments] = useState({});
  const [uploading, setUploading] = useState({});
  const [submitting, setSubmitting] = useState(false);

  const pickImage = useCallback(async (sectionId, docKey, useCamera) => {
    const options = {
      mediaType: 'photo',
      includeBase64: true,
      maxWidth: 1024,
      maxHeight: 1024,
      quality: 0.7,
    };
    const result = useCamera
      ? await launchCamera(options)
      : await launchImageLibrary(options);
    if (result.didCancel || result.errorCode) return;
    const asset = result.assets[0];
    setDocuments((prev) => ({
      ...prev,
      [`${sectionId}_${docKey}`]: {
        base64: asset.base64,
        uri: asset.uri,
        fileName: `${docKey}_${Date.now()}.jpg`,
        sectionId,
        docKey,
      },
    }));
  }, []);

  const removeDocument = (id) => {
    setDocuments((prev) => {
      const next = { ...prev };
      delete next[id];
      return next;
    });
  };

  const handleSubmit = async () => {
    setSubmitting(true);
    const entries = Object.entries(documents);
    let success = 0;
    for (const [id, doc] of entries) {
      setUploading((prev) => ({ ...prev, [id]: true }));
      try {
        await onboardingService.uploadDocumento(
          solicitacaoId,
          doc.docKey,
          doc.fileName,
          doc.base64,
          'PJ'
        );
        success++;
      } catch (e) {
        Alert.alert('Erro', `Falha ao enviar ${doc.docKey}: ${e.message}`);
      }
      setUploading((prev) => ({ ...prev, [id]: false }));
    }
    setSubmitting(false);
    if (success > 0) {
      navigation.replace('SuccessScreen', { protocolo: `DOCS-${solicitacaoId}` });
    }
  };

  const renderDocItem = (sectionId, docKey, label) => {
    const docId = `${sectionId}_${docKey}`;
    const doc = documents[docId];
    return (
      <View key={docId} style={styles.docItem}>
        <Text style={styles.docLabel}>{label}</Text>
        {doc ? (
          <View style={styles.previewContainer}>
            <Image source={{ uri: doc.uri }} style={styles.preview} />
            <View style={styles.previewActions}>
              <TouchableOpacity onPress={() => pickImage(sectionId, docKey, true)} style={styles.smallBtn}>
                <Text style={styles.smallBtnText}>Recapturar</Text>
              </TouchableOpacity>
              <TouchableOpacity onPress={() => removeDocument(docId)} style={styles.removeBtn}>
                <Text style={styles.removeBtnText}>Remover</Text>
              </TouchableOpacity>
            </View>
            {uploading[docId] && <ActivityIndicator size="small" color={Colors.primary} />}
          </View>
        ) : (
          <View style={styles.actions}>
            <TouchableOpacity onPress={() => pickImage(sectionId, docKey, true)} style={styles.captureBtn}>
              <Text style={styles.btnText}>📷 Capturar</Text>
            </TouchableOpacity>
            <TouchableOpacity onPress={() => pickImage(sectionId, docKey, false)} style={styles.galleryBtn}>
              <Text style={styles.btnText}>🖼 Galeria</Text>
            </TouchableOpacity>
          </View>
        )}
      </View>
    );
  };

  const sections = [
    { title: 'Documentos da Empresa', data: COMPANY_DOCS, sectionId: 'empresa' },
    ...(socios || []).map((socio, idx) => ({
      title: `Sócio: ${socio}`,
      data: PARTNER_DOCS,
      sectionId: `socio_${idx}`,
    })),
  ];

  return (
    <ScrollView style={styles.container} contentContainerStyle={styles.content}>
      <Text style={styles.title}>Envio de Documentos</Text>
      <Text style={styles.subtitle}>Anexe os documentos da empresa e dos sócios</Text>

      {sections.map((section) => (
        <View key={section.sectionId} style={styles.section}>
          <Text style={styles.sectionTitle}>{section.title}</Text>
          {section.data.map((doc) => renderDocItem(section.sectionId, doc.key, doc.label))}
        </View>
      ))}

      <TouchableOpacity
        style={[styles.submitBtn, submitting && styles.disabledBtn]}
        onPress={handleSubmit}
        disabled={submitting}
      >
        {submitting ? (
          <ActivityIndicator color="#fff" />
        ) : (
          <Text style={styles.submitBtnText}>Enviar Todos</Text>
        )}
      </TouchableOpacity>
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: Colors.background },
  content: { padding: 20, paddingBottom: 40 },
  title: { fontSize: 24, fontWeight: 'bold', color: Colors.text, marginBottom: 8 },
  subtitle: { fontSize: 14, color: Colors.gray, marginBottom: 24 },
  section: { marginBottom: 24 },
  sectionTitle: { fontSize: 17, fontWeight: '700', color: Colors.text, marginBottom: 12, borderBottomWidth: 1, borderBottomColor: Colors.border, paddingBottom: 8 },
  docItem: { backgroundColor: Colors.surface, borderRadius: 8, padding: 16, marginBottom: 12, borderWidth: 1, borderColor: Colors.border },
  docLabel: { fontSize: 15, fontWeight: '600', color: Colors.text, marginBottom: 12 },
  actions: { flexDirection: 'row', gap: 12 },
  captureBtn: { flex: 1, backgroundColor: Colors.primary, padding: 12, borderRadius: 8, alignItems: 'center' },
  galleryBtn: { flex: 1, backgroundColor: Colors.gray, padding: 12, borderRadius: 8, alignItems: 'center' },
  btnText: { color: '#fff', fontWeight: '600', fontSize: 14 },
  previewContainer: { alignItems: 'center' },
  preview: { width: '100%', height: 160, borderRadius: 8, marginBottom: 8, resizeMode: 'cover' },
  previewActions: { flexDirection: 'row', gap: 12 },
  smallBtn: { backgroundColor: Colors.primary, padding: 8, borderRadius: 6 },
  smallBtnText: { color: '#fff', fontSize: 13 },
  removeBtn: { backgroundColor: Colors.error, padding: 8, borderRadius: 6 },
  removeBtnText: { color: '#fff', fontSize: 13 },
  submitBtn: { backgroundColor: Colors.success, padding: 16, borderRadius: 8, alignItems: 'center', marginTop: 16 },
  disabledBtn: { opacity: 0.5 },
  submitBtnText: { color: '#fff', fontWeight: 'bold', fontSize: 16 },
});
```

- [ ] **Step 2: Add StepDocumentosPJ route to AuthNavigator**

Import:
```js
import { StepDocumentosPJ } from '../pages/onboarding/StepDocumentosPJ';
```

Add after `StepSocios` screen:
```js
<AuthStack.Screen
  name="StepDocumentosPJ"
  component={StepDocumentosPJ}
  options={{ headerTitle: 'Envio de Documentos' }}
/>
```

- [ ] **Step 3: Modify StepSocios to navigate to StepDocumentosPJ**

In `StepSocios.js`, find where it navigates to `SuccessScreen` after successful submission. Change to:

```js
const socioNames = socios.map((s) => s.nome);
navigation.replace('StepDocumentosPJ', {
  solicitacaoId: result.id,
  socios: socioNames,
});
```

- [ ] **Step 4: Commit**

```bash
git add frontend/aurix-mobile/src/pages/onboarding/StepDocumentosPJ.js frontend/aurix-mobile/src/navigation/AuthNavigator.js frontend/aurix-mobile/src/pages/onboarding/StepSocios.js
git commit -m "feat(mobile): add PJ document upload screen"
```

---

## Self-Review Checklist

1. **Spec coverage:**
   - Dashboard stat cards → Task 1, Step 1
   - Dashboard funnel chart → Task 1, Step 2
   - Bulk approve/reject → Task 2, all steps
   - PF document upload screen → Task 4
   - PJ document upload screen → Task 5
   - `uploadDocumento` service method → Task 3, Step 1
   - `react-native-image-picker` dep → Task 3, Step 2
   - Navigation changes → Task 4, Steps 2-3; Task 5, Steps 2-3
   - FormPF redirect → Task 4, Step 3
   - StepSocios redirect → Task 5, Step 3

2. **Placeholder scan:** No TBD, TODO, or placeholder patterns found.

3. **Type consistency:** All file paths, function signatures, and state shapes are consistent across tasks. Navigation params match between caller (FormPF/StepSocios) and receiver (StepDocumentosPF/StepDocumentosPJ).
