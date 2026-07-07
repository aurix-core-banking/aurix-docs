# Frontend Web — Onboarding PF/PJ (aureus-web)

## Overview

Refatorar a página `/onboarding` do `aureus-web` (Internet Banking) para suportar abertura de conta **PF e PJ** integradas, com criação + acompanhamento pós-submissão.

## Arquitetura

A página `Onboarding.js` é reescrita como um container com state machine de 3 modos:

```
select → form (PF ou PJ) → tracking (pós-criação)
```

### State machine

```
SELECT ──escolhe PF──→ FORM (tipo=PF) ──submit──→ TRACKING (id=...)
SELECT ──escolhe PJ──→ FORM (tipo=PJ) ──submit──→ TRACKING (id=...)
TRACKING ──"Nova solicitação"──→ SELECT
```

### Modos

| Mode | Descrição |
|------|-----------|
| `select` | Dois cards grandes (PF / PJ) para escolha inicial |
| `form` | Formulário de criação (PF simples ou PJ wizard multi-step) |
| `tracking` | Acompanhamento pós-criação com status, timeline, documentos, sócios |

## Componentes

```
src/pages/Onboarding.js           → container com state machine
src/components/Onboarding/
  TipoSelector.js                 → cards PF/PJ
  FormPF.js                       → formulário PF (8 campos)
  FormPJ/
    WizardPJ.js                   → stepper MUI com 4 steps
    StepEmpresa.js                → dados da empresa
    StepSocios.js                 → adicionar/remover sócios
    StepDocumentos.js             → upload documentos
    StepRevisao.js                → revisão final + submit
  TrackingPJ.js                   → status + timeline + docs + sócios
```

### TipoSelector
Dois cards lado a lado com ícone (Person / Business), título e descrição curta. Clique define `tipo` e avança para `mode: 'form'`.

### FormPF
Campos: CPF (com máscara), Nome, E-mail, Telefone, Data de Nascimento, Ocupação, Renda Declarada, Endereço.
Submit → `POST /contas/pf/solicitacoes` → recebe `id` → vai para tracking.

### WizardPJ
MUI Stepper horizontal com 4 steps:

1. **StepEmpresa**: CNPJ (máscara), Razão Social, Nome Fantasia, E-mail, Telefone, Endereço
2. **StepSocios**: Tabela de sócios com botão "Adicionar" → modal com CPF, Nome, E-mail, Telefone, Data Nascimento, Nacionalidade, Qualificação, % Participação. Botão remover por linha.
3. **StepDocumentos**: Lista de documentos com input file (react-dropzone já é dependência). Submit individual via `POST /{id}/documentos`.
4. **StepRevisao**: Resumo dos dados + botão "Confirmar e enviar". Submit → `POST /contas/pj`.
   - CNPJ é validado primeiro via `POST /{id}/validar-cnpj` (opcional, se API disponível)

Após submit, recebe `id` e vai para tracking.

### TrackingPJ
Após criação:
- GET `/contas/pj/{id}` → exibe dados da solicitação
- Status badge com cor (EM_PREENCHIMENTO, CNPJ_CONSULTADO, etc.)
- Timeline do admin (StatusTimeline adaptado)
- Tabela de documentos
- Tabela de sócios
- Botão "Nova solicitação" → volta para `select`

## Data Flow

### API Service (`apiService.js`)

Métodos adicionados:

```js
// PF
criarSolicitacaoPF(data)          → POST /api/onboarding/contas/pf/solicitacoes
getSolicitacaoPF(id)              → GET  /api/onboarding/contas/pf/solicitacoes/{id}

// PJ
criarSolicitacaoPJ(data)          → POST /api/onboarding/contas/pj
getSolicitacaoPJ(id)              → GET  /api/onboarding/contas/pj/{id}
validarCNPJPJ(id)                 → POST /api/onboarding/contas/pj/{id}/validar-cnpj
adicionarSocioPJ(id, socio)       → POST /api/onboarding/contas/pj/{id}/socios
removerSocioPJ(id, socioId)       → DELETE /api/onboarding/contas/pj/{id}/socios/{socioId}
adicionarDocumentoPJ(id, doc)     → POST /api/onboarding/contas/pj/{id}/documentos
```

### Backend endpoints usados

| Método | Endpoint | Request | Response |
|--------|----------|---------|----------|
| POST | `/contas/pf/solicitacoes` | `SolicitacaoContaRequest` | `SolicitacaoContaResponse` |
| GET | `/contas/pf/solicitacoes/{id}` | — | `SolicitacaoContaResponse` |
| POST | `/contas/pj` | `SolicitacaoPJRequest` | `SolicitacaoPJResponse` |
| GET | `/contas/pj/{id}` | — | `SolicitacaoPJResponse` |
| POST | `/contas/pj/{id}/socios` | `ParticipanteRequest` | 204 |
| DELETE | `/contas/pj/{id}/socios/{participanteId}` | — | 204 |
| POST | `/contas/pj/{id}/documentos` | `{ tipoDocumento, nomeArquivo, urlStorage }` | 204 |
| POST | `/contas/pj/{id}/validar-cnpj` | — | `SolicitacaoPJResponse` |

### Fixes

1. **API path bug**: O Onboarding.js atual usa `api/onboarding/onboarding/solicitacoes` (onboarding duplicado). Corrigir para `api/onboarding/contas/pf/solicitacoes`.
2. **Raw fetch → axios**: Migrar para `apiService` consistente com o resto do app.

## State & Error Handling

- `loading`, `error`, `success` via `useState` em cada step
- `error` exibido em MUI `Alert` (padrão Login.js)
- Validação manual no `handleSubmit`: campos obrigatórios, formato CNPJ (14 dígitos), CPF (11 dígitos), e-mail
- Tracking mostra `Alert` se `id` não encontrado
- Botões desabilitados durante `loading`

## Testes

- `Onboarding.test.js` atualizado: testar seletor PF/PJ, submissão PF, wizard PJ completo, tracking
- Mock `apiService` (já existe `__mocks__/apiService.js`)

## Arquivos modificados

| Arquivo | Ação |
|---------|------|
| `src/pages/Onboarding.js` | Reescrever (state machine + fluxo completo) |
| `src/pages/Onboarding.test.js` | Atualizar tests |
| `src/services/apiService.js` | +7 métodos de onboarding |
| `src/services/apiService.test.js` | + tests novos métodos |
| `src/components/Onboarding/TipoSelector.js` | Criar |
| `src/components/Onboarding/FormPF.js` | Criar |
| `src/components/Onboarding/FormPJ/WizardPJ.js` | Criar |
| `src/components/Onboarding/FormPJ/StepEmpresa.js` | Criar |
| `src/components/Onboarding/FormPJ/StepSocios.js` | Criar |
| `src/components/Onboarding/FormPJ/StepDocumentos.js` | Criar |
| `src/components/Onboarding/FormPJ/StepRevisao.js` | Criar |
| `src/components/Onboarding/TrackingPJ.js` | Criar |
