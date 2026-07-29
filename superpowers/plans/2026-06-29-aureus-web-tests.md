# Aurix-web Test Coverage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Full test coverage for aurix-web — infra, services, components, pages, and App routing.

**Architecture:** Jest (CRA/react-scripts) + @testing-library/react. Mock services via `jest.mock()` + `__mocks__/` directory. Shared `renderWithProviders` wrapper.

**Tech Stack:** Jest 29, @testing-library/react 13, @testing-library/jest-dom 5, react-scripts 5

## Global Constraints

- All test files go in `aurix-web/src/` mirroring source structure
- No TypeScript — all files are `.js`
- Use `renderWithProviders` for component/page tests (BrowserRouter + ThemeProvider wrapper)
- Mock `apiService` and `authService` via `jest.mock()` in each test suite
- Use `jest.mock('../../services/apiService', () => ({...}))` for page tests
- Setup: `src/setupTests.js` mocks `window.matchMedia` + imports `@testing-library/jest-dom`
- Run: `npm test` from `apps/frontend/aurix-web/` (CRA watch mode) or `npx react-scripts test --watchAll=false`
- Commits target `develop` branch (or `main` per session context)

---
### Task 1: Test Infrastructure (setupTests + mocks + test-utils)

**Files:**
- Create: `apps/frontend/aurix-web/src/setupTests.js`
- Create: `apps/frontend/aurix-web/src/test-utils.js`
- Create: `apps/frontend/aurix-web/src/services/__mocks__/apiService.js`
- Create: `apps/frontend/aurix-web/src/services/__mocks__/authService.js`

- [ ] **Step 1: Create `src/setupTests.js`**

```js
import '@testing-library/jest-dom';

window.matchMedia = (query) => ({
  matches: false,
  media: query,
  onchange: null,
  addListener: () => {},
  removeListener: () => {},
  addEventListener: () => {},
  removeEventListener: () => {},
  dispatchEvent: () => false,
});

window.scrollTo = () => {};
```

- [ ] **Step 2: Create `src/test-utils.js`**

```js
import React from 'react';
import { render } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { ThemeProvider, createTheme } from '@mui/material/styles';

const theme = createTheme({
  palette: { primary: { main: '#1976d2' }, secondary: { main: '#dc004e' } },
});

export const mockUser = {
  nome: 'Maria Silva',
  email: 'maria@test.com',
  cpf: '12345678901',
  telefone: '(11) 99999-8888',
  conta: { id: '1', saldo: 15750.5, numero: '12345-6', agencia: '0001', tipo: 'CORRENTE' },
};

export const mockContas = [
  { id: '1', tipo: 'CORRENTE', saldo: 15750.5, numero: '12345-6', agencia: '0001', status: 'ATIVA', dataAbertura: '2024-01-15', limite: 5000, rendimento: 0.5 },
  { id: '2', tipo: 'POUPANCA', saldo: 25000, numero: '12345-7', agencia: '0001', status: 'ATIVA', dataAbertura: '2024-03-10', rendimento: 0.5 },
  { id: '3', tipo: 'INVESTIMENTO', saldo: 50000, numero: '12345-8', agencia: '0001', status: 'ATIVA', dataAbertura: '2024-06-20', rendimento: 1.2 },
];

export const mockTransacoes = [
  { id: '1', codigo: 'TXN-001', tipo: 'PIX', descricao: 'Transferencia para Joao', valor: 1500, data: '2024-12-01T10:30:00', status: 'CONCLUIDA', contaId: '1' },
  { id: '2', codigo: 'TXN-002', tipo: 'TED', descricao: 'Pagamento fornecedor', valor: 3500, data: '2024-12-02T14:00:00', status: 'CONCLUIDA', contaId: '1' },
  { id: '3', codigo: 'TXN-003', tipo: 'DOC', descricao: 'Aluguel', valor: 2500, data: '2024-12-03T09:00:00', status: 'PENDENTE', contaId: '1' },
];

export const mockInvestimentos = [
  { id: '1', tipo: 'CDB', valorInvestido: 10000, taxa: 13.5, rendimento: 850, valorTotal: 10850, dataAplicacao: '2024-01-10', dataVencimento: '2025-01-10', status: 'ATIVO' },
  { id: '2', tipo: 'LCI', valorInvestido: 5000, taxa: 12.0, rendimento: 300, valorTotal: 5300, dataAplicacao: '2024-03-15', dataVencimento: '2025-03-15', status: 'ATIVO' },
];

export const mockCartoes = [
  { id: '1', bandeira: 'Visa', tipo: 'CREDITO', status: 'ATIVO', numero: '**** **** **** 1234', nomePortador: 'Maria Silva', validade: '12/26', limite: 5000, disponivel: 3750 },
  { id: '2', bandeira: 'Mastercard', tipo: 'DEBITO', status: 'ATIVO', numero: '**** **** **** 5678', nomePortador: 'Maria Silva', validade: '12/26' },
];

export const mockFaturas = [
  { id: '1', cartaoId: '1', mesAno: '12/2024', valorTotal: 1250, valorPago: 1250, valorPendente: 0, vencimento: '10/12/2024', status: 'PAGA' },
  { id: '2', cartaoId: '1', mesAno: '01/2025', valorTotal: 1500, valorPago: 0, valorPendente: 1500, vencimento: '10/01/2025', status: 'PENDENTE' },
];

export function renderWithProviders(ui, { user = mockUser } = {}) {
  return render(
    <BrowserRouter>
      <ThemeProvider theme={theme}>
        {React.cloneElement(ui, { user })}
      </ThemeProvider>
    </BrowserRouter>
  );
}
```

- [ ] **Step 3: Create `src/services/__mocks__/apiService.js`**

```js
const mockContas = [
  { id: '1', tipo: 'CORRENTE', saldo: 15750.5, numero: '12345-6', agencia: '0001', status: 'ATIVA', dataAbertura: '2024-01-15', limite: 5000, rendimento: 0.5 },
  { id: '2', tipo: 'POUPANCA', saldo: 25000, numero: '12345-7', agencia: '0001', status: 'ATIVA', dataAbertura: '2024-03-10', rendimento: 0.5 },
];
const mockTransacoes = [
  { id: '1', codigo: 'TXN-001', tipo: 'PIX', descricao: 'Transferencia', valor: 1500, data: '2024-12-01T10:30:00', status: 'CONCLUIDA', contaId: '1' },
];
const mockInvestimentos = [
  { id: '1', tipo: 'CDB', valorInvestido: 10000, taxa: 13.5, rendimento: 850, valorTotal: 10850, dataAplicacao: '2024-01-10', dataVencimento: '2025-01-10', status: 'ATIVO' },
];
const mockCartoes = [
  { id: '1', bandeira: 'Visa', tipo: 'CREDITO', status: 'ATIVO', numero: '**** **** **** 1234', nomePortador: 'Maria Silva', validade: '12/26', limite: 5000, disponivel: 3750 },
];
const mockFaturas = [
  { id: '1', cartaoId: '1', mesAno: '12/2024', valorTotal: 1250, valorPago: 1250, valorPendente: 0, vencimento: '10/12/2024', status: 'PAGA' },
];

const apiService = {
  getContas: jest.fn().mockResolvedValue({ data: mockContas }),
  getConta: jest.fn().mockResolvedValue({ data: mockContas[0] }),
  getTransacoes: jest.fn().mockResolvedValue({ data: mockTransacoes }),
  criarTransacao: jest.fn().mockResolvedValue({ data: { id: '4', status: 'CONCLUIDA' } }),
  enviarPix: jest.fn().mockResolvedValue({ data: { id: '1', status: 'CONCLUIDA' } }),
  receberPix: jest.fn().mockResolvedValue({ data: { id: '2', qrCode: 'base64...' } }),
  getInvestimentos: jest.fn().mockResolvedValue({ data: mockInvestimentos }),
  criarInvestimento: jest.fn().mockResolvedValue({ data: { id: '3' } }),
  simularInvestimento: jest.fn().mockResolvedValue({ data: { valorInvestido: 10000, valorLiquido: 11500 } }),
  getCartoes: jest.fn().mockResolvedValue({ data: mockCartoes }),
  emitirCartao: jest.fn().mockResolvedValue({ data: { id: '3', status: 'ATIVO' } }),
  getFaturas: jest.fn().mockResolvedValue({ data: mockFaturas }),
  pagarFatura: jest.fn().mockResolvedValue({ data: { id: '1', status: 'PAGA' } }),
};

export default apiService;
```

- [ ] **Step 4: Create `src/services/__mocks__/authService.js`**

```js
const authService = {
  login: jest.fn().mockResolvedValue({ data: { token: 'mock-token', user: { nome: 'Maria Silva', email: 'maria@test.com' } } }),
  logout: jest.fn(),
  getCurrentUser: jest.fn().mockResolvedValue({ data: { nome: 'Maria Silva', email: 'maria@test.com', conta: { saldo: 15750.5 } } }),
  register: jest.fn().mockResolvedValue({ data: { id: '1', status: 'CRIADO' } }),
  requestPasswordReset: jest.fn().mockResolvedValue({ data: { message: 'Email enviado' } }),
  verifyMFA: jest.fn().mockResolvedValue({ data: { valido: true } }),
};

export default authService;
```

- [ ] **Step 5: Verify setup**

```bash
cd /mnt/c/Users/wende/Projects/aurix-platform/apps/frontend/aurix-web
npx react-scripts test --watchAll=false --noStackTrace 2>&1 | tail -20
```
CRA may complain about no tests found (passWithNoTests). Expected: exits with 0.

- [ ] **Step 6: Commit**

```bash
git add apps/frontend/aurix-web/src/setupTests.js apps/frontend/aurix-web/src/test-utils.js apps/frontend/aurix-web/src/services/__mocks__/apiService.js apps/frontend/aurix-web/src/services/__mocks__/authService.js
git commit -m "test(web): test infrastructure - setupTests, renderWithProviders, service mocks"
```

---

### Task 2: Service Tests (apiService + authService)

**Files:**
- Create: `apps/frontend/aurix-web/src/services/apiService.test.js`
- Create: `apps/frontend/aurix-web/src/services/authService.test.js`

- [ ] **Step 1: Create `apiService.test.js`**

```js
jest.mock('axios');
import axios from 'axios';
import api from './apiService';

beforeEach(() => {
  localStorage.setItem('aurix_token', 'test-token');
});
afterEach(() => {
  localStorage.clear();
  jest.clearAllMocks();
});

test('getContas envia GET para /contas com token', async () => {
  axios.get.mockResolvedValue({ data: [{ id: '1' }] });
  const result = await api.getContas();
  expect(axios.get).toHaveBeenCalledWith('/contas', { headers: { Authorization: 'Bearer test-token' } });
  expect(result.data).toHaveLength(1);
});

test('enviarPix envia POST para /pix/enviar', async () => {
  axios.post.mockResolvedValue({ data: { id: '1' } });
  const payload = { chave: 'test@test.com', valor: 100 };
  await api.enviarPix(payload);
  expect(axios.post).toHaveBeenCalledWith('/pix/enviar', payload, { headers: { Authorization: 'Bearer test-token' } });
});

test('receberPix envia POST para /pix/receber', async () => {
  axios.post.mockResolvedValue({ data: { qrCode: 'base64...' } });
  await api.receberPix({ valor: 200 });
  expect(axios.post).toHaveBeenCalledWith('/pix/receber', { valor: 200 }, expect.any(Object));
});

test('getInvestimentos faz GET com contaId', async () => {
  axios.get.mockResolvedValue({ data: [] });
  await api.getInvestimentos('1');
  expect(axios.get).toHaveBeenCalledWith('/investimentos/conta/1', expect.any(Object));
});

test('emitirCartao envia POST para /cartoes/emitir', async () => {
  axios.post.mockResolvedValue({ data: { id: '1' } });
  await api.emitirCartao({ tipo: 'CREDITO', bandeira: 'Visa' });
  expect(axios.post).toHaveBeenCalledWith('/cartoes/emitir', { tipo: 'CREDITO', bandeira: 'Visa' }, expect.any(Object));
});

test('simularInvestimento envia POST para /investimentos/simular', async () => {
  axios.post.mockResolvedValue({ data: { valorLiquido: 11500 } });
  await api.simularInvestimento('CDB', 10000, 13.5, 360);
  expect(axios.post).toHaveBeenCalledWith('/investimentos/simular', { tipo: 'CDB', valor: 10000, taxa: 13.5, dias: 360 }, expect.any(Object));
});

test('interceptor redireciona ao receber 401', async () => {
  delete window.location;
  window.location = { href: '' };
  axios.get.mockRejectedValue({ response: { status: 401 } });
  await expect(api.getContas()).rejects.toThrow();
  expect(localStorage.getItem('aurix_token')).toBeNull();
  expect(window.location.href).toBe('/login');
});
```

- [ ] **Step 2: Create `authService.test.js`**

```js
jest.mock('axios');
import axios from 'axios';
import auth from './authService';

beforeEach(() => {
  localStorage.setItem('aurix_token', 'test-token');
});
afterEach(() => {
  localStorage.clear();
  jest.clearAllMocks();
});

test('login envia POST para /auth/login', async () => {
  axios.post.mockResolvedValue({ data: { token: 'new-token', user: { nome: 'Maria' } } });
  const result = await auth.login({ cpf: '12345678901', senha: '123' });
  expect(axios.post).toHaveBeenCalledWith('/auth/login', { cpf: '12345678901', senha: '123' }, expect.any(Object));
  expect(result.data.token).toBe('new-token');
});

test('getCurrentUser envia GET para /auth/me', async () => {
  axios.get.mockResolvedValue({ data: { nome: 'Maria' } });
  const result = await auth.getCurrentUser();
  expect(axios.get).toHaveBeenCalledWith('/auth/me', expect.any(Object));
  expect(result.data.nome).toBe('Maria');
});

test('logout limpa token do localStorage', () => {
  auth.logout();
  expect(localStorage.getItem('aurix_token')).toBeNull();
});

test('register envia POST para /auth/register', async () => {
  axios.post.mockResolvedValue({ data: { id: '1' } });
  await auth.register({ nome: 'Maria', email: 'maria@test.com' });
  expect(axios.post).toHaveBeenCalledWith('/auth/register', { nome: 'Maria', email: 'maria@test.com' }, expect.any(Object));
});

test('requestPasswordReset envia POST para /auth/password-reset', async () => {
  axios.post.mockResolvedValue({ data: { message: 'Email enviado' } });
  await auth.requestPasswordReset('maria@test.com');
  expect(axios.post).toHaveBeenCalledWith('/auth/password-reset', { email: 'maria@test.com' }, expect.any(Object));
});
```

- [ ] **Step 3: Run tests**

```bash
cd /mnt/c/Users/wende/Projects/aurix-platform/apps/frontend/aurix-web
npx react-scripts test --watchAll=false --noStackTrace 2>&1 | tail -30
```
Expected: 12 tests pass (7 apiService + 5 authService)

- [ ] **Step 4: Commit**

```bash
git add apps/frontend/aurix-web/src/services/apiService.test.js apps/frontend/aurix-web/src/services/authService.test.js
git commit -m "test(web): apiService and authService unit tests"
```

---

### Task 3: Component Tests (Navbar + Sidebar + Dashboard)

**Files:**
- Create: `apps/frontend/aurix-web/src/components/Navbar.test.js`
- Create: `apps/frontend/aurix-web/src/components/Sidebar.test.js`
- Create: `apps/frontend/aurix-web/src/components/Dashboard.test.js`

- [ ] **Step 1: Create `Navbar.test.js`**

```js
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { ThemeProvider, createTheme } from '@mui/material/styles';
import Navbar from './Navbar';

const theme = createTheme();
const mockUser = { nome: 'Maria Silva', email: 'maria@test.com' };
const onLogout = jest.fn();
const onToggleSidebar = jest.fn();

function renderNavbar(props = {}) {
  return render(
    <BrowserRouter>
      <ThemeProvider theme={theme}>
        <Navbar user={mockUser} onLogout={onLogout} onToggleSidebar={onToggleSidebar} {...props} />
      </ThemeProvider>
    </BrowserRouter>
  );
}

test('renderiza nome do banco', () => {
  renderNavbar();
  expect(screen.getByText(/AURIX Internet Banking/i)).toBeInTheDocument();
});

test('renderiza nome do usuario', () => {
  renderNavbar();
  expect(screen.getByText(/Maria Silva/i)).toBeInTheDocument();
});

test('abre menu do profile ao clicar no avatar', () => {
  renderNavbar();
  fireEvent.click(screen.getByText(/MS/i));
  expect(screen.getByText(/Meu Perfil/i)).toBeInTheDocument();
  expect(screen.getByText(/Sair/i)).toBeInTheDocument();
});

test('chama onLogout ao clicar em Sair', () => {
  renderNavbar();
  fireEvent.click(screen.getByText(/MS/i));
  fireEvent.click(screen.getByText(/Sair/i));
  expect(onLogout).toHaveBeenCalled();
});

test('chama onToggleSidebar ao clisar no menu hamburguer', () => {
  renderNavbar();
  const menuBtn = document.querySelector('[data-testid="MenuIcon"]')?.closest('button')
    || document.querySelector('button');
  if (menuBtn) fireEvent.click(menuBtn);
  expect(onToggleSidebar).toHaveBeenCalled();
});

test('mostra badge de notificacoes', () => {
  renderNavbar();
  const badge = screen.getByText('3');
  expect(badge).toBeInTheDocument();
});
```

- [ ] **Step 2: Create `Sidebar.test.js`**

```js
import React from 'react';
import { render, screen } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { ThemeProvider, createTheme } from '@mui/material/styles';
import Sidebar from './Sidebar';

const theme = createTheme();
const onClose = jest.fn();

function renderSidebar(open = true) {
  return render(
    <MemoryRouter initialEntries={['/dashboard']}>
      <ThemeProvider theme={theme}>
        <Sidebar open={open} onClose={onClose} />
      </ThemeProvider>
    </MemoryRouter>
  );
}

test('renderiza itens do menu', () => {
  renderSidebar();
  expect(screen.getByText(/Dashboard/i)).toBeInTheDocument();
  expect(screen.getByText(/Contas/i)).toBeInTheDocument();
  expect(screen.getByText(/Transacoes/i)).toBeInTheDocument();
  expect(screen.getByText(/PIX/i)).toBeInTheDocument();
  expect(screen.getByText(/Investimentos/i)).toBeInTheDocument();
  expect(screen.getByText(/Cartoes/i)).toBeInTheDocument();
  expect(screen.getByText(/Credito/i)).toBeInTheDocument();
});

test('destaca rota ativa', () => {
  renderSidebar();
  const activeItem = screen.getByText(/Dashboard/i).closest('div');
  expect(activeItem).toHaveStyle('background-color: rgb(25, 118, 210)');
});

test('nao renderiza quando fechado em temporary', () => {
  const { container } = renderSidebar(false);
  expect(container.querySelector('.MuiDrawer-root')).toBeInTheDocument();
});
```

- [ ] **Step 3: Create `Dashboard.test.js`**

```js
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { ThemeProvider, createTheme } from '@mui/material/styles';
import Dashboard from './Dashboard';

const theme = createTheme();

function renderDashboard() {
  return render(
    <BrowserRouter>
      <ThemeProvider theme={theme}>
        <Dashboard />
      </ThemeProvider>
    </BrowserRouter>
  );
}

test('renderiza saldo apos carregar', async () => {
  renderDashboard();
  expect(await screen.findByText(/15\.750,50/)).toBeInTheDocument();
});

test('renderiza titulo Resumo Financeiro', async () => {
  renderDashboard();
  expect(await screen.findByText(/Resumo Financeiro/i)).toBeInTheDocument();
});

test('renderiza transacoes recentes', async () => {
  renderDashboard();
  expect(await screen.findByText(/Transferencia para Joao/i)).toBeInTheDocument();
});

test('renderiza investimentos', async () => {
  renderDashboard();
  expect(await screen.findByText(/CDB/i)).toBeInTheDocument();
});

test('renderiza cartoes', async () => {
  renderDashboard();
  expect(await screen.findByText(/Credito/i)).toBeInTheDocument();
});

test('abre dialog do PIX', async () => {
  renderDashboard();
  const pixBtn = await screen.findByText(/Novo PIX/i);
  fireEvent.click(pixBtn);
  expect(screen.getByText(/Enviar PIX/i)).toBeInTheDocument();
});
```

- [ ] **Step 4: Run component tests**

```bash
cd /mnt/c/Users/wende/Projects/aurix-platform/apps/frontend/aurix-web
npx react-scripts test --watchAll=false --noStackTrace 2>&1 | tail -30
```
Expected: ~17 tests pass (6 Navbar + 3 Sidebar + 6 Dashboard component)

- [ ] **Step 5: Commit**

```bash
git add apps/frontend/aurix-web/src/components/Navbar.test.js apps/frontend/aurix-web/src/components/Sidebar.test.js apps/frontend/aurix-web/src/components/Dashboard.test.js
git commit -m "test(web): Navbar, Sidebar, and Dashboard component tests"
```

---

### Task 4: Page Tests — Login + Dashboard + Contas + Transacoes

**Files:**
- Create: `apps/frontend/aurix-web/src/pages/Login.test.js`
- Create: `apps/frontend/aurix-web/src/pages/Dashboard.test.js`
- Create: `apps/frontend/aurix-web/src/pages/Contas.test.js`
- Create: `apps/frontend/aurix-web/src/pages/Transacoes.test.js`

- [ ] **Step 1: Create `Login.test.js`**

```js
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { ThemeProvider, createTheme } from '@mui/material/styles';
import Login from './Login';

const theme = createTheme();
const mockOnLogin = jest.fn();

function renderLogin() {
  return render(
    <BrowserRouter>
      <ThemeProvider theme={theme}>
        <Login onLogin={mockOnLogin} />
      </ThemeProvider>
    </BrowserRouter>
  );
}

test('renderiza formulario de login', () => {
  renderLogin();
  expect(screen.getByLabelText(/CPF/i)).toBeInTheDocument();
  expect(screen.getByLabelText(/Senha/i)).toBeInTheDocument();
  expect(screen.getByRole('button', { name: /Acessar/i })).toBeInTheDocument();
});

test('chama onLogin com dados do formulario', async () => {
  renderLogin();
  fireEvent.change(screen.getByLabelText(/CPF/i), { target: { value: '12345678901' } });
  fireEvent.change(screen.getByLabelText(/Senha/i), { target: { value: 'minhasenha' } });
  fireEvent.click(screen.getByRole('button', { name: /Acessar/i }));
  await waitFor(() => {
    expect(mockOnLogin).toHaveBeenCalledWith('12345678901', 'minhasenha');
  });
});

test('renderiza link de cadastro', () => {
  renderLogin();
  expect(screen.getByText(/Ainda nao tem conta/i)).toBeInTheDocument();
});

test('renderiza link de esqueci senha', () => {
  renderLogin();
  expect(screen.getByText(/Esqueci minha senha/i)).toBeInTheDocument();
});
```

- [ ] **Step 2: Create `pages/Dashboard.test.js`**

```js
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { ThemeProvider, createTheme } from '@mui/material/styles';
import { renderWithProviders, mockUser } from '../../test-utils';
import Dashboard from './Dashboard';

test('renderiza saudacao com nome do usuario', () => {
  renderWithProviders(<Dashboard />);
  expect(screen.getByText(/Maria Silva/i)).toBeInTheDocument();
});

test('renderiza saldo da conta', async () => {
  renderWithProviders(<Dashboard />);
  expect(await screen.findByText(/15\.750,50/)).toBeInTheDocument();
});

test('renderiza resumo financeiro', async () => {
  renderWithProviders(<Dashboard />);
  expect(await screen.findByText(/Resumo Financeiro/i)).toBeInTheDocument();
});

test('renderiza transacoes', async () => {
  renderWithProviders(<Dashboard />);
  expect(await screen.findByText(/Transferencia para Joao/i)).toBeInTheDocument();
});

test('renderiza investimentos', async () => {
  renderWithProviders(<Dashboard />);
  expect(await screen.findByText(/CDB/i)).toBeInTheDocument();
});
```

- [ ] **Step 3: Create `Contas.test.js`**

```js
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import { renderWithProviders, mockContas } from '../../test-utils';
import Contas from './Contas';

test('renderiza titulo Minhas Contas', () => {
  renderWithProviders(<Contas />);
  expect(screen.getByText(/Minhas Contas/i)).toBeInTheDocument();
});

test('renderiza contas mockadas', async () => {
  renderWithProviders(<Contas />);
  expect(await screen.findByText(/Corrente/i)).toBeInTheDocument();
  expect(screen.getByText(/Poupanca/i)).toBeInTheDocument();
});
```

- [ ] **Step 4: Create `Transacoes.test.js`**

```js
jest.mock('../../services/apiService', () => ({
  getTransacoes: jest.fn().mockResolvedValue({
    data: [
      { id: '1', codigo: 'TXN-001', tipo: 'PIX', descricao: 'Transferencia', valor: 1500, data: '2024-12-01T10:30:00', status: 'CONCLUIDA', contaId: '1' },
    ],
  }),
}));

import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { renderWithProviders } from '../../test-utils';
import Transacoes from './Transacoes';

test('renderiza titulo Transacoes', () => {
  renderWithProviders(<Transacoes />);
  expect(screen.getByText(/Transacoes/i)).toBeInTheDocument();
});

test('renderiza filtros', () => {
  renderWithProviders(<Transacoes />);
  expect(screen.getByText(/Conta/i)).toBeInTheDocument();
  expect(screen.getByText(/Tipo/i)).toBeInTheDocument();
  expect(screen.getByText(/Status/i)).toBeInTheDocument();
});

test('renderiza transacoes carregadas', async () => {
  renderWithProviders(<Transacoes />);
  expect(await screen.findByText(/TXN-001/i)).toBeInTheDocument();
});
```

- [ ] **Step 5: Run tests**

```bash
cd /mnt/c/Users/wende/Projects/aurix-platform/apps/frontend/aurix-web
npx react-scripts test --watchAll=false --noStackTrace 2>&1 | tail -40
```
Expected: ~13 tests pass (4 Login + 5 Dashboard page + 2 Contas + 3 Transacoes)

- [ ] **Step 6: Commit**

```bash
git add apps/frontend/aurix-web/src/pages/Login.test.js apps/frontend/aurix-web/src/pages/Dashboard.test.js apps/frontend/aurix-web/src/pages/Contas.test.js apps/frontend/aurix-web/src/pages/Transacoes.test.js
git commit -m "test(web): Login, Dashboard, Contas, Transacoes page tests"
```

---

### Task 5: Page Tests — PIX + Cartoes + Credito + Investimentos

**Files:**
- Create: `apps/frontend/aurix-web/src/pages/PIX.test.js`
- Create: `apps/frontend/aurix-web/src/pages/Cartoes.test.js`
- Create: `apps/frontend/aurix-web/src/pages/Credito.test.js`
- Create: `apps/frontend/aurix-web/src/pages/Investimentos.test.js`

- [ ] **Step 1: Create `PIX.test.js`**

```js
jest.mock('../../services/apiService', () => ({
  enviarPix: jest.fn().mockResolvedValue({ data: { id: '1', status: 'CONCLUIDA' } }),
  receberPix: jest.fn().mockResolvedValue({ data: { qrCode: 'base64...' } }),
}));

import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { renderWithProviders } from '../../test-utils';
import PIX from './PIX';

test('renderiza tres abas', () => {
  renderWithProviders(<PIX />);
  expect(screen.getByText(/Enviar PIX/i)).toBeInTheDocument();
  expect(screen.getByText(/Receber PIX/i)).toBeInTheDocument();
  expect(screen.getByText(/Chaves PIX/i)).toBeInTheDocument();
});

test('aba enviar PIX tem campos do formulario', () => {
  renderWithProviders(<PIX />);
  expect(screen.getByText(/Tipo de Chave/i)).toBeInTheDocument();
  expect(screen.getByText(/Chave/i)).toBeInTheDocument();
});

test('aba receber PIX tem campo de valor', () => {
  renderWithProviders(<PIX />);
  fireEvent.click(screen.getByText(/Receber PIX/i));
  expect(screen.getByText(/Valor/i)).toBeInTheDocument();
});

test('aba chaves PIX mostra placeholder', () => {
  renderWithProviders(<PIX />);
  fireEvent.click(screen.getByText(/Chaves PIX/i));
  expect(screen.getByText(/em desenvolvimento/i)).toBeInTheDocument();
});
```

- [ ] **Step 2: Create `Cartoes.test.js`**

```js
jest.mock('../../services/apiService', () => ({
  getCartoes: jest.fn().mockResolvedValue({
    data: [
      { id: '1', bandeira: 'Visa', tipo: 'CREDITO', status: 'ATIVO', numero: '**** **** **** 1234', nomePortador: 'Maria Silva', validade: '12/26', limite: 5000, disponivel: 3750 },
    ],
  }),
  getFaturas: jest.fn().mockResolvedValue({
    data: [
      { id: '1', cartaoId: '1', mesAno: '12/2024', valorTotal: 1250, valorPago: 1250, valorPendente: 0, vencimento: '10/12/2024', status: 'PAGA' },
    ],
  }),
  emitirCartao: jest.fn().mockResolvedValue({ data: { id: '2' } }),
  pagarFatura: jest.fn().mockResolvedValue({ data: { id: '1', status: 'PAGA' } }),
}));

import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { renderWithProviders } from '../../test-utils';
import Cartoes from './Cartoes';

test('renderiza abas', () => {
  renderWithProviders(<Cartoes />);
  expect(screen.getByText(/Meus Cartoes/i)).toBeInTheDocument();
  expect(screen.getByText(/Faturas/i)).toBeInTheDocument();
});

test('renderiza cartoes carregados', async () => {
  renderWithProviders(<Cartoes />);
  expect(await screen.findByText(/Visa/i)).toBeInTheDocument();
});

test('renderiza faturas', async () => {
  renderWithProviders(<Cartoes />);
  fireEvent.click(screen.getByText(/Faturas/i));
  expect(await screen.findByText(/12\/2024/i)).toBeInTheDocument();
});
```

- [ ] **Step 3: Create `Credito.test.js`**

```js
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { renderWithProviders, mockUser } from '../../test-utils';
import Credito from './Credito';

beforeEach(() => {
  global.fetch = jest.fn().mockResolvedValue({
    ok: true,
    json: () => Promise.resolve({ valorParcela: 500, total: 12000 }),
  });
});

afterEach(() => {
  delete global.fetch;
});

test('renderiza formulario de simulacao', () => {
  renderWithProviders(<Credito />);
  expect(screen.getByText(/Simular Credito/i)).toBeInTheDocument();
  expect(screen.getByLabelText(/valor desejado/i)).toBeInTheDocument();
});

test('chama fetch ao simular', async () => {
  renderWithProviders(<Credito />);
  fireEvent.change(screen.getByLabelText(/valor desejado/i), { target: { value: '10000' } });
  fireEvent.click(screen.getByText(/Simular/i));
  await waitFor(() => expect(global.fetch).toHaveBeenCalled());
});
```

- [ ] **Step 4: Create `Investimentos.test.js`**

```js
jest.mock('../../services/apiService', () => ({
  getInvestimentos: jest.fn().mockResolvedValue({
    data: [
      { id: '1', tipo: 'CDB', valorInvestido: 10000, taxa: 13.5, rendimento: 850, valorTotal: 10850, status: 'ATIVO' },
    ],
  }),
  simularInvestimento: jest.fn().mockResolvedValue({ data: { valorInvestido: 10000, valorLiquido: 11500 } }),
  criarInvestimento: jest.fn().mockResolvedValue({ data: { id: '2' } }),
}));

import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { renderWithProviders } from '../../test-utils';
import Investimentos from './Investimentos';

test('renderiza resumo', () => {
  renderWithProviders(<Investimentos />);
  expect(screen.getByText(/Total Investido/i)).toBeInTheDocument();
});

test('renderiza investimentos carregados', async () => {
  renderWithProviders(<Investimentos />);
  expect(await screen.findByText(/CDB/i)).toBeInTheDocument();
});

test('abre dialog de simulacao', async () => {
  renderWithProviders(<Investimentos />);
  const simBtn = await screen.findByText(/Simular/i);
  fireEvent.click(simBtn);
  expect(screen.getByText(/Tipo de Investimento/i)).toBeInTheDocument();
});
```

- [ ] **Step 5: Run tests**

```bash
cd /mnt/c/Users/wende/Projects/aurix-platform/apps/frontend/aurix-web
npx react-scripts test --watchAll=false --noStackTrace 2>&1 | tail -40
```
Expected: ~13 tests pass (4 PIX + 3 Cartoes + 3 Credito + 3 Investimentos)

- [ ] **Step 6: Commit**

```bash
git add apps/frontend/aurix-web/src/pages/PIX.test.js apps/frontend/aurix-web/src/pages/Cartoes.test.js apps/frontend/aurix-web/src/pages/Credito.test.js apps/frontend/aurix-web/src/pages/Investimentos.test.js
git commit -m "test(web): PIX, Cartoes, Credito, Investimentos page tests"
```

---

### Task 6: Page Tests — Onboarding + Perfil + Configuracoes

**Files:**
- Create: `apps/frontend/aurix-web/src/pages/Onboarding.test.js`
- Create: `apps/frontend/aurix-web/src/pages/Perfil.test.js`
- Create: `apps/frontend/aurix-web/src/pages/Configuracoes.test.js`

- [ ] **Step 1: Create `Onboarding.test.js`**

```js
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { renderWithProviders, mockUser } from '../../test-utils';
import Onboarding from './Onboarding';

beforeEach(() => {
  global.fetch = jest.fn().mockResolvedValue({
    ok: true,
    json: () => Promise.resolve({ id: '123' }),
  });
});

afterEach(() => {
  delete global.fetch;
});

test('renderiza campos do formulario', () => {
  renderWithProviders(<Onboarding />);
  expect(screen.getByLabelText(/CPF/i)).toBeInTheDocument();
  expect(screen.getByLabelText(/Nome/i)).toBeInTheDocument();
  expect(screen.getByLabelText(/E-mail/i)).toBeInTheDocument();
  expect(screen.getByLabelText(/Telefone/i)).toBeInTheDocument();
});

test('envia dados ao submeter', async () => {
  renderWithProviders(<Onboarding />);
  fireEvent.change(screen.getByLabelText(/CPF/i), { target: { value: '12345678901' } });
  fireEvent.change(screen.getByLabelText(/Nome/i), { target: { value: 'Joao' } });
  fireEvent.click(screen.getByText(/Enviar/i));
  await waitFor(() => expect(global.fetch).toHaveBeenCalled());
});
```

- [ ] **Step 2: Create `Perfil.test.js`**

```js
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { renderWithProviders, mockUser } from '../../test-utils';
import Perfil from './Perfil';

test('renderiza dados do usuario', () => {
  renderWithProviders(<Perfil />);
  expect(screen.getByDisplayValue(/Maria Silva/i)).toBeInTheDocument();
  expect(screen.getByDisplayValue(/maria@test.com/i)).toBeInTheDocument();
});

test('habilita edicao ao clicar em Editar', () => {
  renderWithProviders(<Perfil />);
  const nomeInput = screen.getByDisplayValue(/Maria Silva/i);
  expect(nomeInput).toBeDisabled();
  fireEvent.click(screen.getByText(/Editar/i));
  expect(nomeInput).not.toBeDisabled();
});

test('cancela edicao e restaura valores', () => {
  renderWithProviders(<Perfil />);
  fireEvent.click(screen.getByText(/Editar/i));
  fireEvent.change(screen.getByDisplayValue(/Maria Silva/i), { target: { value: 'Outro Nome' } });
  fireEvent.click(screen.getByText(/Cancelar/i));
  expect(screen.getByDisplayValue(/Maria Silva/i)).toBeInTheDocument();
});
```

- [ ] **Step 3: Create `Configuracoes.test.js`**

```js
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import { renderWithProviders, mockUser } from '../../test-utils';
import Configuracoes from './Configuracoes';

test('renderiza secoes de configuracao', () => {
  renderWithProviders(<Configuracoes />);
  expect(screen.getByText(/Seguranca/i)).toBeInTheDocument();
  expect(screen.getByText(/Notificacoes/i)).toBeInTheDocument();
  expect(screen.getByText(/Preferencias/i)).toBeInTheDocument();
});

test('alterna switch de 2FA', () => {
  renderWithProviders(<Configuracoes />);
  const switches = screen.getAllByRole('checkbox');
  fireEvent.click(switches[0]);
});

test('altera idioma', () => {
  renderWithProviders(<Configuracoes />);
  const select = screen.getByDisplayValue(/portugues/i);
  expect(select).toBeInTheDocument();
});
```

- [ ] **Step 5: Run tests**

```bash
cd /mnt/c/Users/wende/Projects/aurix-platform/apps/frontend/aurix-web
npx react-scripts test --watchAll=false --noStackTrace 2>&1 | tail -40
```
Expected: ~9 tests pass (2 Onboarding + 3 Perfil + 3 Configuracoes)

- [ ] **Step 7: Commit**

```bash
git add apps/frontend/aurix-web/src/pages/Onboarding.test.js apps/frontend/aurix-web/src/pages/Perfil.test.js apps/frontend/aurix-web/src/pages/Configuracoes.test.js
git commit -m "test(web): Onboarding, Perfil, Configuracoes page tests"
```

---

### Task 7: App Integration Test (Auth + Routing)

**Files:**
- Modify: `apps/frontend/aurix-web/src/App.test.js`

- [ ] **Step 1: Write `App.test.js`**

```js
jest.mock('./services/authService');

import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import authService from './services/authService';
import App from './App';

function renderApp() {
  return render(<BrowserRouter><App /></BrowserRouter>);
}

beforeEach(() => {
  localStorage.clear();
  jest.clearAllMocks();
});

test('renderiza Login quando nao autenticado', () => {
  authService.getCurrentUser.mockRejectedValue(new Error('Not authenticated'));
  renderApp();
  expect(screen.getByLabelText(/CPF/i)).toBeInTheDocument();
});

test('renderiza Dashboard apos autenticacao', async () => {
  authService.getCurrentUser.mockResolvedValue({ data: { nome: 'Maria', email: 'maria@test.com', conta: { saldo: 10000 } } });
  renderApp();
  expect(await screen.findByText(/Dashboard/i)).toBeInTheDocument();
});

test('login bem sucedido redireciona para Dashboard', async () => {
  authService.getCurrentUser.mockRejectedValue(new Error('Not authenticated'));
  authService.login.mockResolvedValue({ data: { token: 'new-token', user: { nome: 'Maria' } } });
  renderApp();
  fireEvent.change(screen.getByLabelText(/CPF/i), { target: { value: '12345678901' } });
  fireEvent.change(screen.getByLabelText(/Senha/i), { target: { value: 'senha123' } });
  fireEvent.click(screen.getByRole('button', { name: /Acessar/i }));
  expect(await screen.findByText(/Dashboard/i)).toBeInTheDocument();
});

test('login falho mostra erro', async () => {
  authService.getCurrentUser.mockRejectedValue(new Error('Not authenticated'));
  authService.login.mockRejectedValue(new Error('Credenciais invalidas'));
  renderApp();
  fireEvent.change(screen.getByLabelText(/CPF/i), { target: { value: '123' } });
  fireEvent.change(screen.getByLabelText(/Senha/i), { target: { value: 'errada' } });
  fireEvent.click(screen.getByRole('button', { name: /Acessar/i }));
  expect(await screen.findByText(/Credenciais invalidas/i)).toBeInTheDocument();
});

test('logout redireciona para Login', async () => {
  authService.getCurrentUser.mockResolvedValue({ data: { nome: 'Maria', email: 'maria@test.com', conta: { saldo: 10000 } } });
  renderApp();
  expect(await screen.findByText(/Dashboard/i)).toBeInTheDocument();
  fireEvent.click(screen.getByText(/MS/i));
  fireEvent.click(screen.getByText(/Sair/i));
  expect(await screen.findByLabelText(/CPF/i)).toBeInTheDocument();
});
```

- [ ] **Step 2: Run all tests**

```bash
cd /mnt/c/Users/wende/Projects/aurix-platform/apps/frontend/aurix-web
npx react-scripts test --watchAll=false --noStackTrace 2>&1 | tail -40
```
Expected: All ~42 tests pass (12 service + 17 component + 13 page + 5 App)

- [ ] **Step 3: Commit**

```bash
git add apps/frontend/aurix-web/src/App.test.js
git commit -m "test(web): App auth flow and routing integration tests"
```

---

### Task 8: Full Verification

- [ ] **Step 1: Run full test suite**

```bash
cd /mnt/c/Users/wende/Projects/aurix-platform/apps/frontend/aurix-web
npx react-scripts test --watchAll=false 2>&1 | grep -E "Tests:|Test Suites:|PASS|FAIL"
```
Expected: All suites passing, 0 failures.

- [ ] **Step 2: Run lint (if linting is set up)**

```bash
cd /mnt/c/Users/wende/Projects/aurix-platform/apps/frontend/aurix-web
npx eslint src --ext .js,.jsx 2>&1 | tail -10
```

- [ ] **Step 3: Commit any final adjustments**

```bash
git add -A && git diff --cached --stat
# verify only intended files
```
