# Frontend Onboarding PJ Admin — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add PJ onboarding management pages to aureus-admin (back office) with reusable workflow components.

**Architecture:** New `solicitacoes_pj` react-admin resource mapping to `/api/onboarding/contas/pj` with 4 reusable components (`StatusTimeline`, `DocumentList`, `SocioList`, `WorkflowActions`) and 2 pages (`SolicitacaoPJList`, `SolicitacaoPJShow`). Custom actions call the API directly via `fetchUtils.fetchJson`.

**Tech Stack:** React 18, react-admin 4.15, MUI 5, @mui/lab Timeline

## Global Constraints

- All paths are relative to `/mnt/c/Users/wende/Projects/aureus-platform/frontend/aureus-admin/src/`
- Follow existing patterns: functional components, `export const`, no default exports for page components, `label` prop on resource fields
- Resource name must be `solicitacoes_pj` to match URL mapping in `resources.js`
- BASE entry: `onboarding_pj: '/api/onboarding/contas'`
- RESOURCE_PATH entry: `solicitacoes_pj: { base: BASE.onboarding_pj, path: 'pj' }`
- Import `API_URL` and `getResourceUrl` from `../config/resources` for URL construction
- Use `fetchUtils.fetchJson` (from react-admin) for custom action API calls
- StatusOnboarding values: RECEBIDA, DOCUMENTOS_PENDENTES, EM_ANALISE_KYC, KYC_APROVADO, KYC_REJEITADO, EM_ANALISE_MANUAL, APROVADA, REJEITADA, CONTA_CRIADA, EM_PREENCHIMENTO, CNPJ_CONSULTADO, SOCIOS_VALIDADOS, DOCUMENTOS_ANALISADOS, AML_APROVADO, COMPLIANCE_APROVADO, EM_ASSINATURA, CONTRATO_ASSINADO

---

### Task 1: Config — Add resource mapping and App registration

**Files:**
- Modify: `config/resources.js:11-22` (add BASE entry), `config/resources.js:36-46` (add RESOURCE_PATH entry), `config/resources.js:49-55` (add getActionUrl)
- Modify: `App.js:27` (add import), `App.js:150-155` (add Resource)

**Interfaces:**
- Produces: `BASE.onboarding_pj = '/api/onboarding/contas'`
- Produces: `RESOURCE_PATH.solicitacoes_pj = { base: BASE.onboarding_pj, path: 'pj' }`
- Produces: `getActionUrl(resource, id, action)` → `${getResourceUrl(resource, id)}/${action}`

- [ ] **Step 1: Add onboarding_pj BASE entry**

Insert line 12 (after `onboarding:`):

```
  onboarding_pj: '/api/onboarding/contas',
```

- [ ] **Step 2: Add solicitacoes_pj RESOURCE_PATH entry**

Insert line 37 (after `solicitacoes_conta:`):

```
  solicitacoes_pj: { base: BASE.onboarding_pj, path: 'pj' },
```

- [ ] **Step 3: Add getActionUrl function**

Add after `getResourceUrl` (after line 55):

```js
export const getActionUrl = (resource, id, action) => {
  return `${getResourceUrl(resource, id)}/${action}`;
};
```

- [ ] **Step 4: Add import in App.js**

Insert after line 21:

```js
import { SolicitacaoPJList, SolicitacaoPJShow } from './pages/SolicitacoesPJ';
```

- [ ] **Step 5: Add Resource in App.js**

Insert after line 118 (after `solicitacoes_conta` Resource block):

```jsx
    <Resource
      name="solicitacoes_pj"
      list={SolicitacaoPJList}
      show={SolicitacaoPJShow}
      options={{ label: 'Onboarding - PJ' }}
    />
```

- [ ] **Step 6: Verify syntax**

Run: `cd /mnt/c/Users/wende/Projects/aureus-platform/frontend && npm run lint -- --filter=aureus-admin 2>&1 | head -20`

Expected: No errors (or only pre-existing ones unrelated to our changes)

- [ ] **Step 7: Commit**

```bash
git add frontend/aureus-admin/src/config/resources.js frontend/aureus-admin/src/App.js
git commit -m "feat(admin): add solicitacoes_pj resource mapping and registration"
```

---

### Task 2: StatusTimeline component

**Files:**
- Create: `components/StatusTimeline.js`

**Interfaces:**
- Consumes: `historico: Array<{ acao: string, usuarioAnalista: string, dataAcao: string }>`, `statusAtual: string`
- Produces: `<StatusTimeline historico={[...]} statusAtual="EM_PREENCHIMENTO" />`

- [ ] **Step 1: Create StatusTimeline.js**

```jsx
import React from 'react';
import {
  Timeline,
  TimelineItem,
  TimelineSeparator,
  TimelineConnector,
  TimelineContent,
  TimelineDot,
  TimelineOppositeContent,
} from '@mui/lab';
import { Typography } from '@mui/material';
import { CheckCircleOutline, HourglassEmpty, CancelOutlined } from '@mui/icons-material';
import { format } from 'date-fns';

const STATUS_ORDER = [
  'RECEBIDA', 'EM_PREENCHIMENTO', 'CNPJ_CONSULTADO', 'SOCIOS_VALIDADOS',
  'DOCUMENTOS_PENDENTES', 'DOCUMENTOS_ANALISADOS', 'EM_ANALISE_KYC',
  'KYC_APROVADO', 'KYC_REJEITADO', 'EM_ANALISE_MANUAL',
  'AML_APROVADO', 'COMPLIANCE_APROVADO', 'EM_ASSINATURA',
  'CONTRATO_ASSINADO', 'APROVADA', 'CONTA_CRIADA', 'REJEITADA',
];

const STATUS_LABELS = {
  RECEBIDA: 'Recebida',
  EM_PREENCHIMENTO: 'Em Preenchimento',
  CNPJ_CONSULTADO: 'CNPJ Consultado',
  SOCIOS_VALIDADOS: 'Sócios Validados',
  DOCUMENTOS_PENDENTES: 'Documentos Pendentes',
  DOCUMENTOS_ANALISADOS: 'Documentos Analisados',
  EM_ANALISE_KYC: 'Em Análise KYC',
  KYC_APROVADO: 'KYC Aprovado',
  KYC_REJEITADO: 'KYC Rejeitado',
  EM_ANALISE_MANUAL: 'Em Análise Manual',
  AML_APROVADO: 'AML Aprovado',
  COMPLIANCE_APROVADO: 'Compliance Aprovado',
  EM_ASSINATURA: 'Em Assinatura',
  CONTRATO_ASSINADO: 'Contrato Assinado',
  APROVADA: 'Aprovada',
  CONTA_CRIADA: 'Conta Criada',
  REJEITADA: 'Rejeitada',
};

export const StatusTimeline = ({ historico = [], statusAtual }) => {
  const historicoAcoes = new Set(historico.map((h) => h.acao));

  return (
    <Timeline position="right">
      {STATUS_ORDER.map((status) => {
        const isCompleted = historicoAcoes.has(status) || status === statusAtual;
        const isCurrent = status === statusAtual;
        const isRejected = status === 'REJEITADA';
        const isFuture = STATUS_ORDER.indexOf(status) > STATUS_ORDER.indexOf(statusAtual);

        let dotColor = 'grey';
        let icon = <HourglassEmpty fontSize="small" />;
        if (isCompleted && !isRejected) {
          dotColor = 'success';
          icon = <CheckCircleOutline fontSize="small" />;
        } else if (isCurrent) {
          dotColor = 'primary';
          icon = <HourglassEmpty fontSize="small" />;
        } else if (isRejected && isCompleted) {
          dotColor = 'error';
          icon = <CancelOutlined fontSize="small" />;
        }

        const historicoEntry = historico.find((h) => h.acao === status);
        const formattedDate = historicoEntry
          ? format(new Date(historicoEntry.dataAcao), 'dd/MM HH:mm')
          : '';

        return (
          <TimelineItem key={status}>
            <TimelineOppositeContent color="text.secondary" sx={{ flex: 0.2 }}>
              {formattedDate}
            </TimelineOppositeContent>
            <TimelineSeparator>
              <TimelineDot color={dotColor}>{icon}</TimelineDot>
              {status !== STATUS_ORDER[STATUS_ORDER.length - 1] && (
                <TimelineConnector sx={{ bgcolor: isFuture ? 'grey.300' : 'primary.light' }} />
              )}
            </TimelineSeparator>
            <TimelineContent>
              <Typography variant="body2" fontWeight={isCurrent ? 'bold' : 'normal'}>
                {STATUS_LABELS[status] || status}
              </Typography>
              {historicoEntry?.usuarioAnalista && (
                <Typography variant="caption" color="text.secondary">
                  {historicoEntry.usuarioAnalista}
                </Typography>
              )}
            </TimelineContent>
          </TimelineItem>
        );
      })}
    </Timeline>
  );
};
```

- [ ] **Step 2: Verify no syntax errors**

Run: `node -e "require('@babel/core').parse(require('fs').readFileSync('components/StatusTimeline.js','utf8'),{presets:['react']})" 2>&1 || echo "Need babel, checking via lint"` then `cd /mnt/c/Users/wende/Projects/aureus-platform/frontend && npm run lint -- --filter=aureus-admin 2>&1 | head -20`

Expected: No errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/aureus-admin/src/components/StatusTimeline.js
git commit -m "feat(admin): add StatusTimeline component for onboarding workflow"
```

---

### Task 3: DocumentList component

**Files:**
- Create: `components/DocumentList.js`

**Interfaces:**
- Consumes: `documentos: Array<{ id, tipoDocumento, nomeArquivo, validado }>`, `solicitacaoId: number`, `onRefresh: () => void`
- Produces: `<DocumentList documentos={[...]} solicitacaoId={1} onRefresh={fn} />`

- [ ] **Step 1: Create DocumentList.js**

```jsx
import React, { useState } from 'react';
import {
  Table, TableBody, TableCell, TableContainer, TableHead, TableRow,
  Paper, Button, TextField, MenuItem, Box, Chip, Typography,
} from '@mui/material';
import { CloudUpload } from '@mui/icons-material';
import { fetchUtils } from 'react-admin';
import { getResourceUrl } from '../config/resources';

export const DocumentList = ({ documentos = [], solicitacaoId, onRefresh }) => {
  const [tipoDocumento, setTipoDocumento] = useState('CONTRATO_SOCIAL');
  const [conteudo, setConteudo] = useState('');
  const [uploading, setUploading] = useState(false);

  const tipoOptions = [
    'CONTRATO_SOCIAL', 'CNPJ', 'IDENTIDADE_SOCIO',
    'COMPROVANTE_ENDERECO', 'BALANCO_PATRIMONIAL', 'OUTROS',
  ];

  const handleUpload = async () => {
    if (!conteudo) return;
    setUploading(true);
    try {
      const url = `${getResourceUrl('solicitacoes_pj', solicitacaoId)}/documentos`;
      const token = localStorage.getItem('token');
      const options = {
        method: 'POST',
        headers: new Headers({
          'Content-Type': 'application/json',
          'Authorization': token ? `Bearer ${token}` : '',
        }),
        body: JSON.stringify({ tipoDocumento, conteudo }),
      };
      await fetchUtils.fetchJson(url, options);
      if (onRefresh) onRefresh();
    } catch (e) {
      console.error('Upload failed', e);
    } finally {
      setUploading(false);
    }
  };

  return (
    <Box>
      <Box sx={{ mb: 2, display: 'flex', gap: 2, alignItems: 'flex-end' }}>
        <TextField
          select
          label="Tipo Documento"
          value={tipoDocumento}
          onChange={(e) => setTipoDocumento(e.target.value)}
          sx={{ minWidth: 200 }}
          size="small"
        >
          {tipoOptions.map((opt) => (
            <MenuItem key={opt} value={opt}>{opt}</MenuItem>
          ))}
        </TextField>
        <TextField
          label="Base64 do arquivo"
          value={conteudo}
          onChange={(e) => setConteudo(e.target.value)}
          multiline
          maxRows={3}
          sx={{ flex: 1 }}
          size="small"
        />
        <Button
          variant="contained"
          startIcon={<CloudUpload />}
          onClick={handleUpload}
          disabled={uploading || !conteudo}
        >
          {uploading ? 'Enviando...' : 'Upload'}
        </Button>
      </Box>

      {documentos.length === 0 ? (
        <Typography variant="body2" color="text.secondary">
          Nenhum documento enviado
        </Typography>
      ) : (
        <TableContainer component={Paper} variant="outlined">
          <Table size="small">
            <TableHead>
              <TableRow>
                <TableCell>ID</TableCell>
                <TableCell>Tipo</TableCell>
                <TableCell>Arquivo</TableCell>
                <TableCell>Validado</TableCell>
              </TableRow>
            </TableHead>
            <TableBody>
              {documentos.map((doc) => (
                <TableRow key={doc.id}>
                  <TableCell>{doc.id}</TableCell>
                  <TableCell>{doc.tipoDocumento}</TableCell>
                  <TableCell>{doc.nomeArquivo}</TableCell>
                  <TableCell>
                    <Chip
                      label={doc.validado ? 'Sim' : 'Não'}
                      color={doc.validado ? 'success' : 'warning'}
                      size="small"
                    />
                  </TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </TableContainer>
      )}
    </Box>
  );
};
```

- [ ] **Step 2: Verify via lint**

Run: `cd /mnt/c/Users/wende/Projects/aureus-platform/frontend && npm run lint -- --filter=aureus-admin 2>&1 | head -20`

Expected: No errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/aureus-admin/src/components/DocumentList.js
git commit -m "feat(admin): add DocumentList component with upload"
```

---

### Task 4: SocioList component

**Files:**
- Create: `components/SocioList.js`

**Interfaces:**
- Consumes: `socios: Array`, `solicitacaoId: number`, `onRefresh: () => void`
- Produces: `<SocioList socios={[...]} solicitacaoId={1} onRefresh={fn} />`

- [ ] **Step 1: Create SocioList.js**

```jsx
import React, { useState } from 'react';
import {
  Table, TableBody, TableCell, TableContainer, TableHead, TableRow,
  Paper, Button, Dialog, DialogTitle, DialogContent, DialogActions,
  TextField, MenuItem, Box, Typography, IconButton,
} from '@mui/material';
import { Add, Delete } from '@mui/icons-material';
import { fetchUtils } from 'react-admin';
import { getResourceUrl } from '../config/resources';

const initialForm = {
  tipo: 'SOCIO', cpf: '', nome: '', email: '',
  telefone: '', percentualParticipacao: '',
};

export const SocioList = ({ socios = [], solicitacaoId, onRefresh }) => {
  const [modalOpen, setModalOpen] = useState(false);
  const [form, setForm] = useState(initialForm);
  const [saving, setSaving] = useState(false);

  const handleAdd = async () => {
    setSaving(true);
    try {
      const url = `${getResourceUrl('solicitacoes_pj', solicitacaoId)}/socios`;
      const token = localStorage.getItem('token');
      const options = {
        method: 'POST',
        headers: new Headers({
          'Content-Type': 'application/json',
          'Authorization': token ? `Bearer ${token}` : '',
        }),
        body: JSON.stringify(form),
      };
      await fetchUtils.fetchJson(url, options);
      setModalOpen(false);
      setForm(initialForm);
      if (onRefresh) onRefresh();
    } catch (e) {
      console.error('Failed to add socio', e);
    } finally {
      setSaving(false);
    }
  };

  const handleRemove = async (participanteId) => {
    try {
      const url = `${getResourceUrl('solicitacoes_pj', solicitacaoId)}/socios/${participanteId}`;
      const token = localStorage.getItem('token');
      const options = {
        method: 'DELETE',
        headers: new Headers({
          'Authorization': token ? `Bearer ${token}` : '',
        }),
      };
      await fetchUtils.fetchJson(url, options);
      if (onRefresh) onRefresh();
    } catch (e) {
      console.error('Failed to remove socio', e);
    }
  };

  return (
    <Box>
      <Box sx={{ mb: 2 }}>
        <Button variant="contained" startIcon={<Add />} onClick={() => setModalOpen(true)}>
          Adicionar Sócio
        </Button>
      </Box>

      {socios.length === 0 ? (
        <Typography variant="body2" color="text.secondary">
          Nenhum sócio cadastrado
        </Typography>
      ) : (
        <TableContainer component={Paper} variant="outlined">
          <Table size="small">
            <TableHead>
              <TableRow>
                <TableCell>Nome</TableCell>
                <TableCell>CPF</TableCell>
                <TableCell>Tipo</TableCell>
                <TableCell>Participação</TableCell>
                <TableCell>Ação</TableCell>
              </TableRow>
            </TableHead>
            <TableBody>
              {socios.map((s) => (
                <TableRow key={s.id}>
                  <TableCell>{s.nome}</TableCell>
                  <TableCell>{s.cpf}</TableCell>
                  <TableCell>{s.tipo}</TableCell>
                  <TableCell>{s.percentualParticipacao ? `${s.percentualParticipacao}%` : '-'}</TableCell>
                  <TableCell>
                    <IconButton color="error" onClick={() => handleRemove(s.id)} size="small">
                      <Delete />
                    </IconButton>
                  </TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </TableContainer>
      )}

      <Dialog open={modalOpen} onClose={() => setModalOpen(false)} maxWidth="sm" fullWidth>
        <DialogTitle>Adicionar Sócio</DialogTitle>
        <DialogContent>
          <Box sx={{ display: 'flex', flexDirection: 'column', gap: 2, pt: 1 }}>
            <TextField
              select label="Tipo" value={form.tipo}
              onChange={(e) => setForm({ ...form, tipo: e.target.value })}
            >
              {['SOCIO', 'ADMINISTRADOR', 'REPRESENTANTE', 'PROCURADOR', 'BENEFICIARIO_FINAL'].map((t) => (
                <MenuItem key={t} value={t}>{t}</MenuItem>
              ))}
            </TextField>
            <TextField label="CPF" value={form.cpf}
              onChange={(e) => setForm({ ...form, cpf: e.target.value })}
              inputProps={{ maxLength: 11 }}
            />
            <TextField label="Nome" value={form.nome}
              onChange={(e) => setForm({ ...form, nome: e.target.value })}
            />
            <TextField label="Email" type="email" value={form.email}
              onChange={(e) => setForm({ ...form, email: e.target.value })}
            />
            <TextField label="Telefone" value={form.telefone}
              onChange={(e) => setForm({ ...form, telefone: e.target.value })}
            />
            <TextField label="Participação (%)" type="number" value={form.percentualParticipacao}
              onChange={(e) => setForm({ ...form, percentualParticipacao: e.target.value })}
            />
          </Box>
        </DialogContent>
        <DialogActions>
          <Button onClick={() => setModalOpen(false)}>Cancelar</Button>
          <Button variant="contained" onClick={handleAdd} disabled={saving}>
            {saving ? 'Salvando...' : 'Adicionar'}
          </Button>
        </DialogActions>
      </Dialog>
    </Box>
  );
};
```

- [ ] **Step 2: Verify via lint**

Run: `cd /mnt/c/Users/wende/Projects/aureus-platform/frontend && npm run lint -- --filter=aureus-admin 2>&1 | head -20`

Expected: No errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/aureus-admin/src/components/SocioList.js
git commit -m "feat(admin): add SocioList component with add/remove"
```

---

### Task 5: WorkflowActions component

**Files:**
- Create: `components/WorkflowActions.js`

**Interfaces:**
- Produces: `<WorkflowActions solicitacaoId={1} statusAtual="EM_PREENCHIMENTO" onRefresh={fn} />`

- [ ] **Step 1: Create WorkflowActions.js**

```jsx
import React, { useState } from 'react';
import {
  Box, Button, Dialog, DialogTitle, DialogContent, DialogContentText,
  DialogActions, TextField, Typography, Alert,
} from '@mui/material';
import {
  CheckCircle, Cancel, Gavel, Security, Edit, HowToReg, Search, Assignment,
} from '@mui/icons-material';
import { fetchUtils } from 'react-admin';
import { getActionUrl } from '../config/resources';

const ACTION_CONFIG = {
  'EM_PREENCHIMENTO': [
    { action: 'validar-cnpj', label: 'Validar CNPJ', icon: <Search />, color: 'primary' },
  ],
  'CNPJ_CONSULTADO': [
    { action: 'aml-aprovar', label: 'Aprovar AML', icon: <Gavel />, color: 'secondary' },
  ],
  'SOCIOS_VALIDADOS': [
    { action: 'aml-aprovar', label: 'Aprovar AML', icon: <Gavel />, color: 'secondary' },
  ],
  'DOCUMENTOS_ANALISADOS': [
    { action: 'aml-aprovar', label: 'Aprovar AML', icon: <Gavel />, color: 'secondary' },
  ],
  'AML_APROVADO': [
    { action: 'compliance-aprovar', label: 'Aprovar Compliance', icon: <Security />, color: 'info' },
  ],
  'COMPLIANCE_APROVADO': [
    { action: 'assinatura-solicitar', label: 'Solicitar Assinatura', icon: <Edit />, color: 'warning' },
  ],
  'EM_ASSINATURA': [
    { action: 'assinatura-confirmar', label: 'Confirmar Assinatura', icon: <HowToReg />, color: 'success' },
  ],
};

const STATUS_LABELS = {
  RECEBIDA: 'Recebida', EM_PREENCHIMENTO: 'Em Preenchimento',
  CNPJ_CONSULTADO: 'CNPJ Consultado', SOCIOS_VALIDADOS: 'Sócios Validados',
  DOCUMENTOS_PENDENTES: 'Documentos Pendentes', DOCUMENTOS_ANALISADOS: 'Documentos Analisados',
  EM_ANALISE_KYC: 'Em Análise KYC', KYC_APROVADO: 'KYC Aprovado',
  KYC_REJEITADO: 'KYC Rejeitado', EM_ANALISE_MANUAL: 'Em Análise Manual',
  AML_APROVADO: 'AML Aprovado', COMPLIANCE_APROVADO: 'Compliance Aprovado',
  EM_ASSINATURA: 'Em Assinatura', CONTRATO_ASSINADO: 'Contrato Assinado',
  APROVADA: 'Aprovada', CONTA_CRIADA: 'Conta Criada', REJEITADA: 'Rejeitada',
};

export const WorkflowActions = ({ solicitacaoId, statusAtual, onRefresh }) => {
  const [confirmOpen, setConfirmOpen] = useState(false);
  const [pendingAction, setPendingAction] = useState(null);
  const [observacao, setObservacao] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  const actions = ACTION_CONFIG[statusAtual] || [];
  const canReject = !['APROVADA', 'CONTA_CRIADA', 'REJEITADA', 'CONTRATO_ASSINADO'].includes(statusAtual);

  const handleActionClick = (action) => {
    setPendingAction(action);
    setObservacao('');
    setError('');
    setConfirmOpen(true);
  };

  const handleRejectClick = () => {
    setPendingAction('rejeitar');
    setObservacao('');
    setError('');
    setConfirmOpen(true);
  };

  const handleConfirm = async () => {
    setLoading(true);
    setError('');
    try {
      const token = localStorage.getItem('token');
      const headers = new Headers({
        'Content-Type': 'application/json',
        'Authorization': token ? `Bearer ${token}` : '',
      });

      if (pendingAction === 'rejeitar') {
        const url = `${getActionUrl('solicitacoes_pj', solicitacaoId, 'rejeitar')}?usuarioAnalista=admin&observacao=${encodeURIComponent(observacao)}`;
        await fetchUtils.fetchJson(url, { method: 'POST', headers });
      } else if (pendingAction === 'aprovar') {
        const url = `${getActionUrl('solicitacoes_pj', solicitacaoId, 'aprovar')}?usuarioAnalista=admin&observacao=${encodeURIComponent(observacao)}`;
        await fetchUtils.fetchJson(url, { method: 'POST', headers });
      } else if (pendingAction === 'assinatura-solicitar') {
        const url = getActionUrl('solicitacoes_pj', solicitacaoId, 'assinatura-solicitar');
        await fetchUtils.fetchJson(url, {
          method: 'POST',
          headers,
          body: JSON.stringify({ tipoAssinatura: observacao || 'eletronica' }),
        });
      } else {
        const url = getActionUrl('solicitacoes_pj', solicitacaoId, pendingAction);
        await fetchUtils.fetchJson(url, { method: 'POST', headers });
      }

      setConfirmOpen(false);
      if (onRefresh) onRefresh();
    } catch (e) {
      setError(e.message || 'Erro ao executar ação');
    } finally {
      setLoading(false);
    }
  };

  return (
    <Box>
      <Typography variant="subtitle1" gutterBottom fontWeight="bold">
        Status Atual: {STATUS_LABELS[statusAtual] || statusAtual}
      </Typography>

      <Box sx={{ display: 'flex', gap: 1, flexWrap: 'wrap', mb: 2 }}>
        {actions.map(({ action, label, icon, color }) => (
          <Button
            key={action}
            variant="contained"
            color={color}
            startIcon={icon}
            onClick={() => handleActionClick(action)}
          >
            {label}
          </Button>
        ))}
        {canReject && (
          <Button
            variant="outlined"
            color="error"
            startIcon={<Cancel />}
            onClick={handleRejectClick}
          >
            Rejeitar
          </Button>
        )}
      </Box>

      <Dialog open={confirmOpen} onClose={() => setConfirmOpen(false)} maxWidth="sm" fullWidth>
        <DialogTitle>
          {pendingAction === 'rejeitar' ? 'Rejeitar Solicitação' : `Confirmar: ${pendingAction}`}
        </DialogTitle>
        <DialogContent>
          {error && <Alert severity="error" sx={{ mb: 2 }}>{error}</Alert>}
          <DialogContentText>
            {pendingAction === 'rejeitar'
              ? 'Tem certeza que deseja rejeitar esta solicitação? Informe o motivo:'
              : `Tem certeza que deseja executar a ação "${pendingAction}"?`}
          </DialogContentText>
          {(pendingAction === 'rejeitar' || pendingAction === 'aprovar') && (
            <TextField
              autoFocus
              margin="dense"
              label="Observação"
              fullWidth
              multiline
              rows={3}
              value={observacao}
              onChange={(e) => setObservacao(e.target.value)}
            />
          )}
          {pendingAction === 'assinatura-solicitar' && (
            <TextField
              autoFocus
              margin="dense"
              label="Tipo de Assinatura"
              fullWidth
              value={observacao || 'eletronica'}
              onChange={(e) => setObservacao(e.target.value)}
            />
          )}
        </DialogContent>
        <DialogActions>
          <Button onClick={() => setConfirmOpen(false)}>Cancelar</Button>
          <Button variant="contained" onClick={handleConfirm} disabled={loading}>
            {loading ? 'Executando...' : 'Confirmar'}
          </Button>
        </DialogActions>
      </Dialog>
    </Box>
  );
};
```

- [ ] **Step 2: Verify via lint**

Run: `cd /mnt/c/Users/wende/Projects/aureus-platform/frontend && npm run lint -- --filter=aureus-admin 2>&1 | head -20`

Expected: No errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/aureus-admin/src/components/WorkflowActions.js
git commit -m "feat(admin): add WorkflowActions component for onboarding actions"
```

---

### Task 6: SolicitacaoPJList page

**Files:**
- Create: `pages/SolicitacoesPJ/index.js`
- Create: `pages/SolicitacoesPJ/SolicitacaoPJList.js`

**Interfaces:**
- Produces: `SolicitacaoPJList` component (exported from index.js)

- [ ] **Step 1: Create SolicitacaoPJList.js**

```jsx
import React from 'react';
import {
  List, Datagrid, TextField, DateField, ShowButton,
  Filter, SelectInput, SearchInput, TopToolbar, ExportButton,
} from 'react-admin';

const STATUS_CHOICES = [
  { id: 'RECEBIDA', name: 'Recebida' },
  { id: 'EM_PREENCHIMENTO', name: 'Em Preenchimento' },
  { id: 'CNPJ_CONSULTADO', name: 'CNPJ Consultado' },
  { id: 'SOCIOS_VALIDADOS', name: 'Sócios Validados' },
  { id: 'DOCUMENTOS_PENDENTES', name: 'Documentos Pendentes' },
  { id: 'DOCUMENTOS_ANALISADOS', name: 'Documentos Analisados' },
  { id: 'EM_ANALISE_KYC', name: 'Em Análise KYC' },
  { id: 'KYC_APROVADO', name: 'KYC Aprovado' },
  { id: 'AML_APROVADO', name: 'AML Aprovado' },
  { id: 'COMPLIANCE_APROVADO', name: 'Compliance Aprovado' },
  { id: 'EM_ASSINATURA', name: 'Em Assinatura' },
  { id: 'CONTRATO_ASSINADO', name: 'Contrato Assinado' },
  { id: 'APROVADA', name: 'Aprovada' },
  { id: 'REJEITADA', name: 'Rejeitada' },
];

const FilterBar = (props) => (
  <Filter {...props}>
    <SearchInput source="cnpj" alwaysOn />
    <SelectInput source="status" choices={STATUS_CHOICES} />
  </Filter>
);

export const SolicitacaoPJList = (props) => (
  <List {...props} filters={<FilterBar />} sort={{ field: 'id', order: 'DESC' }}>
    <TopToolbar>
      <ExportButton />
    </TopToolbar>
    <Datagrid>
      <TextField source="id" label="ID" />
      <TextField source="cnpj" label="CNPJ" />
      <TextField source="razaoSocial" label="Razão Social" />
      <TextField source="status" label="Status" />
      <DateField source="dataCriacao" label="Criado" showTime />
      <ShowButton />
    </Datagrid>
  </List>
);
```

- [ ] **Step 2: Create index.js**

```js
export { SolicitacaoPJList } from './SolicitacaoPJList';
export { SolicitacaoPJShow } from './SolicitacaoPJShow';
```

- [ ] **Step 3: Verify via lint**

Run: `cd /mnt/c/Users/wende/Projects/aureus-platform/frontend && npm run lint -- --filter=aureus-admin 2>&1 | head -20`

Expected: No errors (warning about `SolicitacaoPJShow` not yet existing is expected — it will be resolved in Task 7).

- [ ] **Step 4: Commit**

```bash
git add frontend/aureus-admin/src/pages/SolicitacoesPJ/
git commit -m "feat(admin): add SolicitacaoPJList page"
```

---

### Task 7: SolicitacaoPJShow page

**Files:**
- Create: `pages/SolicitacoesPJ/SolicitacaoPJShow.js` (modify existing)

**Interfaces:**
- Consumes: StatusTimeline, DocumentList, SocioList, WorkflowActions
- Produces: `SolicitacaoPJShow` component

- [ ] **Step 1: Create SolicitacaoPJShow.js**

```jsx
import React, { useState } from 'react';
import {
  Show, TabbedShowLayout, Tab, TextField, DateField, NumberField,
  useRecordContext, useRefresh,
} from 'react-admin';
import { Box, Typography, Paper } from '@mui/material';
import { StatusTimeline } from '../../components/StatusTimeline';
import { DocumentList } from '../../components/DocumentList';
import { SocioList } from '../../components/SocioList';
import { WorkflowActions } from '../../components/WorkflowActions';

const DetailsTab = () => {
  const record = useRecordContext();
  if (!record) return null;
  return (
    <Box sx={{ p: 2 }}>
      <TextField source="id" label="ID" />
      <TextField source="cnpj" label="CNPJ" />
      <TextField source="razaoSocial" label="Razão Social" />
      <TextField source="nomeFantasia" label="Nome Fantasia" />
      <TextField source="email" label="Email" />
      <TextField source="telefone" label="Telefone" />
      <TextField source="status" label="Status" />
      <NumberField source="clienteIdCriado" label="Cliente ID" emptyText="-" />
      <NumberField source="contaIdCriada" label="Conta ID" emptyText="-" />
      <TextField source="observacoesAnalista" label="Observações" emptyText="-" />
      <DateField source="dataCriacao" label="Criado em" showTime />
      <DateField source="dataAtualizacao" label="Atualizado em" showTime />
    </Box>
  );
};

const SociosTab = () => {
  const record = useRecordContext();
  const refresh = useRefresh();
  if (!record) return null;
  return <SocioList socios={record.socios || []} solicitacaoId={record.id} onRefresh={refresh} />;
};

const DocumentosTab = () => {
  const record = useRecordContext();
  const refresh = useRefresh();
  if (!record) return null;
  return <DocumentList documentos={record.documentos || []} solicitacaoId={record.id} onRefresh={refresh} />;
};

const HistoricoTab = () => {
  const record = useRecordContext();
  if (!record) return null;
  return <StatusTimeline historico={record.historico || []} statusAtual={record.status} />;
};

const AcoesTab = () => {
  const record = useRecordContext();
  const refresh = useRefresh();
  if (!record) return null;
  return (
    <WorkflowActions
      solicitacaoId={record.id}
      statusAtual={record.status}
      onRefresh={refresh}
    />
  );
};

export const SolicitacaoPJShow = (props) => (
  <Show {...props}>
    <TabbedShowLayout>
      <Tab label="Detalhes">
        <DetailsTab />
      </Tab>
      <Tab label="Sócios" path="socios">
        <SociosTab />
      </Tab>
      <Tab label="Documentos" path="documentos">
        <DocumentosTab />
      </Tab>
      <Tab label="Histórico" path="historico">
        <HistoricoTab />
      </Tab>
      <Tab label="Ações" path="acoes">
        <AcoesTab />
      </Tab>
    </TabbedShowLayout>
  </Show>
);
```

- [ ] **Step 2: Verify via lint**

Run: `cd /mnt/c/Users/wende/Projects/aureus-platform/frontend && npm run lint -- --filter=aureus-admin 2>&1 | head -20`

Expected: No errors.

- [ ] **Step 3: Full build check**

Run: `cd /mnt/c/Users/wende/Projects/aureus-platform/frontend && npm run build -- --filter=aureus-admin 2>&1 | tail -20`

Expected: Build succeeds.

- [ ] **Step 4: Commit**

```bash
git add frontend/aureus-admin/src/pages/SolicitacoesPJ/SolicitacaoPJShow.js
git commit -m "feat(admin): add SolicitacaoPJShow page with tabs and workflow actions"
```
