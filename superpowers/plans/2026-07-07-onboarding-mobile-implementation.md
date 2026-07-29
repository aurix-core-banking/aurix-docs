# Mobile Onboarding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build PF and PJ onboarding screens for aurix-mobile with form validation, API integration, and navigation.

**Architecture:** Stack navigation (AuthNavigator) with OnboardingScreen orchestrator → TipoSelector → FormPF or StepEmpresa→StepSocios → SuccessScreen. Singleton axios service class. Screen-local `useState`. Shared validation utils.

**Tech Stack:** React Native 0.73, @react-navigation/stack, axios, @react-native-community/datetimepicker, Jest + @testing-library/react-native

## Global Constraints

- React Native 0.73, React 18.2, plain JavaScript (`.js` files, no TS)
- Follow existing patterns: singleton class services (`export const service = new Service()`), `StyleSheet.create` for all styles, `Colors.js` constants, Portuguese labels
- No global state — form state lives in useState per screen
- All new screens use `export default function ScreenName({ navigation, route })` (matching existing LoginScreen pattern)
- OnboardingService uses `axios.create()` with base URL `http://localhost:8080/api`, timeout 15s
- Validation utils in `src/utils/validation.js`: pure functions, exported individually
- Jest preset `react-native`, tests with `@testing-library/react-native`, axios mocking via `axios-mock-adapter`
- Base path for all file references: `frontend/aurix-mobile/`

---

### Task 1: Infrastructure Setup (package.json + validation utils + test mocks)

**Files:**
- Modify: `package.json`
- Create: `src/utils/validation.js`
- Create: `__mocks__/react-native-linear-gradient/index.js`
- Create: `__mocks__/react-native-vector-icons/MaterialIcons.js`
- Create: `babel.config.js`

**Interfaces:**
- Consumes: nothing
- Produces: `validarCPF(str)`, `validarCNPJ(str)`, `validarEmail(str)`, `validarCEP(str)`, `formatarCPF(str)`, `formatarCNPJ(str)`, `formatarTelefone(str)`, `formatarCEP(str)`, `formatarMoeda(num)` — all pure functions

- [ ] **Step 1: Update package.json with dependencies**

```json
{
  "dependencies": {
    "react": "18.2.0",
    "react-native": "0.73.0",
    "axios": "^1.7.0",
    "@react-native-community/datetimepicker": "^7.6.0",
    "@react-navigation/native": "^6.1.0",
    "@react-navigation/stack": "^6.3.0",
    "react-native-vector-icons": "^10.0.0",
    "react-native-linear-gradient": "^2.8.0",
    "react-native-gesture-handler": "^2.14.0",
    "react-native-safe-area-context": "^4.8.0",
    "react-query": "^3.39.0",
    "@react-native-async-storage/async-storage": "^1.21.0",
    "react-native-encrypted-storage": "^4.0.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-native": "^0.73.0",
    "eslint": "^8.0.0",
    "jest": "^29.0.0",
    "typescript": "^5.0.0",
    "@testing-library/react-native": "^12.4.0",
    "axios-mock-adapter": "^1.22.0"
  },
  "jest": {
    "preset": "react-native",
    "transformIgnorePatterns": [
      "node_modules/(?!(react-native|@react-native|@react-navigation|react-native-vector-icons|react-native-linear-gradient|react-native-gesture-handler|react-native-safe-area-context)/)"
    ]
  }
}
```

- [ ] **Step 2: Create babel.config.js**

```js
module.exports = {
  presets: ['module:@react-native/babel-preset'],
};
```

- [ ] **Step 3: Create mock for react-native-linear-gradient**

`__mocks__/react-native-linear-gradient/index.js`:
```js
import React from 'react';
import { View } from 'react-native';

const LinearGradient = ({ children, style, colors, ...props }) => {
  return <View style={[{ backgroundColor: colors ? colors[0] : '#fff' }, style]}>{children}</View>;
};

export default LinearGradient;
```

- [ ] **Step 4: Create mock for react-native-vector-icons**

`__mocks__/react-native-vector-icons/MaterialIcons.js`:
```js
import React from 'react';
import { View } from 'react-native';

const Icon = ({ name, size, color, style }) => {
  return <View style={[{ width: size || 20, height: size || 20 }, style]} />;
};

export default Icon;
```

- [ ] **Step 5: Create validation utils**

`src/utils/validation.js`:
```js
export function validarCPF(value) {
  const cpf = value.replace(/\D/g, '');
  if (cpf.length !== 11) return false;
  if (/^(\d)\1{10}$/.test(cpf)) return false;

  let sum = 0;
  for (let i = 0; i < 9; i++) sum += parseInt(cpf[i]) * (10 - i);
  let remainder = (sum * 10) % 11;
  if (remainder === 10) remainder = 0;
  if (remainder !== parseInt(cpf[9])) return false;

  sum = 0;
  for (let i = 0; i < 10; i++) sum += parseInt(cpf[i]) * (11 - i);
  remainder = (sum * 10) % 11;
  if (remainder === 10) remainder = 0;
  if (remainder !== parseInt(cpf[10])) return false;

  return true;
}

export function validarCNPJ(value) {
  const cnpj = value.replace(/\D/g, '');
  if (cnpj.length !== 14) return false;
  if (/^(\d)\1{13}$/.test(cnpj)) return false;

  let size = 12;
  let numbers = cnpj.substring(0, size);
  let digits = cnpj.substring(size);
  let sum = 0;
  let pos = size - 7;
  for (let i = size; i >= 1; i--) {
    sum += parseInt(numbers.charAt(size - i)) * pos--;
    if (pos < 2) pos = 9;
  }
  let result = sum % 11 < 2 ? 0 : 11 - (sum % 11);
  if (result !== parseInt(digits.charAt(0))) return false;

  size = 13;
  numbers = cnpj.substring(0, size);
  sum = 0;
  pos = size - 7;
  for (let i = size; i >= 1; i--) {
    sum += parseInt(numbers.charAt(size - i)) * pos--;
    if (pos < 2) pos = 9;
  }
  result = sum % 11 < 2 ? 0 : 11 - (sum % 11);
  if (result !== parseInt(digits.charAt(1))) return false;

  return true;
}

export function validarEmail(value) {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
}

export function validarCEP(value) {
  return /^\d{5}-?\d{3}$/.test(value.replace(/\D/g, ''));
}

export function formatarCPF(value) {
  const digits = value.replace(/\D/g, '').substring(0, 11);
  return digits
    .replace(/(\d{3})(\d)/, '$1.$2')
    .replace(/(\d{3})(\d)/, '$1.$2')
    .replace(/(\d{3})(\d{1,2})$/, '$1-$2');
}

export function formatarCNPJ(value) {
  const digits = value.replace(/\D/g, '').substring(0, 14);
  return digits
    .replace(/(\d{2})(\d)/, '$1.$2')
    .replace(/(\d{3})(\d)/, '$1.$2')
    .replace(/(\d{3})(\d)/, '$1/$2')
    .replace(/(\d{4})(\d{1,2})$/, '$1-$2');
}

export function formatarTelefone(value) {
  const digits = value.replace(/\D/g, '').substring(0, 11);
  if (digits.length <= 10) {
    return digits.replace(/(\d{2})(\d{4})(\d{4})/, '($1) $2-$3');
  }
  return digits.replace(/(\d{2})(\d{5})(\d{4})/, '($1) $2-$3');
}

export function formatarCEP(value) {
  const digits = value.replace(/\D/g, '').substring(0, 8);
  return digits.replace(/(\d{5})(\d{3})/, '$1-$2');
}

export function formatarMoeda(value) {
  const num = typeof value === 'string' ? parseFloat(value.replace(/\D/g, '')) / 100 : value;
  if (isNaN(num)) return '';
  return `R$ ${num.toFixed(2).replace('.', ',').replace(/\B(?=(\d{3})+(?!\d))/g, '.')}`;
}
```

- [ ] **Step 6: Verify tests pass**

Run: `cd frontend/aurix-mobile && npx jest --passWithNoTests`
Expected: `Tests: 0 total` (no test files yet, but no errors)

- [ ] **Step 7: Commit**

```bash
git add frontend/aurix-mobile/package.json frontend/aurix-mobile/babel.config.js frontend/aurix-mobile/__mocks__/ frontend/aurix-mobile/src/utils/validation.js
git commit -m "feat(mobile): add deps, validation utils, test mocks for onboarding"
```

---

### Task 2: Onboarding Service + Tests

**Files:**
- Create: `src/services/onboardingService.js`
- Create: `src/__tests__/OnboardingService.test.js`

**Interfaces:**
- Consumes: `axios` (from package.json), class singleton pattern matching `authService.js`
- Produces: `onboardingService.criarSolicitacaoPF(dados)` → Promise(response), `onboardingService.criarSolicitacaoPJ(dados)` → Promise(response), `onboardingService.consultarStatus(id, tipo)` → Promise(response)

- [ ] **Step 1: Write the failing tests**

`src/__tests__/OnboardingService.test.js`:
```js
import MockAdapter from 'axios-mock-adapter';
import { onboardingService } from '../services/onboardingService';

const mockAxios = new MockAdapter(onboardingService.api);

describe('onboardingService', () => {
  afterEach(() => {
    mockAxios.reset();
  });

  it('criarSolicitacaoPF posts to correct endpoint and returns data', async () => {
    const dados = { cpf: '12345678901', nome: 'João' };
    const resposta = { id: 1, protocolo: 'PROTO-001', status: 'RECEBIDA' };
    mockAxios.onPost('/onboarding/contas/pf/solicitacoes').reply(201, resposta);

    const result = await onboardingService.criarSolicitacaoPF(dados);

    expect(result).toEqual(resposta);
    expect(mockAxios.history.post[0].url).toBe('/onboarding/contas/pf/solicitacoes');
    expect(JSON.parse(mockAxios.history.post[0].data)).toEqual(dados);
  });

  it('criarSolicitacaoPJ posts to correct endpoint and returns data', async () => {
    const dados = { cnpj: '11222333000181', razaoSocial: 'Empresa Ltda', socios: [] };
    const resposta = { id: 2, protocolo: 'PROTO-002', status: 'RECEBIDA' };
    mockAxios.onPost('/onboarding/contas/pj').reply(201, resposta);

    const result = await onboardingService.criarSolicitacaoPJ(dados);

    expect(result).toEqual(resposta);
  });

  it('consultarStatus fetches PF solicitation by id', async () => {
    const resposta = { id: 1, protocolo: 'PROTO-001', status: 'EM_ANALISE' };
    mockAxios.onGet('/onboarding/contas/pf/solicitacoes/1').reply(200, resposta);

    const result = await onboardingService.consultarStatus(1, 'PF');

    expect(result).toEqual(resposta);
  });

  it('consultarStatus fetches PJ solicitation by id', async () => {
    const resposta = { id: 2, protocolo: 'PROTO-002', status: 'EM_ANALISE' };
    mockAxios.onGet('/onboarding/contas/pj/2').reply(200, resposta);

    const result = await onboardingService.consultarStatus(2, 'PJ');

    expect(result).toEqual(resposta);
  });

  it('throws error on network failure', async () => {
    mockAxios.onPost('/onboarding/contas/pf/solicitacoes').networkError();

    await expect(onboardingService.criarSolicitacaoPF({})).rejects.toThrow();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend/aurix-mobile && npx jest src/__tests__/OnboardingService.test.js --no-cache`
Expected: FAIL — "Cannot find module '../services/onboardingService'"

- [ ] **Step 3: Write minimal implementation**

`src/services/onboardingService.js`:
```js
import axios from 'axios';

const API_BASE_URL = 'http://localhost:8080/api';

class OnboardingService {
  constructor() {
    this.api = axios.create({
      baseURL: API_BASE_URL,
      timeout: 15000,
      headers: { 'Content-Type': 'application/json' },
    });
  }

  async criarSolicitacaoPF(dados) {
    const response = await this.api.post('/onboarding/contas/pf/solicitacoes', dados);
    return response.data;
  }

  async criarSolicitacaoPJ(dados) {
    const response = await this.api.post('/onboarding/contas/pj', dados);
    return response.data;
  }

  async consultarStatus(id, tipo = 'PF') {
    const path = tipo === 'PF'
      ? `/onboarding/contas/pf/solicitacoes/${id}`
      : `/onboarding/contas/pj/${id}`;
    const response = await this.api.get(path);
    return response.data;
  }
}

export const onboardingService = new OnboardingService();
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend/aurix-mobile && npx jest src/__tests__/OnboardingService.test.js --no-cache`
Expected: PASS (5 tests)

- [ ] **Step 5: Commit**

```bash
git add frontend/aurix-mobile/src/services/onboardingService.js frontend/aurix-mobile/src/__tests__/OnboardingService.test.js
git commit -m "feat(mobile): add onboarding service with axios + tests"
```

---

### Task 3: TipoSelector Screen + Tests

**Files:**
- Create: `src/pages/onboarding/TipoSelector.js`
- Create: `src/__tests__/TipoSelector.test.js`

**Interfaces:**
- Consumes: `navigation.navigate('FormPF')`, `navigation.navigate('StepEmpresa')`
- Produces: rendered screen with two large cards (PF card, PJ card), consistent with LoginScreen styling

- [ ] **Step 1: Write the failing tests**

`src/__tests__/TipoSelector.test.js`:
```js
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import TipoSelector from '../pages/onboarding/TipoSelector';

const mockNavigate = jest.fn();
const props = { navigation: { navigate: mockNavigate } };

describe('TipoSelector', () => {
  beforeEach(() => {
    mockNavigate.mockClear();
  });

  it('renders both PF and PJ cards', () => {
    const { getByText } = render(<TipoSelector {...props} />);
    expect(getByText('Pessoa Física')).toBeTruthy();
    expect(getByText('Pessoa Jurídica')).toBeTruthy();
  });

  it('renders app title and subtitle', () => {
    const { getByText } = render(<TipoSelector {...props} />);
    expect(getByText('AURIX Banking')).toBeTruthy();
    expect(getByText('Abra sua conta')).toBeTruthy();
  });

  it('navigates to FormPF when PF card is pressed', () => {
    const { getByText } = render(<TipoSelector {...props} />);
    fireEvent.press(getByText('Pessoa Física'));
    expect(mockNavigate).toHaveBeenCalledWith('FormPF');
  });

  it('navigates to StepEmpresa when PJ card is pressed', () => {
    const { getByText } = render(<TipoSelector {...props} />);
    fireEvent.press(getByText('Pessoa Jurídica'));
    expect(mockNavigate).toHaveBeenCalledWith('StepEmpresa');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend/aurix-mobile && npx jest src/__tests__/TipoSelector.test.js --no-cache`
Expected: FAIL — "Cannot find module '../pages/onboarding/TipoSelector'"

- [ ] **Step 3: Write minimal implementation**

`src/pages/onboarding/TipoSelector.js`:
```js
import React from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  Dimensions,
  ScrollView,
} from 'react-native';
import LinearGradient from 'react-native-linear-gradient';
import Icon from 'react-native-vector-icons/MaterialIcons';
import { Colors } from '../../constants/Colors';

const { width } = Dimensions.get('window');
const CARD_WIDTH = width * 0.4;

const TipoSelector = ({ navigation }) => {
  return (
    <LinearGradient colors={Colors.gradientPrimary} style={styles.container}>
      <ScrollView contentContainerStyle={styles.content}>
        <View style={styles.header}>
          <View style={styles.logoCircle}>
            <Text style={styles.logoText}>A</Text>
          </View>
          <Text style={styles.title}>AURIX Banking</Text>
          <Text style={styles.subtitle}>Abra sua conta</Text>
        </View>

        <View style={styles.cardsRow}>
          <TouchableOpacity
            style={styles.card}
            onPress={() => navigation.navigate('FormPF')}
            activeOpacity={0.8}
          >
            <View style={styles.cardIconContainer}>
              <Icon name="person" size={48} color={Colors.primary} />
            </View>
            <Text style={styles.cardTitle}>Pessoa Física</Text>
            <Text style={styles.cardDescription}>
              Conta pessoal com todos os benefícios
            </Text>
          </TouchableOpacity>

          <TouchableOpacity
            style={styles.card}
            onPress={() => navigation.navigate('StepEmpresa')}
            activeOpacity={0.8}
          >
            <View style={styles.cardIconContainer}>
              <Icon name="business" size={48} color={Colors.primary} />
            </View>
            <Text style={styles.cardTitle}>Pessoa Jurídica</Text>
            <Text style={styles.cardDescription}>
              Soluções empresariais completas
            </Text>
          </TouchableOpacity>
        </View>
      </ScrollView>
    </LinearGradient>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
  },
  content: {
    flexGrow: 1,
    justifyContent: 'center',
    alignItems: 'center',
    paddingHorizontal: 20,
  },
  header: {
    alignItems: 'center',
    marginBottom: 50,
  },
  logoCircle: {
    width: 72,
    height: 72,
    borderRadius: 36,
    backgroundColor: Colors.white,
    justifyContent: 'center',
    alignItems: 'center',
    marginBottom: 16,
    elevation: 6,
    shadowColor: Colors.black,
    shadowOffset: { width: 0, height: 3 },
    shadowOpacity: 0.2,
    shadowRadius: 6,
  },
  logoText: {
    fontSize: 32,
    fontWeight: 'bold',
    color: Colors.primary,
  },
  title: {
    fontSize: 28,
    fontWeight: 'bold',
    color: Colors.white,
    marginBottom: 4,
  },
  subtitle: {
    fontSize: 16,
    color: Colors.white,
    opacity: 0.9,
  },
  cardsRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    width: '100%',
    gap: 16,
  },
  card: {
    flex: 1,
    backgroundColor: Colors.white,
    borderRadius: 20,
    padding: 24,
    alignItems: 'center',
    elevation: 8,
    shadowColor: Colors.black,
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.15,
    shadowRadius: 8,
    minHeight: 200,
    justifyContent: 'center',
  },
  cardIconContainer: {
    width: 72,
    height: 72,
    borderRadius: 36,
    backgroundColor: Colors.lightGray,
    justifyContent: 'center',
    alignItems: 'center',
    marginBottom: 16,
  },
  cardTitle: {
    fontSize: 18,
    fontWeight: 'bold',
    color: Colors.text,
    marginBottom: 8,
    textAlign: 'center',
  },
  cardDescription: {
    fontSize: 13,
    color: Colors.textSecondary,
    textAlign: 'center',
    lineHeight: 18,
  },
});

export default TipoSelector;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend/aurix-mobile && npx jest src/__tests__/TipoSelector.test.js --no-cache`
Expected: PASS (4 tests)

- [ ] **Step 5: Commit**

```bash
git add frontend/aurix-mobile/src/pages/onboarding/TipoSelector.js frontend/aurix-mobile/src/__tests__/TipoSelector.test.js
git commit -m "feat(mobile): add TipoSelector screen with PF/PJ cards + tests"
```

---

### Task 4: FormPF Screen + Tests

**Files:**
- Create: `src/pages/onboarding/FormPF.js`
- Create: `src/__tests__/FormPF.test.js`

**Interfaces:**
- Consumes: `onboardingService.criarSolicitacaoPF(dados)`, `navigation.navigate('SuccessScreen', { protocolo })`, `validarCPF()`, `formatarCPF()`, etc. from validation utils
- Produces: scrollable form with grouped fields, inline validation, submit handler

- [ ] **Step 1: Write the failing tests**

`src/__tests__/FormPF.test.js`:
```js
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import FormPF from '../pages/onboarding/FormPF';

const mockNavigate = jest.fn();
const props = { navigation: { navigate: mockNavigate } };

describe('FormPF', () => {
  beforeEach(() => {
    mockNavigate.mockClear();
  });

  it('renders all section headers', () => {
    const { getByText } = render(<FormPF {...props} />);
    expect(getByText('Dados Pessoais')).toBeTruthy();
    expect(getByText('Contato')).toBeTruthy();
    expect(getByText('Financeiro')).toBeTruthy();
    expect(getByText('Endereço')).toBeTruthy();
  });

  it('shows validation errors for empty required fields on submit', () => {
    const { getByText } = render(<FormPF {...props} />);
    fireEvent.press(getByText('Enviar Solicitação'));
    expect(getByText('CPF é obrigatório')).toBeTruthy();
    expect(getByText('Nome é obrigatório')).toBeTruthy();
  });

  it('shows CPF validation error for invalid CPF', () => {
    const { getByText, getByPlaceholderText } = render(<FormPF {...props} />);
    const cpfInput = getByPlaceholderText('000.000.000-00');
    fireEvent.changeText(cpfInput, '123.456.789-00');
    fireEvent.press(getByText('Enviar Solicitação'));
    expect(getByText('CPF inválido')).toBeTruthy();
  });

  it('formats CPF as user types', () => {
    const { getByPlaceholderText } = render(<FormPF {...props} />);
    const cpfInput = getByPlaceholderText('000.000.000-00');
    fireEvent.changeText(cpfInput, '12345678901');
    expect(cpfInput.props.value).toBe('123.456.789-01');
  });

  it('formats phone as user types', () => {
    const { getByPlaceholderText } = render(<FormPF {...props} />);
    const phoneInput = getByPlaceholderText('(00) 00000-0000');
    fireEvent.changeText(phoneInput, '11999998888');
    expect(phoneInput.props.value).toBe('(11) 99999-8888');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend/aurix-mobile && npx jest src/__tests__/FormPF.test.js --no-cache`
Expected: FAIL — "Cannot find module '../pages/onboarding/FormPF'"

- [ ] **Step 3: Write minimal implementation**

`src/pages/onboarding/FormPF.js`:
```js
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  ScrollView,
  Alert,
  KeyboardAvoidingView,
  Platform,
} from 'react-native';
import LinearGradient from 'react-native-linear-gradient';
import Icon from 'react-native-vector-icons/MaterialIcons';
import DateTimePicker from '@react-native-community/datetimepicker';
import { Colors } from '../../constants/Colors';
import {
  validarCPF,
  validarEmail,
  validarCEP,
  formatarCPF,
  formatarTelefone,
  formatarCEP,
  formatarMoeda,
} from '../../utils/validation';
import { onboardingService } from '../../services/onboardingService';

const UFS = [
  'AC','AL','AP','AM','BA','CE','DF','ES','GO','MA','MT','MS',
  'MG','PA','PB','PR','PE','PI','RJ','RN','RS','RO','RR','SC',
  'SP','SE','TO',
];

const FormPF = ({ navigation }) => {
  const [formData, setFormData] = useState({
    cpf: '',
    nome: '',
    dataNascimento: new Date(),
    ocupacao: '',
    email: '',
    telefone: '',
    rendaDeclarada: '',
    cep: '',
    logradouro: '',
    numero: '',
    bairro: '',
    cidade: '',
    uf: '',
  });
  const [showDatePicker, setShowDatePicker] = useState(false);
  const [errors, setErrors] = useState({});
  const [loading, setLoading] = useState(false);

  const handleChange = (field, value) => {
    let formatted = value;
    if (field === 'cpf') formatted = formatarCPF(value);
    else if (field === 'telefone') formatted = formatarTelefone(value);
    else if (field === 'cep') formatted = formatarCEP(value);

    setFormData(prev => ({ ...prev, [field]: formatted }));
    if (errors[field]) setErrors(prev => ({ ...prev, [field]: '' }));
  };

  const handleDateChange = (event, selectedDate) => {
    setShowDatePicker(false);
    if (selectedDate) {
      setFormData(prev => ({ ...prev, dataNascimento: selectedDate }));
    }
  };

  const validate = () => {
    const newErrors = {};
    if (!formData.cpf) newErrors.cpf = 'CPF é obrigatório';
    else if (!validarCPF(formData.cpf)) newErrors.cpf = 'CPF inválido';
    if (!formData.nome) newErrors.nome = 'Nome é obrigatório';
    if (!formData.email) newErrors.email = 'Email é obrigatório';
    else if (!validarEmail(formData.email)) newErrors.email = 'Email inválido';
    if (!formData.telefone) newErrors.telefone = 'Telefone é obrigatório';
    if (!formData.cep) newErrors.cep = 'CEP é obrigatório';
    else if (!validarCEP(formData.cep)) newErrors.cep = 'CEP inválido';
    if (!formData.logradouro) newErrors.logradouro = 'Logradouro é obrigatório';
    if (!formData.numero) newErrors.numero = 'Número é obrigatório';
    if (!formData.bairro) newErrors.bairro = 'Bairro é obrigatório';
    if (!formData.cidade) newErrors.cidade = 'Cidade é obrigatório';
    if (!formData.uf) newErrors.uf = 'UF é obrigatório';
    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleSubmit = async () => {
    if (!validate()) return;
    setLoading(true);
    try {
      const dados = {
        cpf: formData.cpf.replace(/\D/g, ''),
        nome: formData.nome,
        dataNascimento: formData.dataNascimento.toISOString().split('T')[0],
        ocupacao: formData.ocupacao,
        email: formData.email,
        telefone: formData.telefone.replace(/\D/g, ''),
        rendaDeclarada: parseFloat(formData.rendaDeclarada.replace(/\D/g, '')) / 100 || 0,
        endereco: {
          cep: formData.cep.replace(/\D/g, ''),
          logradouro: formData.logradouro,
          numero: formData.numero,
          bairro: formData.bairro,
          cidade: formData.cidade,
          uf: formData.uf,
        },
      };
      const response = await onboardingService.criarSolicitacaoPF(dados);
      navigation.navigate('SuccessScreen', { protocolo: response.protocolo });
    } catch (error) {
      Alert.alert('Erro', 'Erro ao enviar solicitação. Tente novamente.');
    } finally {
      setLoading(false);
    }
  };

  const getInputStyle = (field) => [
    styles.input,
    errors[field] && styles.inputError,
  ];

  const renderError = (field) =>
    errors[field] ? <Text style={styles.errorText}>{errors[field]}</Text> : null;

  return (
    <LinearGradient colors={Colors.gradientPrimary} style={styles.container}>
      <KeyboardAvoidingView
        behavior={Platform.OS === 'ios' ? 'padding' : undefined}
        style={styles.flex}
      >
        <ScrollView contentContainerStyle={styles.scrollContent}>
          <View style={styles.sectionCard}>
            <Text style={styles.sectionTitle}>Dados Pessoais</Text>

            <Text style={styles.label}>CPF</Text>
            <TextInput
              style={getInputStyle('cpf')}
              placeholder="000.000.000-00"
              placeholderTextColor={Colors.gray}
              value={formData.cpf}
              onChangeText={v => handleChange('cpf', v)}
              keyboardType="numeric"
              maxLength={14}
            />
            {renderError('cpf')}

            <Text style={styles.label}>Nome</Text>
            <TextInput
              style={getInputStyle('nome')}
              placeholder="Nome completo"
              placeholderTextColor={Colors.gray}
              value={formData.nome}
              onChangeText={v => handleChange('nome', v)}
            />
            {renderError('nome')}

            <Text style={styles.label}>Data de Nascimento</Text>
            <TouchableOpacity
              style={styles.input}
              onPress={() => setShowDatePicker(true)}
            >
              <Text style={formData.dataNascimento ? styles.inputText : styles.placeholderText}>
                {formData.dataNascimento
                  ? formData.dataNascimento.toLocaleDateString('pt-BR')
                  : 'Selecione a data'}
              </Text>
            </TouchableOpacity>
            {showDatePicker && (
              <DateTimePicker
                value={formData.dataNascimento}
                mode="date"
                display="default"
                onChange={handleDateChange}
                maximumDate={new Date()}
              />
            )}

            <Text style={styles.label}>Ocupação</Text>
            <TextInput
              style={styles.input}
              placeholder="Sua ocupação"
              placeholderTextColor={Colors.gray}
              value={formData.ocupacao}
              onChangeText={v => handleChange('ocupacao', v)}
            />
          </View>

          <View style={styles.sectionCard}>
            <Text style={styles.sectionTitle}>Contato</Text>

            <Text style={styles.label}>Email</Text>
            <TextInput
              style={getInputStyle('email')}
              placeholder="email@exemplo.com"
              placeholderTextColor={Colors.gray}
              value={formData.email}
              onChangeText={v => handleChange('email', v)}
              keyboardType="email-address"
              autoCapitalize="none"
            />
            {renderError('email')}

            <Text style={styles.label}>Telefone</Text>
            <TextInput
              style={getInputStyle('telefone')}
              placeholder="(00) 00000-0000"
              placeholderTextColor={Colors.gray}
              value={formData.telefone}
              onChangeText={v => handleChange('telefone', v)}
              keyboardType="phone-pad"
              maxLength={15}
            />
            {renderError('telefone')}
          </View>

          <View style={styles.sectionCard}>
            <Text style={styles.sectionTitle}>Financeiro</Text>

            <Text style={styles.label}>Renda Declarada</Text>
            <TextInput
              style={styles.input}
              placeholder="R$ 0,00"
              placeholderTextColor={Colors.gray}
              value={formData.rendaDeclarada}
              onChangeText={v => handleChange('rendaDeclarada', v)}
              keyboardType="numeric"
            />
          </View>

          <View style={styles.sectionCard}>
            <Text style={styles.sectionTitle}>Endereço</Text>

            <Text style={styles.label}>CEP</Text>
            <TextInput
              style={getInputStyle('cep')}
              placeholder="00000-000"
              placeholderTextColor={Colors.gray}
              value={formData.cep}
              onChangeText={v => handleChange('cep', v)}
              keyboardType="numeric"
              maxLength={9}
            />
            {renderError('cep')}

            <View style={styles.row}>
              <View style={styles.flex3}>
                <Text style={styles.label}>Logradouro</Text>
                <TextInput
                  style={getInputStyle('logradouro')}
                  placeholder="Rua, Av..."
                  placeholderTextColor={Colors.gray}
                  value={formData.logradouro}
                  onChangeText={v => handleChange('logradouro', v)}
                />
                {renderError('logradouro')}
              </View>
              <View style={styles.flex1}>
                <Text style={styles.label}>Número</Text>
                <TextInput
                  style={getInputStyle('numero')}
                  placeholder="Nº"
                  placeholderTextColor={Colors.gray}
                  value={formData.numero}
                  onChangeText={v => handleChange('numero', v)}
                  keyboardType="numeric"
                />
                {renderError('numero')}
              </View>
            </View>

            <Text style={styles.label}>Bairro</Text>
            <TextInput
              style={getInputStyle('bairro')}
              placeholder="Bairro"
              placeholderTextColor={Colors.gray}
              value={formData.bairro}
              onChangeText={v => handleChange('bairro', v)}
            />
            {renderError('bairro')}

            <View style={styles.row}>
              <View style={styles.flex2}>
                <Text style={styles.label}>Cidade</Text>
                <TextInput
                  style={getInputStyle('cidade')}
                  placeholder="Cidade"
                  placeholderTextColor={Colors.gray}
                  value={formData.cidade}
                  onChangeText={v => handleChange('cidade', v)}
                />
                {renderError('cidade')}
              </View>
              <View style={styles.flex1}>
                <Text style={styles.label}>UF</Text>
                <TextInput
                  style={getInputStyle('uf')}
                  placeholder="UF"
                  placeholderTextColor={Colors.gray}
                  value={formData.uf}
                  onChangeText={v => handleChange('uf', v)}
                  maxLength={2}
                  autoCapitalize="characters"
                />
                {renderError('uf')}
              </View>
            </View>
          </View>

          <TouchableOpacity
            style={[styles.submitButton, loading && styles.submitButtonDisabled]}
            onPress={handleSubmit}
            disabled={loading}
          >
            <Icon name="send" size={20} color={Colors.white} style={styles.buttonIcon} />
            <Text style={styles.submitButtonText}>
              {loading ? 'Enviando...' : 'Enviar Solicitação'}
            </Text>
          </TouchableOpacity>
        </ScrollView>
      </KeyboardAvoidingView>
    </LinearGradient>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1 },
  flex: { flex: 1 },
  scrollContent: {
    padding: 20,
    paddingBottom: 40,
  },
  sectionCard: {
    backgroundColor: Colors.white,
    borderRadius: 16,
    padding: 20,
    marginBottom: 16,
    elevation: 4,
    shadowColor: Colors.black,
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
  },
  sectionTitle: {
    fontSize: 18,
    fontWeight: 'bold',
    color: Colors.primary,
    marginBottom: 16,
  },
  label: {
    fontSize: 13,
    color: Colors.textSecondary,
    marginBottom: 4,
    marginTop: 8,
    fontWeight: '500',
  },
  input: {
    borderWidth: 1,
    borderColor: Colors.border,
    borderRadius: 10,
    paddingHorizontal: 14,
    height: 48,
    fontSize: 15,
    color: Colors.text,
    backgroundColor: Colors.lightGray,
    justifyContent: 'center',
  },
  inputError: {
    borderColor: Colors.error,
    borderWidth: 1.5,
  },
  inputText: {
    fontSize: 15,
    color: Colors.text,
  },
  placeholderText: {
    fontSize: 15,
    color: Colors.gray,
  },
  errorText: {
    color: Colors.error,
    fontSize: 12,
    marginTop: 4,
    marginLeft: 2,
  },
  row: {
    flexDirection: 'row',
    gap: 12,
  },
  flex3: { flex: 3 },
  flex2: { flex: 2 },
  flex1: { flex: 1 },
  submitButton: {
    backgroundColor: Colors.success,
    borderRadius: 14,
    height: 54,
    flexDirection: 'row',
    justifyContent: 'center',
    alignItems: 'center',
    marginTop: 8,
    elevation: 4,
    shadowColor: Colors.success,
    shadowOffset: { width: 0, height: 3 },
    shadowOpacity: 0.3,
    shadowRadius: 5,
  },
  submitButtonDisabled: {
    backgroundColor: Colors.gray,
  },
  buttonIcon: {
    marginRight: 8,
  },
  submitButtonText: {
    color: Colors.white,
    fontSize: 17,
    fontWeight: 'bold',
  },
});

export default FormPF;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend/aurix-mobile && npx jest src/__tests__/FormPF.test.js --no-cache`
Expected: PASS (5 tests)

- [ ] **Step 5: Commit**

```bash
git add frontend/aurix-mobile/src/pages/onboarding/FormPF.js frontend/aurix-mobile/src/__tests__/FormPF.test.js
git commit -m "feat(mobile): add FormPF screen with validation, masks, API integration + tests"
```

---

### Task 5: StepEmpresa Screen

**Files:**
- Create: `src/pages/onboarding/StepEmpresa.js`
- Create: `src/__tests__/StepEmpresa.test.js`

**Interfaces:**
- Consumes: `navigation.navigate('StepSocios', { empresaData })`, `validarCNPJ()`, `formatarCNPJ()`, `validarEmail()`, `validarCEP()`
- Produces: empresaData object passed to StepSocios via navigation params

- [ ] **Step 1: Write the failing tests**

`src/__tests__/StepEmpresa.test.js`:
```js
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import StepEmpresa from '../pages/onboarding/StepEmpresa';

const mockNavigate = jest.fn();
const props = { navigation: { navigate: mockNavigate } };

describe('StepEmpresa', () => {
  beforeEach(() => {
    mockNavigate.mockClear();
  });

  it('renders CNPJ field and Próximo button', () => {
    const { getByText, getByPlaceholderText } = render(<StepEmpresa {...props} />);
    expect(getByPlaceholderText('00.000.000/0000-00')).toBeTruthy();
    expect(getByText('Próximo')).toBeTruthy();
  });

  it('shows CNPJ validation error for invalid CNPJ', () => {
    const { getByText, getByPlaceholderText } = render(<StepEmpresa {...props} />);
    const cnpjInput = getByPlaceholderText('00.000.000/0000-00');
    fireEvent.changeText(cnpjInput, '11.111.111/1111-11');
    fireEvent.press(getByText('Próximo'));
    expect(getByText('CNPJ inválido')).toBeTruthy();
  });

  it('formats CNPJ as user types', () => {
    const { getByPlaceholderText } = render(<StepEmpresa {...props} />);
    const cnpjInput = getByPlaceholderText('00.000.000/0000-00');
    fireEvent.changeText(cnpjInput, '11222333000181');
    expect(cnpjInput.props.value).toBe('11.222.333/0001-81');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend/aurix-mobile && npx jest src/__tests__/StepEmpresa.test.js --no-cache`
Expected: FAIL

- [ ] **Step 3: Write minimal implementation**

`src/pages/onboarding/StepEmpresa.js`:
```js
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  ScrollView,
  KeyboardAvoidingView,
  Platform,
} from 'react-native';
import LinearGradient from 'react-native-linear-gradient';
import Icon from 'react-native-vector-icons/MaterialIcons';
import { Colors } from '../../constants/Colors';
import {
  validarCNPJ,
  validarEmail,
  validarCEP,
  formatarCNPJ,
  formatarTelefone,
  formatarCEP,
} from '../../utils/validation';

const StepEmpresa = ({ navigation }) => {
  const [formData, setFormData] = useState({
    cnpj: '',
    razaoSocial: '',
    nomeFantasia: '',
    email: '',
    telefone: '',
    cep: '',
    logradouro: '',
    numero: '',
    bairro: '',
    cidade: '',
    uf: '',
  });
  const [cnpjValido, setCnpjValido] = useState(false);
  const [errors, setErrors] = useState({});

  const handleChange = (field, value) => {
    let formatted = value;
    if (field === 'cnpj') {
      formatted = formatarCNPJ(value);
      if (formatted.replace(/\D/g, '').length === 14) {
        setCnpjValido(validarCNPJ(formatted));
      } else {
        setCnpjValido(false);
      }
    } else if (field === 'telefone') formatted = formatarTelefone(value);
    else if (field === 'cep') formatted = formatarCEP(value);

    setFormData(prev => ({ ...prev, [field]: formatted }));
    if (errors[field]) setErrors(prev => ({ ...prev, [field]: '' }));
  };

  const validate = () => {
    const newErrors = {};
    if (!formData.cnpj) newErrors.cnpj = 'CNPJ é obrigatório';
    else if (!validarCNPJ(formData.cnpj)) newErrors.cnpj = 'CNPJ inválido';
    if (!formData.razaoSocial) newErrors.razaoSocial = 'Razão Social é obrigatória';
    if (!formData.nomeFantasia) newErrors.nomeFantasia = 'Nome Fantasia é obrigatório';
    if (!formData.email) newErrors.email = 'Email é obrigatório';
    else if (!validarEmail(formData.email)) newErrors.email = 'Email inválido';
    if (!formData.telefone) newErrors.telefone = 'Telefone é obrigatório';
    if (!formData.cep) newErrors.cep = 'CEP é obrigatório';
    else if (!validarCEP(formData.cep)) newErrors.cep = 'CEP inválido';
    if (!formData.logradouro) newErrors.logradouro = 'Logradouro é obrigatório';
    if (!formData.numero) newErrors.numero = 'Número é obrigatório';
    if (!formData.bairro) newErrors.bairro = 'Bairro é obrigatório';
    if (!formData.cidade) newErrors.cidade = 'Cidade é obrigatório';
    if (!formData.uf) newErrors.uf = 'UF é obrigatório';
    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleNext = () => {
    if (!validate()) return;
    navigation.navigate('StepSocios', { empresaData: formData });
  };

  const getInputStyle = (field) => [
    styles.input,
    errors[field] && styles.inputError,
  ];

  const renderError = (field) =>
    errors[field] ? <Text style={styles.errorText}>{errors[field]}</Text> : null;

  return (
    <LinearGradient colors={Colors.gradientPrimary} style={styles.container}>
      <KeyboardAvoidingView
        behavior={Platform.OS === 'ios' ? 'padding' : undefined}
        style={styles.flex}
      >
        <ScrollView contentContainerStyle={styles.scrollContent}>
          <View style={styles.stepIndicator}>
            <Text style={styles.stepText}>Passo 1 de 2</Text>
            <View style={styles.progressBar}>
              <View style={styles.progressFill} />
            </View>
          </View>

          <View style={styles.sectionCard}>
            <Text style={styles.sectionTitle}>Dados da Empresa</Text>

            <Text style={styles.label}>CNPJ</Text>
            <View style={styles.cnpjRow}>
              <TextInput
                style={[getInputStyle('cnpj'), styles.cnpjInput]}
                placeholder="00.000.000/0000-00"
                placeholderTextColor={Colors.gray}
                value={formData.cnpj}
                onChangeText={v => handleChange('cnpj', v)}
                keyboardType="numeric"
                maxLength={18}
              />
              {cnpjValido && (
                <Icon name="check-circle" size={24} color={Colors.success} style={styles.checkIcon} />
              )}
            </View>
            {renderError('cnpj')}

            <Text style={styles.label}>Razão Social</Text>
            <TextInput
              style={getInputStyle('razaoSocial')}
              placeholder="Razão Social"
              placeholderTextColor={Colors.gray}
              value={formData.razaoSocial}
              onChangeText={v => handleChange('razaoSocial', v)}
            />
            {renderError('razaoSocial')}

            <Text style={styles.label}>Nome Fantasia</Text>
            <TextInput
              style={getInputStyle('nomeFantasia')}
              placeholder="Nome Fantasia"
              placeholderTextColor={Colors.gray}
              value={formData.nomeFantasia}
              onChangeText={v => handleChange('nomeFantasia', v)}
            />
            {renderError('nomeFantasia')}

            <Text style={styles.label}>Email</Text>
            <TextInput
              style={getInputStyle('email')}
              placeholder="email@empresa.com"
              placeholderTextColor={Colors.gray}
              value={formData.email}
              onChangeText={v => handleChange('email', v)}
              keyboardType="email-address"
              autoCapitalize="none"
            />
            {renderError('email')}

            <Text style={styles.label}>Telefone</Text>
            <TextInput
              style={getInputStyle('telefone')}
              placeholder="(00) 00000-0000"
              placeholderTextColor={Colors.gray}
              value={formData.telefone}
              onChangeText={v => handleChange('telefone', v)}
              keyboardType="phone-pad"
              maxLength={15}
            />
            {renderError('telefone')}
          </View>

          <View style={styles.sectionCard}>
            <Text style={styles.sectionTitle}>Endereço</Text>

            <Text style={styles.label}>CEP</Text>
            <TextInput
              style={getInputStyle('cep')}
              placeholder="00000-000"
              placeholderTextColor={Colors.gray}
              value={formData.cep}
              onChangeText={v => handleChange('cep', v)}
              keyboardType="numeric"
              maxLength={9}
            />
            {renderError('cep')}

            <View style={styles.row}>
              <View style={styles.flex3}>
                <Text style={styles.label}>Logradouro</Text>
                <TextInput
                  style={getInputStyle('logradouro')}
                  placeholder="Rua, Av..."
                  placeholderTextColor={Colors.gray}
                  value={formData.logradouro}
                  onChangeText={v => handleChange('logradouro', v)}
                />
                {renderError('logradouro')}
              </View>
              <View style={styles.flex1}>
                <Text style={styles.label}>Número</Text>
                <TextInput
                  style={getInputStyle('numero')}
                  placeholder="Nº"
                  placeholderTextColor={Colors.gray}
                  value={formData.numero}
                  onChangeText={v => handleChange('numero', v)}
                  keyboardType="numeric"
                />
                {renderError('numero')}
              </View>
            </View>

            <Text style={styles.label}>Bairro</Text>
            <TextInput
              style={getInputStyle('bairro')}
              placeholder="Bairro"
              placeholderTextColor={Colors.gray}
              value={formData.bairro}
              onChangeText={v => handleChange('bairro', v)}
            />
            {renderError('bairro')}

            <View style={styles.row}>
              <View style={styles.flex2}>
                <Text style={styles.label}>Cidade</Text>
                <TextInput
                  style={getInputStyle('cidade')}
                  placeholder="Cidade"
                  placeholderTextColor={Colors.gray}
                  value={formData.cidade}
                  onChangeText={v => handleChange('cidade', v)}
                />
                {renderError('cidade')}
              </View>
              <View style={styles.flex1}>
                <Text style={styles.label}>UF</Text>
                <TextInput
                  style={getInputStyle('uf')}
                  placeholder="UF"
                  placeholderTextColor={Colors.gray}
                  value={formData.uf}
                  onChangeText={v => handleChange('uf', v)}
                  maxLength={2}
                  autoCapitalize="characters"
                />
                {renderError('uf')}
              </View>
            </View>
          </View>

          <TouchableOpacity
            style={styles.nextButton}
            onPress={handleNext}
          >
            <Text style={styles.nextButtonText}>Próximo</Text>
            <Icon name="arrow-forward" size={20} color={Colors.white} />
          </TouchableOpacity>
        </ScrollView>
      </KeyboardAvoidingView>
    </LinearGradient>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1 },
  flex: { flex: 1 },
  scrollContent: { padding: 20, paddingBottom: 40 },
  stepIndicator: { alignItems: 'center', marginBottom: 16 },
  stepText: { fontSize: 13, color: Colors.white, marginBottom: 8, fontWeight: '500' },
  progressBar: {
    width: '100%',
    height: 6,
    backgroundColor: 'rgba(255,255,255,0.3)',
    borderRadius: 3,
    overflow: 'hidden',
  },
  progressFill: {
    width: '50%',
    height: '100%',
    backgroundColor: Colors.white,
    borderRadius: 3,
  },
  sectionCard: {
    backgroundColor: Colors.white,
    borderRadius: 16,
    padding: 20,
    marginBottom: 16,
    elevation: 4,
    shadowColor: Colors.black,
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
  },
  sectionTitle: {
    fontSize: 18,
    fontWeight: 'bold',
    color: Colors.primary,
    marginBottom: 16,
  },
  label: {
    fontSize: 13,
    color: Colors.textSecondary,
    marginBottom: 4,
    marginTop: 8,
    fontWeight: '500',
  },
  input: {
    borderWidth: 1,
    borderColor: Colors.border,
    borderRadius: 10,
    paddingHorizontal: 14,
    height: 48,
    fontSize: 15,
    color: Colors.text,
    backgroundColor: Colors.lightGray,
    justifyContent: 'center',
  },
  inputError: { borderColor: Colors.error, borderWidth: 1.5 },
  inputText: { fontSize: 15, color: Colors.text },
  errorText: { color: Colors.error, fontSize: 12, marginTop: 4, marginLeft: 2 },
  cnpjRow: { flexDirection: 'row', alignItems: 'center' },
  cnpjInput: { flex: 1 },
  checkIcon: { marginLeft: 8 },
  row: { flexDirection: 'row', gap: 12 },
  flex3: { flex: 3 },
  flex2: { flex: 2 },
  flex1: { flex: 1 },
  nextButton: {
    backgroundColor: Colors.primary,
    borderRadius: 14,
    height: 54,
    flexDirection: 'row',
    justifyContent: 'center',
    alignItems: 'center',
    marginTop: 8,
    elevation: 4,
    shadowColor: Colors.primary,
    shadowOffset: { width: 0, height: 3 },
    shadowOpacity: 0.3,
    shadowRadius: 5,
  },
  nextButtonText: {
    color: Colors.white,
    fontSize: 17,
    fontWeight: 'bold',
    marginRight: 8,
  },
});

export default StepEmpresa;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend/aurix-mobile && npx jest src/__tests__/StepEmpresa.test.js --no-cache`
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```bash
git add frontend/aurix-mobile/src/pages/onboarding/StepEmpresa.js frontend/aurix-mobile/src/__tests__/StepEmpresa.test.js
git commit -m "feat(mobile): add StepEmpresa PJ screen with CNPJ validation + tests"
```

---

### Task 6: StepSocios Screen

**Files:**
- Create: `src/pages/onboarding/StepSocios.js`
- Create: `src/__tests__/StepSocios.test.js`

**Interfaces:**
- Consumes: `route.params.empresaData`, `navigation.navigate('SuccessScreen', { protocolo })`, `validarCPF()`, `onboardingService.criarSolicitacaoPJ(dados)`
- Produces: POST to backend with merged empresaData + socios, then navigate to SuccessScreen

- [ ] **Step 1: Write the failing tests**

`src/__tests__/StepSocios.test.js`:
```js
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import StepSocios from '../pages/onboarding/StepSocios';

const mockNavigate = jest.fn();
const mockRoute = { params: { empresaData: { cnpj: '11.222.333/0001-81', razaoSocial: 'Empresa X' } } };
const props = { navigation: { navigate: mockNavigate }, route: mockRoute };

describe('StepSocios', () => {
  beforeEach(() => {
    mockNavigate.mockClear();
  });

  it('renders step indicator and add button', () => {
    const { getByText } = render(<StepSocios {...props} />);
    expect(getByText('Passo 2 de 2')).toBeTruthy();
    expect(getByText('Adicionar Sócio')).toBeTruthy();
  });

  it('opens modal when Adicionar Sócio is pressed', () => {
    const { getByText, getByPlaceholderText } = render(<StepSocios {...props} />);
    fireEvent.press(getByText('Adicionar Sócio'));
    expect(getByPlaceholderText('000.000.000-00')).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend/aurix-mobile && npx jest src/__tests__/StepSocios.test.js --no-cache`
Expected: FAIL

- [ ] **Step 3: Write minimal implementation**

`src/pages/onboarding/StepSocios.js`:
```js
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  ScrollView,
  Modal,
  Alert,
  KeyboardAvoidingView,
  Platform,
} from 'react-native';
import LinearGradient from 'react-native-linear-gradient';
import Icon from 'react-native-vector-icons/MaterialIcons';
import { Colors } from '../../constants/Colors';
import { validarCPF, formatarCPF } from '../../utils/validation';
import { onboardingService } from '../../services/onboardingService';

const QUALIFICACOES = ['Sócio-Administrador', 'Sócio', 'Diretor', 'Outro'];

const StepSocios = ({ navigation, route }) => {
  const { empresaData } = route.params;
  const [socios, setSocios] = useState([]);
  const [modalVisible, setModalVisible] = useState(false);
  const [socioForm, setSocioForm] = useState({ cpf: '', nome: '', email: '', qualificacao: '' });
  const [socioErrors, setSocioErrors] = useState({});
  const [showQualPicker, setShowQualPicker] = useState(false);
  const [loading, setLoading] = useState(false);

  const resetSocioForm = () => {
    setSocioForm({ cpf: '', nome: '', email: '', qualificacao: '' });
    setSocioErrors({});
  };

  const handleSocioChange = (field, value) => {
    const formatted = field === 'cpf' ? formatarCPF(value) : value;
    setSocioForm(prev => ({ ...prev, [field]: formatted }));
    if (socioErrors[field]) setSocioErrors(prev => ({ ...prev, [field]: '' }));
  };

  const validateSocio = () => {
    const errors = {};
    if (!socioForm.cpf) errors.cpf = 'CPF é obrigatório';
    else if (!validarCPF(socioForm.cpf)) errors.cpf = 'CPF inválido';
    if (!socioForm.nome) errors.nome = 'Nome é obrigatório';
    if (!socioForm.qualificacao) errors.qualificacao = 'Selecione a qualificação';
    setSocioErrors(errors);
    return Object.keys(errors).length === 0;
  };

  const addSocio = () => {
    if (!validateSocio()) return;
    setSocios(prev => [...prev, { ...socioForm }]);
    setModalVisible(false);
    resetSocioForm();
  };

  const removeSocio = (index) => {
    setSocios(prev => prev.filter((_, i) => i !== index));
  };

  const handleSubmit = async () => {
    if (socios.length === 0) {
      Alert.alert('Atenção', 'Adicione pelo menos um sócio.');
      return;
    }
    setLoading(true);
    try {
      const dados = {
        cnpj: empresaData.cnpj.replace(/\D/g, ''),
        razaoSocial: empresaData.razaoSocial,
        nomeFantasia: empresaData.nomeFantasia,
        email: empresaData.email,
        telefone: empresaData.telefone.replace(/\D/g, ''),
        endereco: {
          cep: empresaData.cep.replace(/\D/g, ''),
          logradouro: empresaData.logradouro,
          numero: empresaData.numero,
          bairro: empresaData.bairro,
          cidade: empresaData.cidade,
          uf: empresaData.uf,
        },
        socios: socios.map(s => ({
          cpf: s.cpf.replace(/\D/g, ''),
          nome: s.nome,
          email: s.email,
          qualificacao: s.qualificacao,
        })),
      };
      const response = await onboardingService.criarSolicitacaoPJ(dados);
      navigation.navigate('SuccessScreen', { protocolo: response.protocolo });
    } catch (error) {
      Alert.alert('Erro', 'Erro ao enviar solicitação. Tente novamente.');
    } finally {
      setLoading(false);
    }
  };

  return (
    <LinearGradient colors={Colors.gradientPrimary} style={styles.container}>
      <KeyboardAvoidingView
        behavior={Platform.OS === 'ios' ? 'padding' : undefined}
        style={styles.flex}
      >
        <ScrollView contentContainerStyle={styles.scrollContent}>
          <View style={styles.stepIndicator}>
            <Text style={styles.stepText}>Passo 2 de 2</Text>
            <View style={styles.progressBar}>
              <View style={[styles.progressFill, { width: '100%' }]} />
            </View>
          </View>

          <View style={styles.sectionCard}>
            <View style={styles.sectionHeader}>
              <Text style={styles.sectionTitle}>Sócios</Text>
              <TouchableOpacity
                style={styles.addButton}
                onPress={() => {
                  resetSocioForm();
                  setModalVisible(true);
                }}
              >
                <Icon name="person-add" size={20} color={Colors.white} />
                <Text style={styles.addButtonText}>Adicionar Sócio</Text>
              </TouchableOpacity>
            </View>

            {socios.length === 0 ? (
              <Text style={styles.emptyText}>Nenhum sócio adicionado ainda.</Text>
            ) : (
              socios.map((socio, index) => (
                <View key={index} style={styles.socioCard}>
                  <View style={styles.socioInfo}>
                    <Icon name="person" size={24} color={Colors.primary} />
                    <View style={styles.socioDetails}>
                      <Text style={styles.socioName}>{socio.nome}</Text>
                      <Text style={styles.socioDetail}>{socio.cpf}</Text>
                      <Text style={styles.socioDetail}>{socio.qualificacao}</Text>
                    </View>
                  </View>
                  <TouchableOpacity onPress={() => removeSocio(index)}>
                    <Icon name="delete" size={22} color={Colors.error} />
                  </TouchableOpacity>
                </View>
              ))
            )}
          </View>

          <TouchableOpacity
            style={[styles.submitButton, loading && styles.submitButtonDisabled]}
            onPress={handleSubmit}
            disabled={loading}
          >
            <Icon name="send" size={20} color={Colors.white} style={styles.buttonIcon} />
            <Text style={styles.submitButtonText}>
              {loading ? 'Enviando...' : 'Revisar e Enviar'}
            </Text>
          </TouchableOpacity>
        </ScrollView>

        <Modal visible={modalVisible} animationType="slide" transparent>
          <View style={styles.modalOverlay}>
            <View style={styles.modalContent}>
              <View style={styles.modalHeader}>
                <Text style={styles.modalTitle}>Adicionar Sócio</Text>
                <TouchableOpacity onPress={() => setModalVisible(false)}>
                  <Icon name="close" size={24} color={Colors.text} />
                </TouchableOpacity>
              </View>

              <Text style={styles.label}>CPF</Text>
              <TextInput
                style={[styles.input, socioErrors.cpf && styles.inputError]}
                placeholder="000.000.000-00"
                placeholderTextColor={Colors.gray}
                value={socioForm.cpf}
                onChangeText={v => handleSocioChange('cpf', v)}
                keyboardType="numeric"
                maxLength={14}
              />
              {socioErrors.cpf && <Text style={styles.errorText}>{socioErrors.cpf}</Text>}

              <Text style={styles.label}>Nome</Text>
              <TextInput
                style={[styles.input, socioErrors.nome && styles.inputError]}
                placeholder="Nome completo"
                placeholderTextColor={Colors.gray}
                value={socioForm.nome}
                onChangeText={v => handleSocioChange('nome', v)}
              />
              {socioErrors.nome && <Text style={styles.errorText}>{socioErrors.nome}</Text>}

              <Text style={styles.label}>Email</Text>
              <TextInput
                style={styles.input}
                placeholder="email@exemplo.com"
                placeholderTextColor={Colors.gray}
                value={socioForm.email}
                onChangeText={v => handleSocioChange('email', v)}
                keyboardType="email-address"
                autoCapitalize="none"
              />

              <Text style={styles.label}>Qualificação</Text>
              <TouchableOpacity
                style={styles.input}
                onPress={() => setShowQualPicker(!showQualPicker)}
              >
                <Text style={socioForm.qualificacao ? styles.inputText : styles.placeholderText}>
                  {socioForm.qualificacao || 'Selecione...'}
                </Text>
              </TouchableOpacity>
              {socioErrors.qualificacao && <Text style={styles.errorText}>{socioErrors.qualificacao}</Text>}
              {showQualPicker && (
                <View style={styles.pickerContainer}>
                  {QUALIFICACOES.map(q => (
                    <TouchableOpacity
                      key={q}
                      style={[styles.pickerItem, socioForm.qualificacao === q && styles.pickerItemSelected]}
                      onPress={() => {
                        handleSocioChange('qualificacao', q);
                        setShowQualPicker(false);
                      }}
                    >
                      <Text style={[styles.pickerItemText, socioForm.qualificacao === q && styles.pickerItemTextSelected]}>
                        {q}
                      </Text>
                    </TouchableOpacity>
                  ))}
                </View>
              )}

              <TouchableOpacity style={styles.modalAddButton} onPress={addSocio}>
                <Text style={styles.modalAddButtonText}>Adicionar</Text>
              </TouchableOpacity>
            </View>
          </View>
        </Modal>
      </KeyboardAvoidingView>
    </LinearGradient>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1 },
  flex: { flex: 1 },
  scrollContent: { padding: 20, paddingBottom: 40 },
  stepIndicator: { alignItems: 'center', marginBottom: 16 },
  stepText: { fontSize: 13, color: Colors.white, marginBottom: 8, fontWeight: '500' },
  progressBar: {
    width: '100%', height: 6, backgroundColor: 'rgba(255,255,255,0.3)',
    borderRadius: 3, overflow: 'hidden',
  },
  progressFill: { height: '100%', backgroundColor: Colors.white, borderRadius: 3 },
  sectionCard: {
    backgroundColor: Colors.white, borderRadius: 16, padding: 20,
    marginBottom: 16, elevation: 4, shadowColor: Colors.black,
    shadowOffset: { width: 0, height: 2 }, shadowOpacity: 0.1, shadowRadius: 4,
  },
  sectionHeader: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', marginBottom: 16 },
  sectionTitle: { fontSize: 18, fontWeight: 'bold', color: Colors.primary },
  addButton: {
    flexDirection: 'row', alignItems: 'center', backgroundColor: Colors.primary,
    borderRadius: 10, paddingHorizontal: 14, paddingVertical: 8,
  },
  addButtonText: { color: Colors.white, fontSize: 13, fontWeight: 'bold', marginLeft: 6 },
  emptyText: { textAlign: 'center', color: Colors.textSecondary, fontSize: 14, paddingVertical: 20 },
  socioCard: {
    flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between',
    backgroundColor: Colors.lightGray, borderRadius: 12, padding: 14, marginBottom: 10,
  },
  socioInfo: { flexDirection: 'row', alignItems: 'center', flex: 1 },
  socioDetails: { marginLeft: 12, flex: 1 },
  socioName: { fontSize: 15, fontWeight: '600', color: Colors.text },
  socioDetail: { fontSize: 13, color: Colors.textSecondary, marginTop: 2 },
  submitButton: {
    backgroundColor: Colors.success, borderRadius: 14, height: 54,
    flexDirection: 'row', justifyContent: 'center', alignItems: 'center',
    marginTop: 8, elevation: 4, shadowColor: Colors.success,
    shadowOffset: { width: 0, height: 3 }, shadowOpacity: 0.3, shadowRadius: 5,
  },
  submitButtonDisabled: { backgroundColor: Colors.gray },
  buttonIcon: { marginRight: 8 },
  submitButtonText: { color: Colors.white, fontSize: 17, fontWeight: 'bold' },
  modalOverlay: {
    flex: 1, backgroundColor: Colors.overlay,
    justifyContent: 'flex-end',
  },
  modalContent: {
    backgroundColor: Colors.white, borderTopLeftRadius: 24, borderTopRightRadius: 24,
    padding: 24, maxHeight: '80%',
  },
  modalHeader: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', marginBottom: 20 },
  modalTitle: { fontSize: 20, fontWeight: 'bold', color: Colors.text },
  label: {
    fontSize: 13, color: Colors.textSecondary, marginBottom: 4, marginTop: 12, fontWeight: '500',
  },
  input: {
    borderWidth: 1, borderColor: Colors.border, borderRadius: 10,
    paddingHorizontal: 14, height: 48, fontSize: 15, color: Colors.text,
    backgroundColor: Colors.lightGray, justifyContent: 'center',
  },
  inputError: { borderColor: Colors.error, borderWidth: 1.5 },
  inputText: { fontSize: 15, color: Colors.text },
  placeholderText: { fontSize: 15, color: Colors.gray },
  errorText: { color: Colors.error, fontSize: 12, marginTop: 4, marginLeft: 2 },
  pickerContainer: {
    backgroundColor: Colors.white, borderWidth: 1, borderColor: Colors.border,
    borderRadius: 10, marginTop: 4,
  },
  pickerItem: { padding: 14, borderBottomWidth: 1, borderBottomColor: Colors.borderLight },
  pickerItemSelected: { backgroundColor: Colors.lightGray },
  pickerItemText: { fontSize: 15, color: Colors.text },
  pickerItemTextSelected: { fontWeight: 'bold', color: Colors.primary },
  modalAddButton: {
    backgroundColor: Colors.primary, borderRadius: 12, height: 50,
    justifyContent: 'center', alignItems: 'center', marginTop: 20,
  },
  modalAddButtonText: { color: Colors.white, fontSize: 16, fontWeight: 'bold' },
});

export default StepSocios;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend/aurix-mobile && npx jest src/__tests__/StepSocios.test.js --no-cache`
Expected: PASS (2 tests)

- [ ] **Step 5: Commit**

```bash
git add frontend/aurix-mobile/src/pages/onboarding/StepSocios.js frontend/aurix-mobile/src/__tests__/StepSocios.test.js
git commit -m "feat(mobile): add StepSocios PJ screen with partner management + tests"
```

---

### Task 7: SuccessScreen

**Files:**
- Create: `src/pages/onboarding/SuccessScreen.js`

**Interfaces:**
- Consumes: `route.params.protocolo`, `navigation.navigate('Login')`
- Produces: success screen with protocol display and navigation buttons

- [ ] **Step 1: Write code (manual verification)**

`src/pages/onboarding/SuccessScreen.js`:
```js
import React from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
} from 'react-native';
import LinearGradient from 'react-native-linear-gradient';
import Icon from 'react-native-vector-icons/MaterialIcons';
import { Colors } from '../../constants/Colors';

const SuccessScreen = ({ navigation, route }) => {
  const { protocolo } = route.params || {};

  return (
    <LinearGradient colors={Colors.gradientSuccess} style={styles.container}>
      <View style={styles.content}>
        <View style={styles.iconCircle}>
          <Icon name="check" size={56} color={Colors.white} />
        </View>

        <Text style={styles.title}>Solicitação enviada!</Text>
        <Text style={styles.subtitle}>
          Sua solicitação foi recebida com sucesso e será analisada em breve.
        </Text>

        {protocolo && (
          <View style={styles.protocolBox}>
            <Text style={styles.protocolLabel}>Protocolo</Text>
            <Text style={styles.protocolNumber}>{protocolo}</Text>
          </View>
        )}

        <TouchableOpacity
          style={styles.primaryButton}
          onPress={() => navigation.navigate('Login')}
        >
          <Text style={styles.primaryButtonText}>Voltar ao Início</Text>
        </TouchableOpacity>
      </View>
    </LinearGradient>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1 },
  content: {
    flex: 1, justifyContent: 'center', alignItems: 'center',
    paddingHorizontal: 40,
  },
  iconCircle: {
    width: 100, height: 100, borderRadius: 50,
    backgroundColor: 'rgba(255,255,255,0.3)',
    justifyContent: 'center', alignItems: 'center',
    marginBottom: 24,
  },
  title: {
    fontSize: 28, fontWeight: 'bold', color: Colors.white, marginBottom: 12,
    textAlign: 'center',
  },
  subtitle: {
    fontSize: 15, color: Colors.white, textAlign: 'center',
    opacity: 0.9, lineHeight: 22, marginBottom: 32,
  },
  protocolBox: {
    backgroundColor: 'rgba(255,255,255,0.2)',
    borderRadius: 16, padding: 20, alignItems: 'center',
    marginBottom: 40, width: '100%',
  },
  protocolLabel: {
    fontSize: 13, color: Colors.white, opacity: 0.8, marginBottom: 6,
    textTransform: 'uppercase', letterSpacing: 1,
  },
  protocolNumber: {
    fontSize: 22, fontWeight: 'bold', color: Colors.white,
    letterSpacing: 2,
  },
  primaryButton: {
    backgroundColor: Colors.white, borderRadius: 14,
    height: 54, justifyContent: 'center', alignItems: 'center',
    width: '100%', elevation: 4, shadowColor: Colors.black,
    shadowOffset: { width: 0, height: 3 }, shadowOpacity: 0.2, shadowRadius: 5,
  },
  primaryButtonText: {
    color: Colors.success, fontSize: 17, fontWeight: 'bold',
  },
});

export default SuccessScreen;
```

- [ ] **Step 2: Verify all tests still pass**

Run: `cd frontend/aurix-mobile && npx jest --no-cache`
Expected: PASS (14 tests total)

- [ ] **Step 3: Commit**

```bash
git add frontend/aurix-mobile/src/pages/onboarding/SuccessScreen.js
git commit -m "feat(mobile): add SuccessScreen with protocol display"
```

---

### Task 8: OnboardingScreen Orchestrator + AuthNavigator Routes

**Files:**
- Create: `src/pages/OnboardingScreen.js`
- Modify: `src/navigation/AuthNavigator.js`

**Interfaces:**
- Consumes: `biometricsEnabled` prop from AuthNavigator, all onboarding screen components
- Produces: renders TipoSelector as the initial onboarding content; AuthNavigator gets 4 new routes

- [ ] **Step 1: Create OnboardingScreen orchestrator**

`src/pages/OnboardingScreen.js`:
```js
import React from 'react';
import TipoSelector from './onboarding/TipoSelector';

const OnboardingScreen = (props) => {
  return <TipoSelector {...props} />;
};

export default OnboardingScreen;
```

- [ ] **Step 2: Update AuthNavigator with onboarding routes**

Edit `src/navigation/AuthNavigator.js`:
```js
import React from 'react';
import { createStackNavigator } from '@react-navigation/stack';

// Screens
import LoginScreen from '../pages/LoginScreen';
import BiometricsLoginScreen from '../pages/BiometricsLoginScreen';
import ForgotPasswordScreen from '../pages/ForgotPasswordScreen';
import RegisterScreen from '../pages/RegisterScreen';
import OnboardingScreen from '../pages/OnboardingScreen';
import FormPF from '../pages/onboarding/FormPF';
import StepEmpresa from '../pages/onboarding/StepEmpresa';
import StepSocios from '../pages/onboarding/StepSocios';
import SuccessScreen from '../pages/onboarding/SuccessScreen';

// Constants
import { Colors } from '../constants/Colors';

const Stack = createStackNavigator();

const AuthNavigator = ({ onLogin, biometricsEnabled }) => {
  return (
    <Stack.Navigator
      initialRouteName="Onboarding"
      screenOptions={{
        headerStyle: {
          backgroundColor: Colors.primary,
          elevation: 0,
          shadowOpacity: 0,
        },
        headerTintColor: Colors.white,
        headerTitleStyle: {
          fontWeight: 'bold',
          fontSize: 18,
        },
        headerBackTitleVisible: false,
        cardStyle: {
          backgroundColor: Colors.background,
        },
      }}
    >
      <Stack.Screen 
        name="Onboarding" 
        options={{ headerShown: false }}
      >
        {props => (
          <OnboardingScreen 
            {...props} 
            biometricsEnabled={biometricsEnabled}
          />
        )}
      </Stack.Screen>
      
      <Stack.Screen 
        name="Login" 
        options={{ 
          title: 'Entrar',
          headerShown: false 
        }}
      >
        {props => (
          <LoginScreen 
            {...props} 
            onLogin={onLogin}
            biometricsEnabled={biometricsEnabled}
          />
        )}
      </Stack.Screen>
      
      <Stack.Screen 
        name="BiometricsLogin" 
        options={{ 
          title: 'Login Biométrico',
          headerShown: false 
        }}
      >
        {props => (
          <BiometricsLoginScreen 
            {...props} 
            onLogin={onLogin}
          />
        )}
      </Stack.Screen>
      
      <Stack.Screen 
        name="ForgotPassword" 
        component={ForgotPasswordScreen}
        options={{ 
          title: 'Recuperar Senha',
          headerStyle: { backgroundColor: Colors.primary },
          headerTintColor: Colors.white,
        }}
      />
      
      <Stack.Screen 
        name="Register" 
        component={RegisterScreen}
        options={{ 
          title: 'Criar Conta',
          headerStyle: { backgroundColor: Colors.primary },
          headerTintColor: Colors.white,
        }}
      />

      <Stack.Screen
        name="FormPF"
        component={FormPF}
        options={{
          title: 'Abertura de Conta PF',
          headerStyle: { backgroundColor: Colors.primary },
          headerTintColor: Colors.white,
        }}
      />

      <Stack.Screen
        name="StepEmpresa"
        component={StepEmpresa}
        options={{
          title: 'Dados da Empresa',
          headerStyle: { backgroundColor: Colors.primary },
          headerTintColor: Colors.white,
        }}
      />

      <Stack.Screen
        name="StepSocios"
        component={StepSocios}
        options={{
          title: 'Sócios',
          headerStyle: { backgroundColor: Colors.primary },
          headerTintColor: Colors.white,
        }}
      />

      <Stack.Screen
        name="SuccessScreen"
        component={SuccessScreen}
        options={{ headerShown: false }}
      />
    </Stack.Navigator>
  );
};

export default AuthNavigator;
```

- [ ] **Step 3: Run all tests to verify everything passes**

Run: `cd frontend/aurix-mobile && npx jest --no-cache`
Expected: PASS (all tests)

- [ ] **Step 4: Commit**

```bash
git add frontend/aurix-mobile/src/pages/OnboardingScreen.js frontend/aurix-mobile/src/navigation/AuthNavigator.js
git commit -m "feat(mobile): wire onboarding screens into AuthNavigator"
```

---

## Self-Review Checklist

1. **Spec coverage:** All spec sections implemented — navigation (TipoSelector→FormPF/StepEmpresa→StepSocios→SuccessScreen), API service (3 methods), FormPF (4 sections, validation, masks), PJ wizard (2 steps), SuccessScreen (protocol display), deps (axios, datetimepicker), tests (3 test files, 14+ tests)
2. **Placeholder scan:** No TBD, TODO, or "implement later" in any task. Every code block is complete.
3. **Type consistency:** `navigation.navigate('FormPF')` (Task 3) → route name `'FormPF'` registered in AuthNavigator (Task 8). `route.params.protocolo` (Task 7) matches what FormPF passes. `route.params.empresaData` (Task 6) matches what StepEmpresa passes. All function names in validation.js match across all tasks.
