# Mobile Onboarding — Design

## Goal
Create PF and PJ onboarding screens for `aurix-mobile` (React Native 0.73), matching the existing web onboarding in API contract but with a mobile-optimized UX.

## Architecture

### Navigation
`AuthNavigator` (Stack) — `initialRouteName: "Onboarding"`:
```
Onboarding → TipoSelector → FormPF | StepEmpresa → StepSocios
                                                → SuccessScreen
                                          → SuccessScreen
```

### API Service
New `src/services/onboardingService.js` using `axios`:

| Method | Endpoint |
|--------|----------|
| `criarSolicitacaoPF(dados)` | `POST /api/onboarding/contas/pf/solicitacoes` |
| `criarSolicitacaoPJ(dados)` | `POST /api/onboarding/contas/pj` |
| `consultarStatus(id)` | `GET /onboarding/contas/pf/solicitacoes/{id}` (PF) ou `/onboarding/contas/pj/{id}` (PJ) |

Base URL: `http://localhost:8080/api` (shared with `authService.js`). Timeout 15s.

### Screen Flow

#### TipoSelector
- Two large cards: "Pessoa Física" (user icon) and "Pessoa Jurídica" (building icon)
- Tapping a card navigates to the respective form
- Styled consistently with existing mobile screens (LinearGradient cards, rounded corners)

#### FormPF (single scrollable screen)
Groups with section headers and card containers:

1. **Dados Pessoais** — CPF (xxx.xxx.xxx-xx mask), Nome, Data Nascimento (datetimepicker), Ocupação
2. **Contato** — Email, Telefone ((xx) xxxxx-xxxx mask)
3. **Financeiro** — Renda Declarada (R$ currency format)
4. **Endereço** — CEP (xxxxx-xxx mask), Logradouro, Número, Bairro, Cidade, UF (picker)

Validation: required fields, CPF checksum, email regex, CEP format. Inline error messages below each field. Button "Enviar Solicitação" at bottom disables while submitting.

On success: navigate to `SuccessScreen` with protocol number.
On error: show Alert with server message.

#### StepEmpresa (PJ passo 1)
- CNPJ (xx.xxx.xxx/xxxx-xx mask), Razão Social, Nome Fantasia
- Email, Telefone
- Endereço (CEP, Logradouro, Número, Bairro, Cidade, UF)
- "Validar CNPJ" button → POST to validation endpoint → if valid, store CNPJ data and advance
- "Próximo" button (enabled after validation)

#### StepSocios (PJ passo 2)
- List of added partners (name, CPF, qualification)
- "Adicionar Sócio" button → opens Modal with: CPF, Nome, Email, Qualificação (picker: Sócio-Administrador / Sócio / Diretor / Outro)
- Swipe-to-delete or "Remover" button per partner
- "Revisar e Enviar" → POST to create solicitation with all socios
- On success → SuccessScreen

#### SuccessScreen
- Checkmark icon, "Solicitação enviada!" title
- Protocol number displayed
- "Acompanhar Status" button → goes to tracking (future) or LoginScreen
- "Voltar ao Início" → LoginScreen

### Dependencies
Need to be declared in `package.json`:
- `axios` (new)
- `@react-native-community/datetimepicker` (date picker)
- Manual masks via JS functions (avoid adding heavy mask lib)

Already used but undeclared (hoisted from monorepo root):
- `@react-navigation/native`, `@react-navigation/stack`, `@react-navigation/bottom-tabs`
- `react-native-vector-icons`, `react-native-linear-gradient`
- `react-native-gesture-handler`, `react-native-safe-area-context`

### State Management
- Screen-local `useState` for form fields
- `useState` for validation errors (object keyed by field name)
- `useState` for loading/submitting state
- No global state needed — forms are transient

### Error Handling
- Network error: Alert "Erro de conexão. Verifique sua internet."
- Validation error: inline below field, red text
- API error (4xx): Alert with server message
- API error (5xx): Alert "Erro interno. Tente novamente."

### Testing
- `__tests__/OnboardingService.test.js` — API calls mocked via jest
- `__tests__/FormPF.test.js` — render, fill fields, validate, submit
- `__tests__/TipoSelector.test.js` — render both cards, tap navigates
- Jest preset: `react-native`

### Files to create
| File | Purpose |
|------|---------|
| `src/services/onboardingService.js` | Axios HTTP client for onboarding API |
| `src/pages/onboarding/TipoSelector.js` | PF/PJ card selector |
| `src/pages/onboarding/FormPF.js` | PF form (scrollable, single screen) |
| `src/pages/onboarding/StepEmpresa.js` | PJ company info step |
| `src/pages/onboarding/StepSocios.js` | PJ partners step |
| `src/pages/onboarding/SuccessScreen.js` | Post-submit success screen |
| `src/pages/OnboardingScreen.js` | Orchestrator (replaces placeholder) |
| `src/__tests__/OnboardingService.test.js` | API service tests |
| `src/__tests__/TipoSelector.test.js` | TipoSelector tests |
| `src/__tests__/FormPF.test.js` | FormPF tests |

### Files to modify
| File | Change |
|------|--------|
| `package.json` | Add missing deps (axios, datetimepicker, navigation, etc.) |
| `App.js` | Wire `OnboardingScreen` into auth flow (already referenced) |

### Depends on
- Existing `Colors.js` constants
- Existing `apiService.js`-style patterns (but mobile uses raw axios)
- `AuthNavigator.js` already has `Onboarding` route registered
