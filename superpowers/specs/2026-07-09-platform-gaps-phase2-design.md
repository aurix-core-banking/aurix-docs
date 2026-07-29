# Platform Gaps — Phase 2 Design

## Overview

Fill 5 remaining gaps across the platform after onboarding delivery:
1. OpenFinance stub endpoints (cartão de crédito + identificações pessoais)
2. Auth/password flows (forgot-password, reset-password, refresh token, lockout timed)
3. Kafka staging profile (Schema Registry + Kafka Connect + env sync)
4. Web frontend missing pages (Extrato, Transferencia, Pagamento, Recarga)
5. ML/BI/Chatbot stubs (profile-gating + interface extraction)

---

## Gap 1 — OpenFinance: Cartões de Crédito e Identificações Pessoais

**Controller:** `OpenFinanceController.java` — endpoints `GET /credit-cards-accounts` and `GET /customers/personal/identifications` currently return `"[]"` hardcoded.

### Data Service

Add two methods to `OpenFinanceDataService`, following the pattern of `listarContasPorToken()` and `listarTransacoesPorConta()`:

```java
public List<Map<String, Object>> listarCartoesCreditoPorToken(TokenOpenFinance token) {
    // 1. Validate consent permission includes CREDIT_CARDS_ACCOUNTS
    // 2. Get authorized account IDs from consent
    // 3. Call CoreApiClient.getCreditCards(authorizedAccounts)
    // 4. Transform to BACEN Open Finance format
}

public List<Map<String, Object>> listarIdentificacoesPessoaisPorToken(TokenOpenFinance token) {
    // 1. Validate consent permission includes IDENTIFICACAO
    // 2. Get customer data via CoreApiClient.getCliente(token.getUserId())
    // 3. Transform to BACEN format (civilName, socialName, birthDate, etc.)
}
```

### CoreApiClient additions

Two new methods:
- `getCreditCards(List<String> accountIds)` — calls existing cartoes API or returns filtered data
- `getPersonalIdentifications(Long userId)` — fetches customer data from shared entity

### BACEN response format

Credit cards:
```json
{
  "data": {
    "creditCards": [{
      "brand": "VISA",
      "companyName": "Aurix Pagamentos S.A.",
      "name": "Cartão Platinum",
      "productType": "PLATINUM",
      "creditCardNetwork": "VISA"
    }]
  }
}
```

Personal identifications:
```json
{
  "data": {
    "personalIdentifications": [{
      "updateDateTime": "2024-01-15T10:30:00Z",
      "personalId": "12345678901",
      "brand": "Aurix",
      "civilName": "João da Silva",
      "socialName": null,
      "birthDate": "1990-05-20",
      "documentType": "CPF"
    }]
  }
}
```

### Controller changes

Replace both TODO stubs with real calls to `openFinanceDataService`, maintaining existing token validation, rate limiting, and access logging.

### Tests

- `OpenFinanceDataServiceTest` — mock CoreApiClient, verify BACEN format transformation
- `OpenFinanceControllerTest` — HTTP test with valid token, verify correct response structure

---

## Gap 2 — Auth/Password: Forgot-Password, Reset-Password, Refresh Token, Lockout Timed

### New entities

**PasswordResetToken** (table `aurix.password_reset_tokens`):
- `id` (serial PK)
- `usuario_id` (FK → usuarios)
- `token` (UUID string, unique)
- `expira_em` (timestamp, 15min from creation)
- `utilizado` (boolean)
- `criado_em` (timestamp)

**RefreshToken** (table `aurix.refresh_tokens`):
- `id` (serial PK)
- `usuario_id` (FK → usuarios)
- `token` (UUID string, unique)
- `expira_em` (timestamp, 7 days from creation)
- `revogado` (boolean)
- `criado_em` (timestamp)

### New repositories

- `PasswordResetTokenRepository` — findByToken(String), deleteByUsuarioId(Long)
- `RefreshTokenRepository` — findByToken(String), findByUsuarioIdAndRevogadoFalse(Long)

### AuthController — new endpoints

| Endpoint | Method | Body | Response |
|----------|--------|------|---------|
| `/auth/forgot-password` | POST | `{ "email": "..." }` | 200 — token logado em dev, placeholder em prod |
| `/auth/reset-password` | POST | `{ "token": "...", "novaSenha": "..." }` | 204 |
| `/auth/refresh` | POST | `{ "refreshToken": "..." }` | `LoginResponseDTO` with new JWT + new refresh token |

### AuthService — new methods

**forgotPassword(email):**
1. Validate email exists and user is active
2. Invalidate any existing reset tokens for user
3. Generate UUID token, save with 15min expiry
4. In dev: log token; in prod: would send email (placeholder)

**resetPassword(token, novaSenha):**
1. Find token, validate not expired and not used
2. Find user, encode new password via BCrypt
3. Reset password expiry to +90 days
4. Mark token as used, reset login attempts, unlock account

**refreshToken(refreshToken):**
1. Find token, validate not expired and not revoked
2. Generate new JWT (24h)
3. Generate new refresh token (UUID, 7d)
4. Revoke old refresh token
5. Return `LoginResponseDTO` with new JWT + new refresh token

### Lockout timed

In `AuthService.login()`, modify block check:
```java
if (usuario.getContaBloqueada()) {
    if (usuario.getUltimoLogin().plusMinutes(5).isAfter(LocalDateTime.now())) {
        throw new IllegalStateException("Conta temporariamente bloqueada. Tente novamente em 5 minutos.");
    }
    usuario.setContaBloqueada(false); // auto-unlock after 5min
}
```

### Tests

- `AuthServiceTest` — forgotPassword, resetPassword, refreshToken, lockout auto-unlock, expired token, invalid token
- `AuthControllerTest` — endpoint integration tests

---

## Gap 3 — Kafka Staging: Schema Registry + Kafka Connect + Env Sync

### docker-compose.staging.yml additions

Insert `schema-registry` and `kafka-connect` services between kafka-3 and kafka-ui:

**schema-registry:**
- Image: `confluentinc/cp-schema-registry:7.4.0`
- Port: `${STAGING_SCHEMA_REGISTRY_PORT:-8081}:8081`
- Connects to all 3 Kafka brokers
- Network: `aurix-data-network-staging`

**kafka-connect:**
- Image: `confluentinc/cp-kafka-connect:7.4.0`
- Port: `${STAGING_KAFKA_CONNECT_PORT:-8083}:8083`
- Avro converter pointing to schema-registry
- Group ID: `connect-cluster-staging`
- Config/offset/status topics namespaced with `-staging` suffix
- Uses separate topics to avoid collision with dev cluster
- Network: `aurix-data-network-staging`

### .env.example update

Add staging-specific variables:
```env
STAGING_KAFKA_UI_PORT=8085
STAGING_SCHEMA_REGISTRY_PORT=8081
STAGING_KAFKA_CONNECT_PORT=8083
STAGING_KAFKA_REPLICATION_FACTOR=3
```

Keep existing `KAFKA_UI_PORT=8085` and `KAFKA_REPLICATION_FACTOR=1` for dev profile.

### Broker ID note

No change needed — broker IDs 1/2/3 are already correct for 3-broker staging.

---

## Gap 4 — Web Pages: Extrato, Transferencia, Pagamento, Recarga

### New page files

| File | Path |
|------|------|
| `Extrato.js` | `apps/frontend/aurix-web/src/pages/Extrato.js` |
| `Extrato.test.js` | `apps/frontend/aurix-web/src/pages/Extrato.test.js` |
| `Transferencia.js` | `apps/frontend/aurix-web/src/pages/Transferencia.js` |
| `Transferencia.test.js` | `apps/frontend/aurix-web/src/pages/Transferencia.test.js` |
| `Pagamento.js` | `apps/frontend/aurix-web/src/pages/Pagamento.js` |
| `Pagamento.test.js` | `apps/frontend/aurix-web/src/pages/Pagamento.test.js` |
| `Recarga.js` | `apps/frontend/aurix-web/src/pages/Recarga.js` |
| `Recarga.test.js` | `apps/frontend/aurix-web/src/pages/Recarga.test.js` |

### Extrato.js

- Dropdown to select account (from `apiService.getContas()`)
- Date range filter (start date, end date)
- Table: data, descrição, valor, saldo após
- Download PDF button
- Mock data on initial load, structure for API integration

### Transferencia.js

- Form: tipo (TED/DOC/PIX), conta destino (banco, agência, conta, dígito), valor, data agendamento
- PIX fields: chave PIX (CPF/CNPJ/email/telefone/aleatória)
- Confirmation dialog before submit
- Success/error feedback

### Pagamento.js

- Input: código de barras (boleto)
- Auto-complete: concessionária, tributo, etc.
- Display: vencimento, valor, multa/juros
- Payment confirmation dialog
- Success/error feedback

### Recarga.js

- Select: operadora (Vivo, Claro, TIM, Oi)
- Input: número de telefone (mascara)
- Select: valor (R$10, R$15, R$20, R$50, R$100, custom)
- Confirmation dialog
- Success/error feedback

### App.js — new routes

```jsx
import Extrato from './pages/Extrato';
import Transferencia from './pages/Transferencia';
import Pagamento from './pages/Pagamento';
import Recarga from './pages/Recarga';

<Route path="/extrato" element={<Extrato user={user} />} />
<Route path="/transferencia" element={<Transferencia user={user} />} />
<Route path="/pagamento" element={<Pagamento user={user} />} />
<Route path="/recarga" element={<Recarga user={user} />} />
```

### Sidebar.js

Add 4 new navigation items with icons:
- Extrato (Receipt icon)
- Transfrência (SwapHoriz icon)
- Pagamento (Payment icon)
- Recarga (PhoneAndroid icon)

### Coding conventions

Same as existing pages (`Contas.js`, `Cartoes.js`):
- Functional components with `export default function` syntax
- MUI 5 components (Box, Card, Grid, TextField, Button, Table, Dialog)
- `apiService` for API calls (with mock fallbacks)
- `numeral` for currency formatting
- `date-fns` with `ptBR` locale
- Portuguese labels throughout

### Tests

Each test file co-located (`Extrato.test.js`, etc.):
- Renders page title
- Displays mock data
- Form submit triggers expected action
- Error states handled gracefully

---

## Gap 5 — ML/BI/Chatbot Stubs: Interfaces + Profile-Gating

### Service interfaces (new)

Create in `apps/backend/aurix-analytics/src/main/java/com/aurix/platform/analytics/service/`:

| Interface | Methods | Endpoints served |
|-----------|---------|-------------------|
| `FraudService` | `avaliarFraude(Map) → Map` | `POST /api/analytics/ml/fraude/avaliar` |
| `CreditScoreService` | `obterScore(String) → Map` | `GET /api/analytics/ml/credito/score` |
| `ChatbotService` | `processarMensagem(String) → Map` | `POST /api/analytics/chatbot/mensagem` |
| `BiService` | `obterKpis() → Map`, `obterDashboard() → Map` | `GET /api/analytics/bi/kpis`, `GET /api/analytics/bi/dashboard` |

### Stub implementations (dev)

Each implements its interface, annotated `@Profile("!prod")`:

| Service | Implements |
|---------|-----------|
| `MlFraudServiceStub` | `FraudService` |
| `CreditScoreStubService` | `CreditScoreService` |
| `ChatbotStubService` | `ChatbotService` |
| `BiStubService` | `BiService` |

Same hardcoded/stub logic as current controllers, just extracted to services.

### Prod placeholder implementations

Each annotated `@Profile("prod")`, returns minimal safe values:

| Service | Returns |
|---------|---------|
| `MlFraudServiceProd` | `{ riscoFraude: 0.0, aprovado: true, modelo: "prod-placeholder" }` |
| `CreditScoreServiceProd` | `{ score: 500, faixa: "MEDIO_RISCO", modelo: "prod-placeholder" }` |
| `ChatbotServiceProd` | `{ resposta: "Funcionalidade em implementação", escalarParaHumano: true }` |
| `BiServiceProd` | Uses `MetricaRepository` + empty KPIs |

### Controllers (refactored)

Remove stub logic from `MlStubController`, `ChatbotStubController`, `BiStubController`. Rename to `MlController`, `ChatbotController`, `BiController`. Each injects their respective interface:

```java
@RestController
@RequestMapping("/api/analytics/ml")
public class MlController {
    private final FraudService fraudService;
    private final CreditScoreService creditScoreService;

    @PostMapping("/fraude/avaliar")
    public ResponseEntity<Map<String, Object>> avaliarFraude(@RequestBody Map<String, Object> transacao) {
        return ResponseEntity.ok(fraudService.avaliarFraude(transacao));
    }

    @GetMapping("/credito/score")
    public ResponseEntity<Map<String, Object>> scoreCredito(@RequestParam String clienteId) {
        return ResponseEntity.ok(creditScoreService.obterScore(clienteId));
    }
}
```

### Credit module alignment

`CreditBureauStub` already follows this pattern (`@Profile("!producao")`, implements `CreditBureauService`). New analytics stubs align with the same pattern, just using `@Profile("!prod")` (Spring profile name consistency).

### Tests

- Stub service unit tests (same logic as current controller tests)
- Prod placeholder unit tests
- Controller integration tests (interface injection resolves to dev stubs)
- `AnalyticsFlowIntegrationTest` updated to reference interfaces

---

## Implementation order

1. **Gap 2 (Auth/Password)** — backend-only, no dependencies on other gaps
2. **Gap 1 (OpenFinance)** — backend-only, extends existing service
3. **Gap 5 (ML/BI/Chatbot)** — backend-only, refactoring existing code
4. **Gap 3 (Kafka Staging)** — infra-only, no code dependencies
5. **Gap 4 (Web Pages)** — frontend-only, depends on understanding page patterns

Each gap can be planned and implemented independently.