# Camada 2: Mapa de Domínios — Plano de Implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Consolidar 43 módulos fonte em 10 domínios alvo, movendo entidades, serviços e controllers para os pacotes corretos e eliminando duplicatas.

**Architecture:** 10 fases independentes (F1–F10), cada uma movendo módulos para um dos 10 svc-* placeholder services criados na F0. Cada fase é um merge de 1–7 módulos fonte em 1 módulo alvo, compilável e testável independentemente.

**Tech Stack:** Java 25, Spring Boot 4.1, Maven, JPA/Hibernate.

## Global Constraints

- Nenhum módulo fonte (`aurix-*`) é deletado durante a migração — apenas copiado/movido
- Código copiado para `apps/backend/svc-{domain}/` mantém a funcionalidade original
- Entidades duplicadas são resolvidas conforme seção 2 do spec (versão canônica)
- Shared library `aurix-shared` permanece como dependência Maven
- Cada fase termina com `mvn clean compile -pl svc-{domain}` passando
- `mvn test -pl svc-{domain}` deve passar (ou estar esverdeado nos testes existentes)

---

## File Structure (por domínio)

```
apps/backend/svc-{domain}/
  pom.xml                                      ← já existe (F0), adicionar dependências dos módulos fonte
  Dockerfile                                   ← já existe (F0)
  src/main/java/com/aurix/platform/{domain}/
    entity/                                    ← entidades movidas dos módulos fonte
    service/                                   ← serviços movidos
    controller/                                ← controllers movidos
    repository/                                ← repositórios movidos
    config/                                    ← SecurityConfig, KafkaConfig (merge)
    client/                                    ← Feign clients (se houver)
    dto/                                       ← DTOs movidos
    event/                                     ← produtores/consumidores Kafka
    {Domain}Application.java                   ← já existe (F0)
  src/main/resources/
    application.yml                            ← já existe (F0), atualizar
  src/test/java/                               ← testes movidos
  src/test/resources/                          ← test config movida
```

---

### Task 1: F1 — customer-identity (pilot)

**Merge destes módulos:** `aurix-customer` + `aurix-kyc` + `aurix-onboarding` + `aurix-security`

**Para cada módulo fonte, mover:**

| Módulo fonte | Pacote alvo | Entidades | Serviços | Controllers |
|---|---|---|---|---|
| `aurix-customer` | `customer` | `Cliente`, `Contato`, `Endereco` | `ClienteService` | `ClienteController` |
| `aurix-kyc` | `customer.kyc` | `DocumentoKYC`, `ScoreKYC`, `SolicitacaoKYC` | `SolicitacaoKycService` | `SolicitacaoKycController` |
| `aurix-onboarding` | `customer.onboarding` | `SolicitacaoOnboarding`, `DocumentoOnboarding`, `SolicitacaoPF`, `SolicitacaoPJ`, `Empresa`, `Participante`, `Pep`, `HistoricoAprovacao` | `OnboardingPFService`, `OnboardingPJService` | `ControllerPF`, `ControllerPJ`, `IntegracaoTerceirosController` |
| `aurix-security` | `customer.security` | `MfaConfig`, `MfaToken`, `PasswordResetToken`, `RefreshToken` | `AuthService`, `JwtService`, `MfaService`, `PermissaoGranularService` | `AuthController`, `CriptografiaController`, `MfaController`, `PermissaoGranularController` |

- [ ] **Step 1: Criar subpacotes no svc-customer**

```bash
mkdir -p apps/backend/svc-customer/src/main/java/com/aurix/platform/customer/kyc
mkdir -p apps/backend/svc-customer/src/main/java/com/aurix/platform/customer/onboarding
mkdir -p apps/backend/svc-customer/src/main/java/com/aurix/platform/customer/security
mkdir -p apps/backend/svc-customer/src/main/java/com/aurix/platform/customer/repository
mkdir -p apps/backend/svc-customer/src/main/java/com/aurix/platform/customer/config
mkdir -p apps/backend/svc-customer/src/main/java/com/aurix/platform/customer/client
mkdir -p apps/backend/svc-customer/src/main/java/com/aurix/platform/customer/dto
mkdir -p apps/backend/svc-customer/src/main/java/com/aurix/platform/customer/event
mkdir -p apps/backend/svc-customer/src/main/java/com/aurix/platform/customer/job
mkdir -p apps/backend/svc-customer/src/test/java/com/aurix/platform/customer
mkdir -p apps/backend/svc-customer/src/test/resources
```

- [ ] **Step 2: Copiar entidades de aurix-customer**

Copiar cada arquivo `.java` de `apps/backend/aurix-customer/src/main/java/.../entity/` para `apps/backend/svc-customer/src/main/java/com/aurix/platform/customer/entity/`, atualizando o `package` declaration de `com.aurix.platform.customer.entity` para `com.aurix.platform.customer.entity`.

- [ ] **Step 3: Copiar entidades de aurix-kyc**

Copiar de `apps/backend/aurix-kyc/src/main/java/.../entity/` para `apps/backend/svc-customer/src/main/java/com/aurix/platform/customer/kyc/entity/`, atualizando package.

- [ ] **Step 4: Copiar entidades de aurix-onboarding**

Copiar de `apps/backend/aurix-onboarding/src/main/java/.../entity/` para `apps/backend/svc-customer/src/main/java/com/aurix/platform/customer/onboarding/entity/`, atualizando package.

- [ ] **Step 5: Copiar entidades de aurix-security**

Copiar de `apps/backend/aurix-security/src/main/java/.../entity/` para `apps/backend/svc-customer/src/main/java/com/aurix/platform/customer/security/entity/`, atualizando package.

- [ ] **Step 6: Copiar repositórios**

Para cada módulo fonte, copiar todos os `*Repository.java` para o subpacote `repository` correspondente no alvo. Atualizar `package` e `imports`.

- [ ] **Step 7: Copiar serviços**

Copiar todos os `*Service.java` para `service/` ou `kyc/service/` etc. Atualizar `package`, `imports`, e referências a entidades/repositórios (que mudaram de pacote).

**Atenção:** `OnboardingPJService` e `OnboardingPFService` têm dependências externas (BureauGateway, SerasaProvider, QuodProvider, UnicoProvider, ReceitaFederalStub). Mover esses clients para `client/`.

- [ ] **Step 8: Copiar controllers**

Copiar todos os `*Controller.java` para `controller/` ou subpacotes. Atualizar `package`, `imports`, `@RequestMapping` paths (se necessário).

- [ ] **Step 9: Copiar config**

Copiar `SecurityConfig.java` de cada módulo fonte. Fazer merge em um único `SecurityConfig` no alvo. Mover `KafkaConfig` e `JwtService` (que é serviço de security, vai para `customer.security.service`).

- [ ] **Step 10: Atualizar pom.xml do svc-customer**

Adicionar dependências necessárias para os 4 módulos fonte:
- `aurix-shared` (já deve estar)
- `spring-boot-starter-data-jpa`
- `spring-boot-starter-security`
- `spring-boot-starter-validation`
- `spring-kafka`
- Drivers de bureau externo (Serasa, Quod, Unico) — se disponíveis como stubs/mocks

- [ ] **Step 11: Configurar application.yml**

Atualizar `apps/backend/svc-customer/src/main/resources/application.yml`:
```yaml
server:
  port: 8204

spring:
  application:
    name: svc-customer
  jpa:
    hibernate:
      ddl-auto: none
    show-sql: false
  datasource:
    url: ${SPRING_DATASOURCE_URL:jdbc:postgresql://localhost:5432/aurix_db}
    username: ${SPRING_DATASOURCE_USERNAME:aurix_user}
    password: ${SPRING_DATASOURCE_PASSWORD:aurix_dev_password}
  kafka:
    bootstrap-servers: ${SPRING_KAFKA_BOOTSTRAP_SERVERS:localhost:9092}

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
  server:
    port: 9204

springdoc:
  api-docs:
    path: /api/customer/v3/api-docs
```

- [ ] **Step 12: Compilar e corrigir**

```bash
cd backend
mvn clean compile -pl svc-customer -q 2>&1 | grep "ERROR\|BUILD"
```

Corrigir erros de compilação — tipicamente: imports incorretos, classes faltando, dependências ausentes — até `BUILD SUCCESS`.

- [ ] **Step 13: Migrar testes**

Copiar testes de cada módulo fonte para `apps/backend/svc-customer/src/test/java/`, atualizando packages e imports. Executar:

```bash
mvn test -pl svc-customer -q
```

Corrigir até `BUILD SUCCESS`.

- [ ] **Step 14: Atualizar Traefik v2 paths**

Adicionar os prefixes ao router `customer` em `infra/traefik/dynamic.v2.yml`:
```yaml
customer:
  rule: "PathPrefix(`/api/customer`) || PathPrefix(`/api/kyc`) || PathPrefix(`/api/onboarding`) || PathPrefix(`/api/auth`) || PathPrefix(`/api/mfa`) || PathPrefix(`/api/security`)"
```

(Nota: o `/api/security` já está no router do spec da F0.)

- [ ] **Step 15: Commit**

```bash
git add apps/backend/svc-customer/ infra/traefik/dynamic.v2.yml
git commit -m "feat(domain): merge customer, kyc, onboarding, security into svc-customer (F1)"
```

---

### Task 2: F2 — banking-core

**Merge destes módulos:** `aurix-core` + `aurix-poupanca` + `aurix-salario` + `aurix-pricing` + `aurix-settlement`

**Subpacotes alvo:** `banking.core`, `banking.poupanca`, `banking.salario`, `banking.pricing`, `banking.settlement`

- [ ] **Step 1–15:** Seguir o mesmo padrão da Task 1 (criar pacotes, mover entidades, serviços, controllers, repositórios, config, pom.xml deps, compilar, testar, commit)

**Atenção a duplicatas:**
- `OutboxEvent` / `OutboxRelay` — presente em core e settlement → manter UMA versão em `shared` ou em `banking.core.event`
- `ConciliacaoBancaria` — presente em core e settlement → versão canônica vai para compliance (accounting). Manter only em banking-core como DTO/evento, não como entidade JPA
- `PacoteTarifas` / `Tarifa` — presente em core e pricing → manter em `banking.pricing`

**Controllers finais:** `ContaController`, `TransacaoController`, `ContaPoupancaController`, `ContaSalarioController`, `FolhaController`, `PortabilidadeController`, `SettlementController`, `PricingController`, `LiquidacaoController`

---

### Task 3: F3 — payments

**Merge destes módulos:** `aurix-pix` + `aurix-bacen` (parte SPI/STR)

- [ ] **Steps 1–15:** Seguir padrão da Task 1

**Boleto é movido de aurix-core (via aurix-core fica em banking-core? Não — boleto é pagamento).** Na Task 2, `Boleto` e `BoletoService` NÃO entram em banking-core. Entram aqui em payments.

**Atenção:** `aurix-bacen` tem duas partes — SPI/STR (entra em payments) e relatórios/Cosif (entra em compliance). Separar os dois.

---

### Task 4: F4 — credit

**Merge destes módulos:** `aurix-credit` + `aurix-consignado` + `aurix-financiamento`

- [ ] **Steps 1–15:** Seguir padrão da Task 1

**Garantia (`aurix-guarantee`):** Garantia é parte de financiamento. Entra em `credit.financiamento`. Não separar para products.

---

### Task 5: F5 — products

**Merge destes módulos:** `aurix-investimento` + `aurix-cambio` + `aurix-seguros` + `aurix-cartoes` + `aurix-treasury` + `aurix-guarantee` (se não entrou em credit)

- [ ] **Steps 1–15:** Seguir padrão da Task 1

**Atenção:** `aurix-treasury` tem `InvestimentoService` que duplica o de `aurix-investimento`. Manter a versão de investimento, descartar a de treasury.

---

### Task 6: F6 — fraud-risk

**Merge destes módulos:** `aurix-fraud` + `MlFraudService` + `CreditScoreService` (de `aurix-analytics`)

- [ ] **Steps 1–15:** Seguir padrão da Task 1

---

### Task 7: F7 — compliance

**Merge destes módulos:** `aurix-compliance` + `aurix-audit` + `aurix-tax` + `aurix-accounting` + `aurix-bacen` (relatórios Cosif)

- [ ] **Steps 1–15:** Seguir padrão da Task 1

**Atenção a duplicatas:**
- `ConciliacaoBancaria` — presente em accounting (canônico), core, settlement, financial → manter a versão de accounting
- `SaldoConta` — presente em accounting e settlement → manter em accounting
- `PlanoContas` — único em accounting

---

### Task 8: F8 — finance-mgmt

**Merge destes módulos:** `aurix-controller` + `aurix-budget` + `aurix-cost` + `aurix-financial`

- [ ] **Steps 1–15:** Seguir padrão da Task 1

**Atenção a duplicatas:**
- `Orcamento`, `ItemOrcamento`, `PlanejamentoEstrategico` — controller e budget → manter controller como canônico
- `CentroCusto` — controller e cost → manter cost como canônico
- `ConciliacaoBancaria` — financial tem cópia → descartar (canônico em compliance/accounting)

---

### Task 9: F9 — platform

**Merge destes módulos:** `aurix-provisioning` + `aurix-billing` + `aurix-webhooks` + `aurix-notification` + `aurix-catalog` + `aurix-baas` + `aurix-organization`

- [ ] **Steps 1–15:** Seguir padrão da Task 1

**Maior merge** — 7 módulos fonte, 28 entidades só de catalog. Subpacotes por módulo original.

---

### Task 10: F10 — intelligence

**Merge destes módulos:** `aurix-analytics` + `aurix-ai` + `aurix-openfinance` + `aurix-internet-banking` + `aurix-mobile-banking`

- [ ] **Steps 1–15:** Seguir padrão da Task 1

---

### Task 11: Remover módulos fonte (pós-migração)

Após todas as fases F1-F10 estarem verdes, remover módulos fonte:

- [ ] **Step 1:** Remover módulos de `apps/backend/pom.xml` (um por um, validando build)
- [ ] **Step 2:** Remover diretórios `apps/backend/aurix-*` (exceto `aurix-shared`)
- [ ] **Step 3:** Remover entradas legadas de `infra/docker-compose.yml`
- [ ] **Step 4:** Remover entradas legadas de `infra/traefik/dynamic.yml`
- [ ] **Step 5:** Remover jobs CI legados de `.github/workflows/`
- [ ] **Step 6:** Renomear `docker-compose.v2.yml` → `docker-compose.yml` e `dynamic.v2.yml` → `dynamic.yml`
- [ ] **Step 7:** Remover `aurix-gateway` módulo Maven (Traefik é o gateway real)
- [ ] **Step 8:** Commit final

```bash
git commit -m "feat: remove legacy modules, finalize 10-domain consolidation"
```
