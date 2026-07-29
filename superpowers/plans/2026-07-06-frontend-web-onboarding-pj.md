# Frontend Web Onboarding PF/PJ Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite `aurix-web` Onboarding page with integrated PF/PJ selector, multi-step PJ wizard, and post-creation tracking.

**Architecture:** Container component (`Onboarding.js`) with 3-mode state machine (`select → form → tracking`). PF gets a simple single-page form; PJ gets a 4-step MUI Stepper wizard that creates the solicitation in Step 1 and augments it with socios/documentos in subsequent steps. API calls through `apiService` (axios), no raw fetch.

**Tech Stack:** React 18, MUI 5, react-router-dom 6, axios, date-fns, react-dropzone (already in deps)

## Global Constraints

- `user` prop pattern: all pages receive `user` from App.js, no React Context
- Form state: plain `useState` (no react-hook-form despite being in package.json)
- API calls: `apiService` axios instance, not raw `fetch`
- Portuguese labels throughout (same as existing pages)
- Currency/number: `Intl.NumberFormat('pt-BR')` or `numeral`
- Date format: `date-fns` with `ptBR` locale
- Loading: MUI CircularProgress or Skeleton
- Error: MUI Alert (same pattern as Login.js)
- No TypeScript — all files are `.js`
- Default export for pages (`export default Onboarding`), but components within `components/Onboarding/` can use named exports

---
## File Structure

```
Modified:
  src/services/apiService.js                    +7 onboarding methods
  src/services/apiService.test.js               +tests for new methods
  src/pages/Onboarding.js                        rewrite (container + state machine)
  src/pages/Onboarding.test.js                   rewrite tests
  src/components/Onboarding/                     NEW directory

Created:
  src/components/Onboarding/TipoSelector.js      PF/PJ selection cards
  src/components/Onboarding/TipoSelector.test.js
  src/components/Onboarding/FormPF.js            PF single form
  src/components/Onboarding/FormPF.test.js
  src/components/Onboarding/FormPJ/WizardPJ.js   MUI Stepper container
  src/components/Onboarding/FormPJ/StepEmpresa.js
  src/components/Onboarding/FormPJ/StepSocios.js
  src/components/Onboarding/FormPJ/StepDocumentos.js
  src/components/Onboarding/FormPJ/StepRevisao.js
  src/components/Onboarding/FormPJ/WizardPJ.test.js
  src/components/Onboarding/TrackingPJ.js        post-creation status view
  src/components/Onboarding/TrackingPJ.test.js
```

---
### Task 1: API Service — Onboarding Methods

**Files:**
- Modify: `src/services/apiService.js`
- Modify: `src/services/apiService.test.js`

**Interfaces:**
- Produces: 8 new methods on `apiService` (all return axios Promise)

```js
criarSolicitacaoPF(data)           → POST   /onboarding/contas/pf/solicitacoes
getSolicitacaoPF(id)               → GET    /onboarding/contas/pf/solicitacoes/{id}
criarSolicitacaoPJ(data)           → POST   /onboarding/contas/pj
getSolicitacaoPJ(id)               → GET    /onboarding/contas/pj/{id}
validarCNPJPJ(id)                  → POST   /onboarding/contas/pj/{id}/validar-cnpj
adicionarSocioPJ(id, socio)        → POST   /onboarding/contas/pj/{id}/socios
removerSocioPJ(id, socioId)        → DELETE /onboarding/contas/pj/{id}/socios/{socioId}
adicionarDocumentoPJ(id, doc)      → POST   /onboarding/contas/pj/{id}/documentos
```

- [ ] **Step 1: Add methods to `apiService.js`**

After the `pagarFatura` method (inside the `apiService` object, before the closing `};`), add:

```js
  async criarSolicitacaoPF(data) {
    const response = await api.post('/onboarding/contas/pf/solicitacoes', data);
    return response.data;
  },

  async getSolicitacaoPF(id) {
    const response = await api.get(`/onboarding/contas/pf/solicitacoes/${id}`);
    return response.data;
  },

  async criarSolicitacaoPJ(data) {
    const response = await api.post('/onboarding/contas/pj', data);
    return response.data;
  },

  async getSolicitacaoPJ(id) {
    const response = await api.get(`/onboarding/contas/pj/${id}`);
    return response.data;
  },

  async validarCNPJPJ(id) {
    const response = await api.post(`/onboarding/contas/pj/${id}/validar-cnpj`);
    return response.data;
  },

  async adicionarSocioPJ(id, socio) {
    await api.post(`/onboarding/contas/pj/${id}/socios`, socio);
  },

  async removerSocioPJ(id, socioId) {
    await api.delete(`/onboarding/contas/pj/${id}/socios/${socioId}`);
  },

  async adicionarDocumentoPJ(id, doc) {
    await api.post(`/onboarding/contas/pj/${id}/documentos`, doc);
  },
```

- [ ] **Step 2: Add tests to `apiService.test.js`**

Add after existing tests:

```js
import api from './apiService';

// Note: axios is already mocked via jest.mock('axios') at top of file

describe('Onboarding API', () => {
  const data = { cpf: '12345678901', nome: 'Teste', email: 'teste@test.com' };
  const socio = { cpf: '98765432100', nome: 'Socio', tipo: 'SOCIO' };
  const doc = { tipoDocumento: 'CONTRATO_SOCIAL', nomeArquivo: 'contrato.pdf', urlStorage: 'https://storage.com/doc.pdf' };

  test('criarSolicitacaoPF', async () => {
    axios.post.mockResolvedValue({ data: { id: 1 } });
    const res = await api.criarSolicitacaoPF(data);
    expect(axios.post).toHaveBeenCalledWith('/onboarding/contas/pf/solicitacoes', data, expect.any(Object));
    expect(res.id).toBe(1);
  });

  test('getSolicitacaoPF', async () => {
    axios.get.mockResolvedValue({ data: { id: 1, status: 'EM_ANALISE' } });
    const res = await api.getSolicitacaoPF(1);
    expect(axios.get).toHaveBeenCalledWith('/onboarding/contas/pf/solicitacoes/1', expect.any(Object));
    expect(res.status).toBe('EM_ANALISE');
  });

  test('criarSolicitacaoPJ', async () => {
    axios.post.mockResolvedValue({ data: { id: 10, cnpj: '12345678000190' } });
    const res = await api.criarSolicitacaoPJ(data);
    expect(axios.post).toHaveBeenCalledWith('/onboarding/contas/pj', data, expect.any(Object));
    expect(res.id).toBe(10);
  });

  test('getSolicitacaoPJ', async () => {
    axios.get.mockResolvedValue({ data: { id: 10, status: 'EM_PREENCHIMENTO' } });
    const res = await api.getSolicitacaoPJ(10);
    expect(axios.get).toHaveBeenCalledWith('/onboarding/contas/pj/10', expect.any(Object));
    expect(res.status).toBe('EM_PREENCHIMENTO');
  });

  test('validarCNPJPJ', async () => {
    axios.post.mockResolvedValue({ data: { id: 10, status: 'CNPJ_CONSULTADO' } });
    const res = await api.validarCNPJPJ(10);
    expect(axios.post).toHaveBeenCalledWith('/onboarding/contas/pj/10/validar-cnpj', undefined, expect.any(Object));
    expect(res.status).toBe('CNPJ_CONSULTADO');
  });

  test('adicionarSocioPJ', async () => {
    axios.post.mockResolvedValue({});
    await api.adicionarSocioPJ(10, socio);
    expect(axios.post).toHaveBeenCalledWith('/onboarding/contas/pj/10/socios', socio, expect.any(Object));
  });

  test('removerSocioPJ', async () => {
    axios.delete.mockResolvedValue({});
    await api.removerSocioPJ(10, 5);
    expect(axios.delete).toHaveBeenCalledWith('/onboarding/contas/pj/10/socios/5', expect.any(Object));
  });

  test('adicionarDocumentoPJ', async () => {
    axios.post.mockResolvedValue({});
    await api.adicionarDocumentoPJ(10, doc);
    expect(axios.post).toHaveBeenCalledWith('/onboarding/contas/pj/10/documentos', doc, expect.any(Object));
  });
});
```

- [ ] **Step 3: Run tests to verify they pass**

Run: `cd frontend/aurix-web && npx react-scripts test --watchAll=false --testPathPattern=apiService`
Expected: all 8 new tests pass

- [ ] **Step 4: Commit**

```bash
git add frontend/aurix-web/src/services/apiService.js frontend/aurix-web/src/services/apiService.test.js
git commit -m "feat(aurix-web): add onboarding API methods to apiService"
```

---
### Task 2: TipoSelector + FormPF Components

**Files:**
- Create: `src/components/Onboarding/TipoSelector.js`
- Create: `src/components/Onboarding/TipoSelector.test.js`
- Create: `src/components/Onboarding/FormPF.js`
- Create: `src/components/Onboarding/FormPF.test.js`

**Interfaces:**
- Consumes: `apiService.criarSolicitacaoPF`, `apiService.getSolicitacaoPF`
- Produces: `TipoSelector({ onSelect: (tipo) => void })`, `FormPF({ onComplete: (id) => void })`

- [ ] **Step 1: Create `TipoSelector.js`**

```js
import React from 'react';
import { Box, Card, CardContent, Typography } from '@mui/material';
import { Person, Business } from '@mui/icons-material';

function TipoSelector({ onSelect }) {
  const options = [
    { tipo: 'PF', icon: <Person sx={{ fontSize: 48 }} />, title: 'Pessoa Física', desc: 'Conta pessoal' },
    { tipo: 'PJ', icon: <Business sx={{ fontSize: 48 }} />, title: 'Pessoa Jurídica', desc: 'Conta empresarial' },
  ];

  return (
    <Box>
      <Typography variant="h5" gutterBottom>Abertura de conta</Typography>
      <Typography variant="body2" color="text.secondary" sx={{ mb: 3 }}>
        Selecione o tipo de conta que deseja abrir
      </Typography>
      <Box sx={{ display: 'flex', gap: 3, flexWrap: 'wrap' }}>
        {options.map((opt) => (
          <Card
            key={opt.tipo}
            sx={{
              flex: '1 1 280px', cursor: 'pointer', transition: '0.2s',
              '&:hover': { transform: 'translateY(-4px)', boxShadow: 4 }
            }}
            onClick={() => onSelect(opt.tipo)}
          >
            <CardContent sx={{ textAlign: 'center', py: 4 }}>
              {opt.icon}
              <Typography variant="h6" sx={{ mt: 2 }}>{opt.title}</Typography>
              <Typography variant="body2" color="text.secondary">{opt.desc}</Typography>
            </CardContent>
          </Card>
        ))}
      </Box>
    </Box>
  );
}

export default TipoSelector;
```

- [ ] **Step 2: Create `TipoSelector.test.js`**

```js
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import TipoSelector from './TipoSelector';

test('renders PF and PJ cards', () => {
  const onSelect = jest.fn();
  render(<TipoSelector onSelect={onSelect} />);
  expect(screen.getByText('Pessoa Física')).toBeInTheDocument();
  expect(screen.getByText('Pessoa Jurídica')).toBeInTheDocument();
});

test('calls onSelect with PF when PF card clicked', () => {
  const onSelect = jest.fn();
  render(<TipoSelector onSelect={onSelect} />);
  fireEvent.click(screen.getByText('Pessoa Física'));
  expect(onSelect).toHaveBeenCalledWith('PF');
});

test('calls onSelect with PJ when PJ card clicked', () => {
  const onSelect = jest.fn();
  render(<TipoSelector onSelect={onSelect} />);
  fireEvent.click(screen.getByText('Pessoa Jurídica'));
  expect(onSelect).toHaveBeenCalledWith('PJ');
});
```

- [ ] **Step 3: Create `FormPF.js`**

```js
import React, { useState } from 'react';
import { Box, Card, CardContent, Typography, TextField, Button, Alert } from '@mui/material';
import { apiService } from '../../services/apiService';

const formatCPF = (value) => value.replace(/\D/g, '').replace(/(\d{3})(\d)/, '$1.$2').replace(/(\d{3})(\d)/, '$1.$2').replace(/(\d{3})(\d{1,2})/, '$1-$2').substr(0, 14);

const formatTelefone = (value) => value.replace(/\D/g, '').replace(/(\d{2})(\d)/, '($1) $2').replace(/(\d{5})(\d)/, '$1-$2').substr(0, 15);

function FormPF({ onComplete }) {
  const [form, setForm] = useState({ cpf: '', nome: '', email: '', telefone: '', dataNascimento: '', ocupacao: '', rendaDeclarada: '', endereco: '' });
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  const handleChange = (field) => (e) => setForm({ ...form, [field]: e.target.value });

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!form.cpf || !form.nome || !form.email) {
      setError('CPF, Nome e E-mail são obrigatórios');
      return;
    }
    setLoading(true);
    setError('');
    try {
      const payload = {
        cpf: form.cpf.replace(/\D/g, ''),
        nome: form.nome,
        email: form.email,
        telefone: form.telefone.replace(/\D/g, ''),
        dataNascimento: form.dataNascimento || null,
        ocupacao: form.ocupacao || null,
        rendaDeclarada: form.rendaDeclarada ? Number(form.rendaDeclarada) : null,
        endereco: form.endereco || null,
      };
      const res = await apiService.criarSolicitacaoPF(payload);
      onComplete(res.data.id);
    } catch (err) {
      setError(err.response?.data?.message || err.message || 'Erro ao enviar solicitação');
    } finally {
      setLoading(false);
    }
  };

  return (
    <Box>
      <Typography variant="h5" gutterBottom>Abertura de conta PF</Typography>
      <Card>
        <CardContent>
          {error && <Alert severity="error" sx={{ mb: 2 }}>{error}</Alert>}
          <form onSubmit={handleSubmit}>
            <TextField fullWidth label="CPF" value={form.cpf} onChange={(e) => setForm({ ...form, cpf: formatCPF(e.target.value) })} margin="normal" required />
            <TextField fullWidth label="Nome completo" value={form.nome} onChange={handleChange('nome')} margin="normal" required />
            <TextField fullWidth label="E-mail" type="email" value={form.email} onChange={handleChange('email')} margin="normal" required />
            <TextField fullWidth label="Telefone" value={form.telefone} onChange={(e) => setForm({ ...form, telefone: formatTelefone(e.target.value) })} margin="normal" />
            <TextField fullWidth label="Data de nascimento" type="date" value={form.dataNascimento} onChange={handleChange('dataNascimento')} margin="normal" InputLabelProps={{ shrink: true }} />
            <TextField fullWidth label="Ocupação" value={form.ocupacao} onChange={handleChange('ocupacao')} margin="normal" />
            <TextField fullWidth label="Renda declarada (R$)" type="number" value={form.rendaDeclarada} onChange={handleChange('rendaDeclarada')} margin="normal" />
            <TextField fullWidth label="Endereço" value={form.endereco} onChange={handleChange('endereco')} margin="normal" multiline rows={2} />
            <Box sx={{ display: 'flex', gap: 2, mt: 2 }}>
              <Button type="submit" variant="contained" disabled={loading}>Solicitar abertura</Button>
            </Box>
          </form>
        </CardContent>
      </Card>
    </Box>
  );
}

export default FormPF;
```

- [ ] **Step 4: Create `FormPF.test.js`**

```js
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import FormPF from './FormPF';
import { apiService } from '../../services/apiService';

jest.mock('../../services/apiService');

test('renders PF form fields', () => {
  render(<FormPF onComplete={jest.fn()} />);
  expect(screen.getByLabelText('CPF')).toBeInTheDocument();
  expect(screen.getByLabelText('Nome completo')).toBeInTheDocument();
  expect(screen.getByLabelText('E-mail')).toBeInTheDocument();
});

test('submits form successfully', async () => {
  apiService.criarSolicitacaoPF.mockResolvedValue({ data: { id: 42 } });
  const onComplete = jest.fn();
  render(<FormPF onComplete={onComplete} />);
  fireEvent.change(screen.getByLabelText('CPF'), { target: { value: '123.456.789-00' } });
  fireEvent.change(screen.getByLabelText('Nome completo'), { target: { value: 'João' } });
  fireEvent.change(screen.getByLabelText('E-mail'), { target: { value: 'joao@test.com' } });
  fireEvent.submit(screen.getByRole('button', { name: /solicitar/i }));
  await waitFor(() => expect(onComplete).toHaveBeenCalledWith(42));
});

test('shows error on missing required fields', () => {
  render(<FormPF onComplete={jest.fn()} />);
  fireEvent.submit(screen.getByRole('button', { name: /solicitar/i }));
  expect(screen.getByText(/obrigatórios/i)).toBeInTheDocument();
});
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd frontend/aurix-web && npx react-scripts test --watchAll=false --testPathPattern="TipoSelector|FormPF"`
Expected: all 6 tests pass

- [ ] **Step 6: Commit**

```bash
git add frontend/aurix-web/src/components/Onboarding/
git commit -m "feat(aurix-web): add TipoSelector and FormPF components"
```

---
### Task 3: WizardPJ — Multi-Step Wizard

**Files:**
- Create: `src/components/Onboarding/FormPJ/WizardPJ.js`
- Create: `src/components/Onboarding/FormPJ/StepEmpresa.js`
- Create: `src/components/Onboarding/FormPJ/StepSocios.js`
- Create: `src/components/Onboarding/FormPJ/StepDocumentos.js`
- Create: `src/components/Onboarding/FormPJ/StepRevisao.js`
- Create: `src/components/Onboarding/FormPJ/WizardPJ.test.js`

**Interfaces:**
- Consumes: `apiService.criarSolicitacaoPJ`, `apiService.adicionarSocioPJ`, `apiService.removerSocioPJ`, `apiService.adicionarDocumentoPJ`
- Produces: `WizardPJ({ onComplete: (id) => void })`

**Flow:** Step 1 (Empresa) POSTs to create the solicitation → gets `id` → Steps 2-3 augment with socios/docs → Step 4 shows summary + "Acompanhar" button

- [ ] **Step 1: Create `StepEmpresa.js`**

```js
import React, { useState } from 'react';
import { TextField, Alert } from '@mui/material';

const formatCNPJ = (value) => value.replace(/\D/g, '').replace(/^(\d{2})(\d)/, '$1.$2').replace(/^(\d{2})\.(\d{3})(\d)/, '$1.$2.$3').replace(/\.(\d{3})(\d)/, '.$1/$2').replace(/(\d{4})(\d)/, '$1-$2').substr(0, 18);

function StepEmpresa({ form, setForm, error }) {
  const handleChange = (field) => (e) => setForm({ ...form, [field]: e.target.value });

  return (
    <>
      {error && <Alert severity="error" sx={{ mb: 2 }}>{error}</Alert>}
      <TextField fullWidth label="CNPJ" value={form.cnpj || ''} onChange={(e) => setForm({ ...form, cnpj: formatCNPJ(e.target.value) })} margin="normal" required />
      <TextField fullWidth label="Razão Social" value={form.razaoSocial || ''} onChange={handleChange('razaoSocial')} margin="normal" required />
      <TextField fullWidth label="Nome Fantasia" value={form.nomeFantasia || ''} onChange={handleChange('nomeFantasia')} margin="normal" />
      <TextField fullWidth label="E-mail" type="email" value={form.email || ''} onChange={handleChange('email')} margin="normal" required />
      <TextField fullWidth label="Telefone" value={form.telefone || ''} onChange={(e) => setForm({ ...form, telefone: e.target.value.replace(/\D/g, '').substr(0, 11) })} margin="normal" />
      <TextField fullWidth label="Endereço" value={form.endereco || ''} onChange={handleChange('endereco')} margin="normal" multiline rows={2} />
    </>
  );
}

export default StepEmpresa;
```

- [ ] **Step 2: Create `StepSocios.js`**

```js
import React, { useState } from 'react';
import { Box, Typography, Button, Table, TableBody, TableCell, TableContainer, TableHead, TableRow, Paper, Dialog, DialogTitle, DialogContent, DialogActions, TextField, IconButton, Alert } from '@mui/material';
import { Delete, Add } from '@mui/icons-material';
import { apiService } from '../../../services/apiService';

function StepSocios({ solicitacaoId, onSocioChange }) {
  const [socios, setSocios] = useState([]);
  const [dialogOpen, setDialogOpen] = useState(false);
  const [socioForm, setSocioForm] = useState({ cpf: '', nome: '', email: '', telefone: '', dataNascimento: '', nacionalidade: '', qualificacao: '', percentualParticipacao: '' });
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  const handleAdd = async () => {
    if (!socioForm.cpf || !socioForm.nome) { setError('CPF e Nome são obrigatórios'); return; }
    setLoading(true);
    setError('');
    try {
      const payload = {
        tipo: 'SOCIO',
        cpf: socioForm.cpf.replace(/\D/g, ''),
        nome: socioForm.nome,
        email: socioForm.email || null,
        telefone: socioForm.telefone?.replace(/\D/g, '') || null,
        dataNascimento: socioForm.dataNascimento || null,
        nacionalidade: socioForm.nacionalidade || null,
        qualificacao: socioForm.qualificacao || null,
        percentualParticipacao: socioForm.percentualParticipacao ? Number(socioForm.percentualParticipacao) : null,
      };
      await apiService.adicionarSocioPJ(solicitacaoId, payload);
      const newList = [...socios, payload];
      setSocios(newList);
      if (onSocioChange) onSocioChange(newList.length);
      setDialogOpen(false);
      setSocioForm({ cpf: '', nome: '', email: '', telefone: '', dataNascimento: '', nacionalidade: '', qualificacao: '', percentualParticipacao: '' });
    } catch (err) {
      setError(err.response?.data?.message || 'Erro ao adicionar sócio');
    } finally {
      setLoading(false);
    }
  };

  const handleRemove = async (idx, cpf) => {
    setError('');
    const socio = socios[idx];
    if (socio.id) {
      try { await apiService.removerSocioPJ(solicitacaoId, socio.id); } catch {}
    }
    const filtered = socios.filter((_, i) => i !== idx);
    setSocios(filtered);
    if (onSocioChange) onSocioChange(filtered.length);
  };

  return (
    <Box>
      <Typography variant="subtitle1" gutterBottom>Sócios</Typography>
      {error && <Alert severity="error" sx={{ mb: 2 }}>{error}</Alert>}
      <Button variant="outlined" startIcon={<Add />} onClick={() => setDialogOpen(true)} sx={{ mb: 2 }}>
        Adicionar sócio
      </Button>
      {socios.length > 0 && (
        <TableContainer component={Paper} variant="outlined">
          <Table size="small">
            <TableHead>
              <TableRow>
                <TableCell>Nome</TableCell>
                <TableCell>CPF</TableCell>
                <TableCell>Participação</TableCell>
                <TableCell align="right">Ações</TableCell>
              </TableRow>
            </TableHead>
            <TableBody>
              {socios.map((s, i) => (
                <TableRow key={i}>
                  <TableCell>{s.nome}</TableCell>
                  <TableCell>{s.cpf}</TableCell>
                  <TableCell>{s.percentualParticipacao ? `${s.percentualParticipacao}%` : '-'}</TableCell>
                  <TableCell align="right">
                    <IconButton size="small" color="error" onClick={() => handleRemove(i, s.cpf)}><Delete /></IconButton>
                  </TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </TableContainer>
      )}
      {socios.length === 0 && <Typography variant="body2" color="text.secondary">Nenhum sócio adicionado</Typography>}

      <Dialog open={dialogOpen} onClose={() => setDialogOpen(false)} maxWidth="sm" fullWidth>
        <DialogTitle>Adicionar sócio</DialogTitle>
        <DialogContent>
          <TextField fullWidth label="CPF" value={socioForm.cpf} onChange={(e) => setSocioForm({ ...socioForm, cpf: e.target.value.replace(/\D/g, '').substr(0, 11) })} margin="dense" required />
          <TextField fullWidth label="Nome" value={socioForm.nome} onChange={(e) => setSocioForm({ ...socioForm, nome: e.target.value })} margin="dense" required />
          <TextField fullWidth label="E-mail" value={socioForm.email} onChange={(e) => setSocioForm({ ...socioForm, email: e.target.value })} margin="dense" />
          <TextField fullWidth label="Telefone" value={socioForm.telefone} onChange={(e) => setSocioForm({ ...socioForm, telefone: e.target.value.replace(/\D/g, '').substr(0, 11) })} margin="dense" />
          <TextField fullWidth label="Data de nascimento" type="date" value={socioForm.dataNascimento} onChange={(e) => setSocioForm({ ...socioForm, dataNascimento: e.target.value })} margin="dense" InputLabelProps={{ shrink: true }} />
          <TextField fullWidth label="Nacionalidade" value={socioForm.nacionalidade} onChange={(e) => setSocioForm({ ...socioForm, nacionalidade: e.target.value })} margin="dense" />
          <TextField fullWidth label="Qualificação" value={socioForm.qualificacao} onChange={(e) => setSocioForm({ ...socioForm, qualificacao: e.target.value })} margin="dense" />
          <TextField fullWidth label="% Participação" type="number" value={socioForm.percentualParticipacao} onChange={(e) => setSocioForm({ ...socioForm, percentualParticipacao: e.target.value })} margin="dense" />
        </DialogContent>
        <DialogActions>
          <Button onClick={() => setDialogOpen(false)}>Cancelar</Button>
          <Button onClick={handleAdd} variant="contained" disabled={loading}>Adicionar</Button>
        </DialogActions>
      </Dialog>
    </Box>
  );
}

export default StepSocios;
```

- [ ] **Step 3: Create `StepDocumentos.js`**

```js
import React, { useState } from 'react';
import { Box, Typography, Button, Table, TableBody, TableCell, TableContainer, TableHead, TableRow, Paper, Alert, Chip } from '@mui/material';
import { CloudUpload } from '@mui/icons-material';
import { apiService } from '../../../services/apiService';

function StepDocumentos({ solicitacaoId, onDocumentoChange }) {
  const [documentos, setDocumentos] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  const tipos = [
    { value: 'CONTRATO_SOCIAL', label: 'Contrato Social' },
    { value: 'CNPJ', label: 'Cartão CNPJ' },
    { value: 'IDENTIDADE', label: 'RG / CNH' },
    { value: 'COMPROVANTE_ENDERECO', label: 'Comprovante de Endereço' },
    { value: 'OUTRO', label: 'Outro' },
  ];

  const handleUpload = async (tipoDocumento) => {
    const nomeArquivo = `${tipoDocumento}_${Date.now()}.pdf`;
    const urlStorage = `https://storage.aurix.com/documents/${solicitacaoId}/${nomeArquivo}`;
    setLoading(true);
    setError('');
    try {
      await apiService.adicionarDocumentoPJ(solicitacaoId, { tipoDocumento, nomeArquivo, urlStorage });
      const newList = [...documentos, { tipoDocumento, nomeArquivo, validado: false }];
      setDocumentos(newList);
      if (onDocumentoChange) onDocumentoChange(newList.length);
    } catch (err) {
      setError('Erro ao enviar documento');
    } finally {
      setLoading(false);
    }
  };

  return (
    <Box>
      <Typography variant="subtitle1" gutterBottom>Documentos</Typography>
      {error && <Alert severity="error" sx={{ mb: 2 }}>{error}</Alert>}
      <Box sx={{ display: 'flex', gap: 1, flexWrap: 'wrap', mb: 2 }}>
        {tipos.map((t) => (
          <Button key={t.value} variant="outlined" size="small" startIcon={<CloudUpload />} onClick={() => handleUpload(t.value)} disabled={loading}>
            {t.label}
          </Button>
        ))}
      </Box>
      {documentos.length > 0 && (
        <TableContainer component={Paper} variant="outlined">
          <Table size="small">
            <TableHead>
              <TableRow>
                <TableCell>Tipo</TableCell>
                <TableCell>Arquivo</TableCell>
                <TableCell>Status</TableCell>
              </TableRow>
            </TableHead>
            <TableBody>
              {documentos.map((d, i) => (
                <TableRow key={i}>
                  <TableCell>{tipos.find((t) => t.value === d.tipoDocumento)?.label || d.tipoDocumento}</TableCell>
                  <TableCell>{d.nomeArquivo}</TableCell>
                  <TableCell><Chip size="small" label={d.validado ? 'Validado' : 'Pendente'} color={d.validado ? 'success' : 'default'} /></TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </TableContainer>
      )}
      {documentos.length === 0 && <Typography variant="body2" color="text.secondary">Nenhum documento enviado</Typography>}
    </Box>
  );
}

export default StepDocumentos;
```

- [ ] **Step 4: Create `StepRevisao.js`**

```js
import React from 'react';
import { Box, Typography, Table, TableBody, TableCell, TableRow, Button } from '@mui/material';

function StepRevisao({ form, sociosCount, documentosCount, solicitacaoId }) {
  const formatCNPJ = (v) => v ? v.replace(/\D/g, '').replace(/^(\d{2})(\d{3})(\d{3})(\d{4})(\d{2})$/, '$1.$2.$3/$4-$5') : '-';

  const rows = [
    ['CNPJ', formatCNPJ(form.cnpj)],
    ['Razão Social', form.razaoSocial],
    ['Nome Fantasia', form.nomeFantasia || '-'],
    ['E-mail', form.email],
    ['Telefone', form.telefone || '-'],
    ['Endereço', form.endereco || '-'],
    ['Sócios', `${sociosCount} adicionado(s)`],
    ['Documentos', `${documentosCount} enviado(s)`],
  ];

  return (
    <Box>
      <Typography variant="subtitle1" gutterBottom>Revisar dados</Typography>
      <Table size="small">
        <TableBody>
          {rows.map(([label, value]) => (
            <TableRow key={label}>
              <TableCell sx={{ fontWeight: 600, width: 200 }}>{label}</TableCell>
              <TableCell>{value}</TableCell>
            </TableRow>
          ))}
        </TableBody>
      </Table>
      <Typography variant="body2" color="text.secondary" sx={{ mt: 2 }}>
        Solicitação #{solicitacaoId} criada com sucesso. Você pode adicionar mais sócios e documentos depois.
      </Typography>
    </Box>
  );
}

export default StepRevisao;
```

- [ ] **Step 5: Create `WizardPJ.js`**

```js
import React, { useState } from 'react';
import { Box, Stepper, Step, StepLabel, Typography, Button, Card, CardContent, Alert } from '@mui/material';
import StepEmpresa from './StepEmpresa';
import StepSocios from './StepSocios';
import StepDocumentos from './StepDocumentos';
import StepRevisao from './StepRevisao';
import { apiService } from '../../../services/apiService';

const steps = ['Dados da Empresa', 'Sócios', 'Documentos', 'Revisão'];

function WizardPJ({ onComplete }) {
  const [activeStep, setActiveStep] = useState(0);
  const [solicitacaoId, setSolicitacaoId] = useState(null);
  const [form, setForm] = useState({});
  const [sociosCount, setSociosCount] = useState(0);
  const [documentosCount, setDocumentosCount] = useState(0);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  const handleNext = async () => {
    if (activeStep === 0) {
      if (!form.cnpj || !form.razaoSocial || !form.email) {
        setError('CNPJ, Razão Social e E-mail são obrigatórios');
        return;
      }
      setError('');
      setLoading(true);
      try {
        const payload = {
          cnpj: form.cnpj.replace(/\D/g, ''),
          razaoSocial: form.razaoSocial,
          nomeFantasia: form.nomeFantasia || null,
          email: form.email,
          telefone: form.telefone?.replace(/\D/g, '') || null,
          endereco: form.endereco || null,
        };
        const res = await apiService.criarSolicitacaoPJ(payload);
        setSolicitacaoId(res.data.id);
        setActiveStep(1);
      } catch (err) {
        setError(err.response?.data?.message || 'Erro ao criar solicitação');
      } finally {
        setLoading(false);
      }
    } else {
      setActiveStep((prev) => prev + 1);
    }
  };

  const handleFinish = () => {
    onComplete(solicitacaoId);
  };

  const getStepContent = (step) => {
    switch (step) {
      case 0: return <StepEmpresa form={form} setForm={setForm} error={error} />;
      case 1: return <StepSocios solicitacaoId={solicitacaoId} onSocioChange={setSociosCount} />;
      case 2: return <StepDocumentos solicitacaoId={solicitacaoId} onDocumentoChange={setDocumentosCount} />;
      case 3: return <StepRevisao form={form} sociosCount={sociosCount} documentosCount={documentosCount} solicitacaoId={solicitacaoId} />;
      default: return null;
    }
  };

  return (
    <Box>
      <Typography variant="h5" gutterBottom>Abertura de conta PJ</Typography>
      <Stepper activeStep={activeStep} sx={{ mb: 3 }}>
        {steps.map((label) => <Step key={label}><StepLabel>{label}</StepLabel></Step>)}
      </Stepper>
      <Card>
        <CardContent>
          {getStepContent(activeStep)}
          <Box sx={{ display: 'flex', justifyContent: 'space-between', mt: 3 }}>
            <Button disabled={activeStep === 0} onClick={() => setActiveStep((prev) => prev - 1)}>Anterior</Button>
            {activeStep < 3 ? (
              <Button variant="contained" onClick={handleNext} disabled={loading}>
                {activeStep === 0 ? (loading ? 'Criando...' : 'Criar solicitação') : 'Próximo'}
              </Button>
            ) : (
              <Button variant="contained" color="success" onClick={handleFinish}>
                Ver acompanhamento
              </Button>
            )}
          </Box>
        </CardContent>
      </Card>
    </Box>
  );
}

export default WizardPJ;
```

- [ ] **Step 6: Create `WizardPJ.test.js`**

```js
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import WizardPJ from './WizardPJ';
import { apiService } from '../../../services/apiService';

jest.mock('../../../services/apiService');

test('renders step 1: empresa form', () => {
  render(<WizardPJ onComplete={jest.fn()} />);
  expect(screen.getByText('Dados da Empresa')).toBeInTheDocument();
  expect(screen.getByLabelText('CNPJ')).toBeInTheDocument();
  expect(screen.getByLabelText('Razão Social')).toBeInTheDocument();
});

test('submits empresa and advances to socios step', async () => {
  apiService.criarSolicitacaoPJ.mockResolvedValue({ data: { id: 99 } });
  render(<WizardPJ onComplete={jest.fn()} />);
  fireEvent.change(screen.getByLabelText('CNPJ'), { target: { value: '12.345.678/0001-90' } });
  fireEvent.change(screen.getByLabelText('Razão Social'), { target: { value: 'Empresa Ltda' } });
  fireEvent.change(screen.getByLabelText('E-mail'), { target: { value: 'contato@empresa.com' } });
  fireEvent.click(screen.getByRole('button', { name: /criar solicitação/i }));
  await waitFor(() => expect(screen.getByText('Sócios')).toBeInTheDocument());
  expect(apiService.criarSolicitacaoPJ).toHaveBeenCalled();
});

test('shows error when empresa required fields missing', () => {
  render(<WizardPJ onComplete={jest.fn()} />);
  fireEvent.click(screen.getByRole('button', { name: /criar solicitação/i }));
  expect(screen.getByText(/obrigatórios/i)).toBeInTheDocument();
});

test('full wizard flow to completion', async () => {
  apiService.criarSolicitacaoPJ.mockResolvedValue({ data: { id: 50 } });
  const onComplete = jest.fn();
  render(<WizardPJ onComplete={onComplete} />);

  // Step 0: empresa
  fireEvent.change(screen.getByLabelText('CNPJ'), { target: { value: '12.345.678/0001-90' } });
  fireEvent.change(screen.getByLabelText('Razão Social'), { target: { value: 'Empresa Ltda' } });
  fireEvent.change(screen.getByLabelText('E-mail'), { target: { value: 'c@e.com' } });
  fireEvent.click(screen.getByRole('button', { name: /criar solicitação/i }));
  await waitFor(() => expect(screen.getByText('Sócios')).toBeInTheDocument());

  // Step 1: socios (skip)
  fireEvent.click(screen.getByRole('button', { name: /próximo/i }));
  await waitFor(() => expect(screen.getByText('Documentos')).toBeInTheDocument());

  // Step 2: documentos (skip)
  fireEvent.click(screen.getByRole('button', { name: /próximo/i }));
  await waitFor(() => expect(screen.getByText('Revisão')).toBeInTheDocument());

  // Step 3: revisao → finish
  fireEvent.click(screen.getByRole('button', { name: /ver acompanhamento/i }));
  expect(onComplete).toHaveBeenCalledWith(50);
});
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `cd frontend/aurix-web && npx react-scripts test --watchAll=false --testPathPattern="WizardPJ"`
Expected: all 3 tests pass

- [ ] **Step 8: Commit**

```bash
git add frontend/aurix-web/src/components/Onboarding/FormPJ/
git commit -m "feat(aurix-web): add PJ onboarding wizard with 4 steps"
```

---
### Task 4: TrackingPJ Component

**Files:**
- Create: `src/components/Onboarding/TrackingPJ.js`
- Create: `src/components/Onboarding/TrackingPJ.test.js`

**Interfaces:**
- Consumes: `apiService.getSolicitacaoPJ`
- Produces: `TrackingPJ({ solicitacaoId, onNew })`

- [ ] **Step 1: Create `TrackingPJ.js`**

```js
import React, { useState, useEffect } from 'react';
import { Box, Typography, Card, CardContent, Chip, Table, TableBody, TableCell, TableRow, Alert, CircularProgress, Button, Divider } from '@mui/material';
import { CheckCircle, HourglassEmpty, Cancel, ArrowBack } from '@mui/icons-material';
import { format } from 'date-fns';
import { ptBR } from 'date-fns/locale';
import { apiService } from '../../services/apiService';

const statusColors = {
  EM_PREENCHIMENTO: 'default',
  CNPJ_CONSULTADO: 'info',
  SOCIOS_VALIDADOS: 'info',
  DOCUMENTOS_ANALISADOS: 'info',
  AML_APROVADO: 'info',
  COMPLIANCE_APROVADO: 'info',
  EM_ASSINATURA: 'warning',
  CONTRATO_ASSINADO: 'success',
  CONTA_CRIADA: 'success',
  REJEITADA: 'error',
};

const statusLabels = {
  EM_PREENCHIMENTO: 'Em preenchimento',
  CNPJ_CONSULTADO: 'CNPJ consultado',
  SOCIOS_VALIDADOS: 'Sócios validados',
  DOCUMENTOS_ANALISADOS: 'Documentos analisados',
  AML_APROVADO: 'AML aprovado',
  COMPLIANCE_APROVADO: 'Compliance aprovado',
  EM_ASSINATURA: 'Em assinatura',
  CONTRATO_ASSINADO: 'Contrato assinado',
  CONTA_CRIADA: 'Conta criada',
  REJEITADA: 'Rejeitada',
};

function TrackingPJ({ solicitacaoId, onNew }) {
  const [solicitacao, setSolicitacao] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState('');

  const load = async () => {
    setLoading(true);
    setError('');
    try {
      const res = await apiService.getSolicitacaoPJ(solicitacaoId);
      setSolicitacao(res.data);
    } catch (err) {
      setError('Não foi possível carregar os dados da solicitação');
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => { load(); }, [solicitacaoId]);

  if (loading) return <Box sx={{ display: 'flex', justifyContent: 'center', py: 4 }}><CircularProgress /></Box>;

  if (error) return <Alert severity="error">{error}</Alert>;

  if (!solicitacao) return null;

  const statusColor = statusColors[solicitacao.status] || 'default';
  const isRejected = solicitacao.status === 'REJEITADA';
  const isApproved = solicitacao.status === 'CONTA_CRIADA' || solicitacao.status === 'CONTRATO_ASSINADO';

  const details = [
    ['CNPJ', solicitacao.cnpj],
    ['Razão Social', solicitacao.razaoSocial],
    ['Nome Fantasia', solicitacao.nomeFantasia],
    ['E-mail', solicitacao.email],
    ['Telefone', solicitacao.telefone],
    ['Criada em', solicitacao.dataCriacao ? format(new Date(solicitacao.dataCriacao), 'dd/MM/yyyy HH:mm', { locale: ptBR }) : '-'],
    ['Atualizada em', solicitacao.dataAtualizacao ? format(new Date(solicitacao.dataAtualizacao), 'dd/MM/yyyy HH:mm', { locale: ptBR }) : '-'],
  ];

  return (
    <Box>
      <Box sx={{ display: 'flex', alignItems: 'center', gap: 2, mb: 3, flexWrap: 'wrap' }}>
        <Typography variant="h5">Solicitação #{solicitacao.id}</Typography>
        <Chip label={statusLabels[solicitacao.status] || solicitacao.status} color={statusColor} />
        {isRejected && <Chip icon={<Cancel />} label="Rejeitada" color="error" />}
        {isApproved && <Chip icon={<CheckCircle />} label="Aprovada" color="success" />}
      </Box>

      {solicitacao.observacoesAnalista && (
        <Alert severity={isRejected ? 'error' : 'info'} sx={{ mb: 2 }}>
          {solicitacao.observacoesAnalista}
        </Alert>
      )}

      <Card sx={{ mb: 3 }}>
        <CardContent>
          <Typography variant="subtitle1" gutterBottom>Dados da solicitação</Typography>
          <Table size="small">
            <TableBody>
              {details.map(([label, value]) => (
                <TableRow key={label}>
                  <TableCell sx={{ fontWeight: 600, width: 200 }}>{label}</TableCell>
                  <TableCell>{value || '-'}</TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </CardContent>
      </Card>

      {solicitacao.historico && solicitacao.historico.length > 0 && (
        <Card sx={{ mb: 3 }}>
          <CardContent>
            <Typography variant="subtitle1" gutterBottom>Histórico</Typography>
            {solicitacao.historico.map((h, i) => (
              <Box key={i} sx={{ display: 'flex', alignItems: 'center', gap: 1, py: 0.5 }}>
                <HourglassEmpty fontSize="small" color="action" />
                <Typography variant="body2">
                  {h.acao} — {h.usuarioAnalista ? `${h.usuarioAnalista} — ` : ''}{h.dataAcao ? format(new Date(h.dataAcao), 'dd/MM/yyyy HH:mm', { locale: ptBR }) : ''}
                </Typography>
              </Box>
            ))}
          </CardContent>
        </Card>
      )}

      {solicitacao.documentos && solicitacao.documentos.length > 0 && (
        <Card sx={{ mb: 3 }}>
          <CardContent>
            <Typography variant="subtitle1" gutterBottom>Documentos</Typography>
            {solicitacao.documentos.map((d, i) => (
              <Box key={i} sx={{ display: 'flex', alignItems: 'center', gap: 1, py: 0.5 }}>
                <Chip size="small" label={d.validado ? 'OK' : 'Pendente'} color={d.validado ? 'success' : 'default'} />
                <Typography variant="body2">{d.tipoDocumento}: {d.nomeArquivo}</Typography>
              </Box>
            ))}
          </CardContent>
        </Card>
      )}

      <Button variant="outlined" startIcon={<ArrowBack />} onClick={onNew}>
        Nova solicitação
      </Button>
    </Box>
  );
}

export default TrackingPJ;
```

- [ ] **Step 2: Create `TrackingPJ.test.js`**

```js
import React from 'react';
import { render, screen, waitFor } from '@testing-library/react';
import TrackingPJ from './TrackingPJ';
import { apiService } from '../../services/apiService';

jest.mock('../../services/apiService');

const mockSolicitacao = {
  id: 99,
  cnpj: '12345678000190',
  razaoSocial: 'Empresa Ltda',
  email: 'contato@empresa.com',
  status: 'EM_PREENCHIMENTO',
  dataCriacao: '2026-07-06T10:00:00',
  dataAtualizacao: '2026-07-06T10:30:00',
  historico: [{ acao: 'Solicitação criada', dataAcao: '2026-07-06T10:00:00' }],
  documentos: [{ tipoDocumento: 'CONTRATO_SOCIAL', nomeArquivo: 'contrato.pdf', validado: false }],
};

test('renders solicitation details', async () => {
  apiService.getSolicitacaoPJ.mockResolvedValue({ data: mockSolicitacao });
  render(<TrackingPJ solicitacaoId={99} onNew={jest.fn()} />);
  await waitFor(() => expect(screen.getByText('Solicitação #99')).toBeInTheDocument());
  expect(screen.getByText('12345678000190')).toBeInTheDocument();
  expect(screen.getByText('Empresa Ltda')).toBeInTheDocument();
});

test('shows loading state', () => {
  apiService.getSolicitacaoPJ.mockReturnValue(new Promise(() => {}));
  render(<TrackingPJ solicitacaoId={99} onNew={jest.fn()} />);
  expect(screen.getByRole('progressbar')).toBeInTheDocument();
});

test('shows error on API failure', async () => {
  apiService.getSolicitacaoPJ.mockRejectedValue(new Error('not found'));
  render(<TrackingPJ solicitacaoId={999} onNew={jest.fn()} />);
  await waitFor(() => expect(screen.getByText(/não foi possível/i)).toBeInTheDocument());
});
```

- [ ] **Step 3: Run tests to verify they pass**

Run: `cd frontend/aurix-web && npx react-scripts test --watchAll=false --testPathPattern="TrackingPJ"`
Expected: all 3 tests pass

- [ ] **Step 4: Commit**

```bash
git add frontend/aurix-web/src/components/Onboarding/TrackingPJ.js frontend/aurix-web/src/components/Onboarding/TrackingPJ.test.js
git commit -m "feat(aurix-web): add TrackingPJ post-creation component"
```

---
### Task 5: Onboarding.js — Container Rewrite

**Files:**
- Modify: `src/pages/Onboarding.js`
- Modify: `src/pages/Onboarding.test.js`

**Interfaces:**
- Consumes: `TipoSelector`, `FormPF`, `WizardPJ`, `TrackingPJ`
- Produces: Page component with `({ user })` prop (existing pattern)

- [ ] **Step 1: Rewrite `Onboarding.js`**

```js
import React, { useState } from 'react';
import { Box, Button, Alert, Typography } from '@mui/material';
import { ArrowBack } from '@mui/icons-material';
import TipoSelector from '../components/Onboarding/TipoSelector';
import FormPF from '../components/Onboarding/FormPF';
import WizardPJ from '../components/Onboarding/FormPJ/WizardPJ';
import TrackingPJ from '../components/Onboarding/TrackingPJ';

function Onboarding({ user }) {
  const [mode, setMode] = useState('select'); // select | form | tracking | success
  const [tipo, setTipo] = useState(null); // PF | PJ
  const [solicitacaoId, setSolicitacaoId] = useState(null);

  const handleSelect = (selectedTipo) => {
    setTipo(selectedTipo);
    setMode('form');
  };

  const handleComplete = (id) => {
    setSolicitacaoId(id);
    if (tipo === 'PF') {
      setMode('success');
    } else {
      setMode('tracking');
    }
  };

  const handleNew = () => {
    setMode('select');
    setTipo(null);
    setSolicitacaoId(null);
  };

  if (mode === 'select') {
    return <TipoSelector onSelect={handleSelect} />;
  }

  if (mode === 'success') {
    return (
      <Box>
        <Alert severity="success" sx={{ mb: 2 }}>
          Solicitação enviada com sucesso! Número de protocolo: <strong>#{solicitacaoId}</strong>
        </Alert>
        <Typography variant="body2" color="text.secondary" sx={{ mb: 2 }}>
          Acompanhe o andamento pelo seu e-mail ou entre em contato com nosso suporte.
        </Typography>
        <Button variant="outlined" onClick={handleNew}>Nova solicitação</Button>
      </Box>
    );
  }

  if (mode === 'tracking' && solicitacaoId) {
    return <TrackingPJ solicitacaoId={solicitacaoId} onNew={handleNew} />;
  }

  return (
    <Box>
      <Button startIcon={<ArrowBack />} onClick={handleNew} sx={{ mb: 2 }}>
        Voltar
      </Button>
      {tipo === 'PF' ? (
        <FormPF onComplete={handleComplete} />
      ) : (
        <WizardPJ onComplete={handleComplete} />
      )}
    </Box>
  );
}

export default Onboarding;
```

- [ ] **Step 2: Update `Onboarding.test.js`**

```js
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import Onboarding from './Onboarding';
import { apiService } from '../services/apiService';

jest.mock('../services/apiService');

test('renders tipo selector initially', () => {
  render(<Onboarding user={{ nome: 'Test' }} />);
  expect(screen.getByText('Pessoa Física')).toBeInTheDocument();
  expect(screen.getByText('Pessoa Jurídica')).toBeInTheDocument();
});

test('shows PF form when PF card clicked', () => {
  render(<Onboarding user={{ nome: 'Test' }} />);
  fireEvent.click(screen.getByText('Pessoa Física'));
  expect(screen.getByLabelText('CPF')).toBeInTheDocument();
});

test('shows PJ wizard when PJ card clicked', () => {
  render(<Onboarding user={{ nome: 'Test' }} />);
  fireEvent.click(screen.getByText('Pessoa Jurídica'));
  expect(screen.getByText('Dados da Empresa')).toBeInTheDocument();
  expect(screen.getByLabelText('CNPJ')).toBeInTheDocument();
});

test('PF flow: from form to success after submit', async () => {
  apiService.criarSolicitacaoPF.mockResolvedValue({ data: { id: 42 } });
  render(<Onboarding user={{ nome: 'Test' }} />);
  fireEvent.click(screen.getByText('Pessoa Física'));
  fireEvent.change(screen.getByLabelText('CPF'), { target: { value: '123.456.789-00' } });
  fireEvent.change(screen.getByLabelText('Nome completo'), { target: { value: 'João' } });
  fireEvent.change(screen.getByLabelText('E-mail'), { target: { value: 'joao@test.com' } });
  fireEvent.submit(screen.getByRole('button', { name: /solicitar/i }));
  await waitFor(() => expect(screen.getByText(/protocolo/i)).toBeInTheDocument());
  expect(screen.getByText(/#42/i)).toBeInTheDocument();
});

test('PJ flow: from wizard to tracking after create', async () => {
  apiService.criarSolicitacaoPJ.mockResolvedValue({ data: { id: 99 } });
  apiService.getSolicitacaoPJ.mockResolvedValue({ data: { id: 99, cnpj: '12345678000190', razaoSocial: 'Empresa', email: 'c@e.com', status: 'EM_PREENCHIMENTO', historico: [], documentos: [] } });
  render(<Onboarding user={{ nome: 'Test' }} />);
  fireEvent.click(screen.getByText('Pessoa Jurídica'));
  fireEvent.change(screen.getByLabelText('CNPJ'), { target: { value: '12.345.678/0001-90' } });
  fireEvent.change(screen.getByLabelText('Razão Social'), { target: { value: 'Empresa Ltda' } });
  fireEvent.change(screen.getByLabelText('E-mail'), { target: { value: 'c@e.com' } });
  fireEvent.click(screen.getByRole('button', { name: /criar solicitação/i }));
  await waitFor(() => expect(apiService.criarSolicitacaoPJ).toHaveBeenCalled());
  fireEvent.click(screen.getByRole('button', { name: /ver acompanhamento/i }));
  await waitFor(() => expect(screen.getByText(/solicitação #99/i)).toBeInTheDocument());
});

test('back button returns to selector from form', () => {
  render(<Onboarding user={{ nome: 'Test' }} />);
  fireEvent.click(screen.getByText('Pessoa Jurídica'));
  fireEvent.click(screen.getByText('Voltar'));
  expect(screen.getByText('Pessoa Física')).toBeInTheDocument();
});
```

- [ ] **Step 3: Run all tests to verify they pass**

Run: `cd frontend/aurix-web && npx react-scripts test --watchAll=false --testPathPattern="Onboarding"`
Expected: all tests pass (TipoSelector: 3, FormPF: 3, WizardPJ: 3, TrackingPJ: 3, Onboarding: 5 = 17 total)

- [ ] **Step 4: Run lint check**

Run: `cd frontend/aurix-web && npx eslint src/pages/Onboarding.js src/components/Onboarding/ --ext .js`
Expected: 0 errors, pre-existing warnings only (no new ones)

- [ ] **Step 5: Commit**

```bash
git add frontend/aurix-web/src/pages/Onboarding.js frontend/aurix-web/src/pages/Onboarding.test.js
git commit -m "feat(aurix-web): rewrite onboarding page with PF/PJ flow"
```
