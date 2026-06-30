# aureus-web Test Coverage Design

> **Sub-project 1 of 3** (aureus-web first, aureus-admin and aureus-mobile follow)

**Goal:** Full test coverage for the Internet Banking portal (aureus-web) — all components, pages, services, and App routing.

**Architecture:** Jest (via CRA react-scripts) + @testing-library/react. Mock services via `jest.mock()` + `__mocks__` directory. Shared render wrapper for MUI ThemeProvider + BrowserRouter.

**Tech Stack:** Jest 29, @testing-library/react 13, @testing-library/jest-dom 5, react-scripts 5 (CRA)

## Files

### New
- `aureus-web/src/setupTests.js` — matchMedia mock + jest-dom import
- `aureus-web/src/services/__mocks__/apiService.js` — mock factory for all API methods
- `aureus-web/src/services/__mocks__/authService.js` — mock factory for auth methods
- `aureus-web/src/test-utils.js` — shared `renderWithProviders` wrapper
- `aureus-web/src/__mocks__/axios.js` — mock axios for service tests (optional, can use jest.mock inline)

### Test Suites

| # | Suite | File | Tests | Category |
|---|-------|------|-------|----------|
| 1 | setup | `setupTests.js` | — | infra |
| 2 | apiService | `services/apiService.test.js` | ~6 | service |
| 3 | authService | `services/authService.test.js` | ~5 | service |
| 4 | App (auth/routing) | `App.test.js` | ~8 | integration |
| 5 | Navbar | `components/Navbar.test.js` | ~6 | component |
| 6 | Sidebar | `components/Sidebar.test.js` | ~4 | component |
| 7 | Dashboard | `components/Dashboard.test.js` | ~8 | component |
| 8 | Login | `pages/Login.test.js` | ~8 | page |
| 9 | Dashboard | `pages/Dashboard.test.js` | ~8 | page |
| 10 | Contas | `pages/Contas.test.js` | ~8 | page |
| 11 | Transacoes | `pages/Transacoes.test.js` | ~6 | page |
| 12 | PIX | `pages/PIX.test.js` | ~10 | page |
| 13 | Cartoes | `pages/Cartoes.test.js` | ~10 | page |
| 14 | Credito | `pages/Credito.test.js` | ~4 | page |
| 15 | Investimentos | `pages/Investimentos.test.js` | ~8 | page |
| 16 | Onboarding | `pages/Onboarding.test.js` | ~4 | page |
| 17 | Perfil | `pages/Perfil.test.js` | ~6 | page |
| 18 | Configuracoes | `pages/Configuracoes.test.js` | ~4 | page |

## Test Patterns

### Pattern 1: Mock Services
All pages that call `apiService` use the shared mock. Mock data lives alongside the test file or in a shared `test-utils.js`.

```js
// setup all api methods to return mock data
jest.mock('../../services/apiService');
```

### Pattern 2: renderWithProviders
Every component/page test uses this wrapper:

```js
import { render } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { ThemeProvider, createTheme } from '@mui/material/styles';

const theme = createTheme();
const mockUser = { nome: 'Test', email: 'test@test.com', conta: { saldo: 10000 } };

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

### Pattern 3: Three-State Tests
Each page tests three states:

1. **Loading** — `screen.getByRole('progressbar')` or similar
2. **Success** — `await screen.findByText(/dado esperado/i)`
3. **Error** — mock throws, verify error message rendered

Pages with mock data inline (Dashboard, Contas) skip loading/error states and test purely rendering + interactions.

### Pattern 4: Interaction Tests
- Form submit: `fireEvent.change(input, { target: { value } })` + `fireEvent.click(submitBtn)` → verify mock was called with expected data
- Dialog open/close: click button → dialog appears → click close → dialog gone
- Tab switching: click tab → verify content changes
- Filter: change select/date → verify table refreshes

## Service Test Patterns

### apiService.test.js
```js
jest.mock('axios');
import axios from 'axios';
import api from './apiService';

beforeEach(() => { localStorage.setItem('aureus_token', 'test-token'); });
afterEach(() => { localStorage.clear(); });

test('adiciona token no header', ...);
test('getContas faz GET /contas', ...);
test('enviarPix faz POST /pix/enviar', ...);
test('401 redireciona para login', ...);
```

### authService.test.js
```js
jest.mock('axios');
import axios from 'axios';
import auth from './authService';

test('login faz POST /auth/login', ...);
test('getCurrentUser faz GET /auth/me', ...);
test('logout limpa localStorage', ...);
test('requestPasswordReset faz POST /auth/password-reset', ...);
```

## Test Data (Mock Data Sets)

Centralized mock data in `src/test-utils.js`:

```js
export const mockContas = [
  { id: '1', tipo: 'CORRENTE', saldo: 15750.50, numero: '12345-6', agencia: '0001', status: 'ATIVA' },
  { id: '2', tipo: 'POUPANCA', saldo: 25000, numero: '12345-7', agencia: '0001', status: 'ATIVA' },
];

export const mockTransacoes = [
  { id: '1', tipo: 'PIX', valor: 1500, descricao: 'Transferencia', status: 'CONCLUIDA' },
];

export const mockInvestimentos = [...];
export const mockCartoes = [...];
export const mockFaturas = [...];
export const mockUser = { nome: 'Maria Silva', email: 'maria@test.com', conta: { saldo: 15750.50 } };
```
