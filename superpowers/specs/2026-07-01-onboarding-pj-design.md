# Onboarding PJ — Design Spec

## Goal

Expand the `aurix-onboarding` module to support **Pessoa Jurídica** onboarding with a separate domain-specific flow while reusing a shared core (protocol, documents, events, integrations, audit, common services).

## Architecture

```
Onboarding
├── Núcleo
│   ├── SolicitacaoOnboarding (protocolo comum)
│   ├── Documento (reusado do existente)
│   ├── Evento / Auditoria
│   ├── Serviços compartilhados (DocumentoService, KYCService, AMLService, CoreBankingService)
│   └── WorkflowEngine
├── PF (existente, refatorado)
│   ├── SolicitacaoPF (dados específicos)
│   ├── WorkflowPF
│   └── ControllerPF (/onboarding/contas/pf)
└── PJ (novo)
    ├── SolicitacaoPJ (dados específicos)
    ├── Empresa (Receita Federal snapshot)
    ├── Participante (sócios, admins, procuradores, beneficiário final)
    ├── WorkflowPJ
    └── ControllerPJ (/onboarding/contas/pj)
```

---

## 1. Núcleo Compartilhado

### SolicitacaoOnboarding (refatorado do existente `SolicitacaoConta`)

```
id: Long (PK)
tenantId: String
tipoPessoa: TipoPessoa [FISICA, JURIDICA]
status: String (delegado ao workflow — "EM_PREENCHIMENTO", etc.)
canal: String (e.g., "WEB", "MOBILE", "AGENCIA", "API")
produto: String (e.g., "CONTA_CORRENTE", "CONTA_DIGITAL")
dataCriacao: LocalDateTime
dataAtualizacao: LocalDateTime
```

### TipoPessoa enum

`FISICA, JURIDICA`

### Documento (reutilizado do existente `DocumentoOnboarding`)

Mantido como está — `@ManyToOne` para `SolicitacaoOnboarding`.

### Serviços Compartilhados

- **DocumentoService** — upload, validação, listagem (reusado)
- **KYCService** — interface `KycProvider` (existente, stub)
- **AMLService** — consulta PEP, sanções, bureau
- **CoreBankingService** — `CoreApiClient` expandido (criar cliente PF/PJ + conta)
- **NotificacaoService** — eventos de mudança de estado
- **WorkflowEngine** — define transições por tipoPessoa, valida status machine

---

## 2. Fluxo PF (Refatorado)

### SolicitacaoPF

Dados específicos extraídos da `SolicitacaoConta` atual.

```
id: Long (PK)
solicitacaoId: Long (FK → SolicitacaoOnboarding.id, unique)
cpf: String (11)
nome: String
dataNascimento: LocalDate
ocupacao: String
rendaDeclarada: BigDecimal
pep: Boolean
scoreBureau: Integer
resultadoKyc: String
clienteIdCriado: Long (FK → Cliente.id)
contaIdCriada: Long (FK → Conta.id)
contaLimitadaAteKyc: Boolean
observacoesAnalista: String
```

### WorkflowPF

Mantém o status machine existente:

```
RECEBIDA → DOCUMENTOS_PENDENTES → EM_ANALISE_KYC → KYC_APROVADO/KYC_REJEITADO
→ APROVADA → CONTA_CRIADA
```

### ControllerPF

`@RequestMapping("/onboarding/contas/pf")` — endpoints do fluxo PF existentes.

---

## 3. Fluxo PJ (Novo)

### SolicitacaoPJ

```
id: Long (PK)
solicitacaoId: Long (FK → SolicitacaoOnboarding.id, unique)
cnpj: String (14)
razaoSocial: String
nomeFantasia: String
naturezaJuridica: String (e.g., "EI", "LTDA", "SA", "MEI")
porte: PorteEmpresa [MEI, ME, EPP, DEMAIS]
capitalSocial: BigDecimal
dataConstituicao: LocalDate
inscricaoEstadual: String
inscricaoMunicipal: String
faturamentoMensal: BigDecimal
numeroFuncionarios: Integer
clienteIdCriado: Long (FK → Cliente.id, nullable)
contaIdCriada: Long (FK → Conta.id, nullable)
observacoesAnalista: String
```

### PorteEmpresa enum

```
MEI, ME, EPP, DEMAIS
```

### Empresa (snapshot da Receita Federal)

```
id: Long (PK)
solicitacaoId: Long (FK → SolicitacaoOnboarding.id, unique)
cnpj: String (14)
razaoSocial: String
nomeFantasia: String
cnaePrincipal: String
cnaeSecundarios: String (JSONB array)
endereco: String (JSONB)
situacaoCadastral: SituacaoCNPJ [ATIVA, INAPTA, BAIXADA, SUSPENSA, NULA]
dataSituacao: LocalDate
regimeTributario: String (e.g., "SIMPLES_NACIONAL", "MEI", "LUCRO_PRESUMIDO", "LUCRO_REAL")
dadosAbertos: String (JSONB — resposta completa da Receita)
```

### Participante

```
id: Long (PK)
solicitacaoId: Long (FK → SolicitacaoOnboarding.id)
tipo: TipoParticipante
cpf: String (11)
nome: String
email: String
telefone: String
dataNascimento: LocalDate
nacionalidade: String
qualificacao: String (e.g., "Sócio-Administrador", "Diretor")
percentualParticipacao: BigDecimal
documentos: List<Documento> (@OneToMany)
validado: Boolean
```

### TipoParticipante enum

```
SOCIO, ADMINISTRADOR, REPRESENTANTE, PROCURADOR, BENEFICIARIO_FINAL
```

### WorkflowPJ

```
EM_PREENCHIMENTO → CNPJ_CONSULTADO → SOCIOS_VALIDADOS →
DOCUMENTOS_ANALISADOS → AML_APROVADO → COMPLIANCE_APROVADO →
EM_ASSINATURA → CONTRATO_ASSINADO → CONTA_CRIADA
```

Rejeição possível em qualquer etapa: → `REJEITADA`

### ControllerPJ

`@RequestMapping("/onboarding/contas/pj")`

| Method | Path | Description |
|--------|------|-------------|
| POST | `/` | Iniciar onboarding PJ (body: cnpj + dados básicos) |
| GET | `/{id}` | Consultar status |
| GET | `/` | Listar (back office, filtro por status) |
| POST | `/{id}/socios` | Adicionar sócio/participante |
| DELETE | `/{id}/socios/{participanteId}` | Remover participante |
| POST | `/{id}/documentos` | Adicionar documento |
| POST | `/{id}/validar-cnpj` | Consultar Receita Federal, popular Empresa |
| POST | `/{id}/aprovar` | Aprovar (+ criar cliente PJ + conta) |
| POST | `/{id}/rejeitar` | Rejeitar |
| GET | `/{id}/socios` | Listar participantes |

---

## 4. Refatorações no Existente

1. `SolicitacaoConta` → `SolicitacaoOnboarding` (núcleo) + `SolicitacaoPF` (específico)
2. `OnboardingService` → mantém lógica PF, novo `OnboardingPFService`
3. Novo `OnboardingPJService` + atualizações
4. `CoreApiClient` expandido: criarClientePJ (envia cnpj, razaoSocial, tipoPessoa)
5. `OnboardingController` → `ControllerPF` + `ControllerPJ`

---

## 5. Integrações

- **Receita Federal:** `consultarCnpj(cnpj)` → preenche Empresa + sócios (quando API real retornar quadro societário)
- **KYC/AML:** reusado do núcleo para validar participantes
- **Core Banking:** `POST /api/core/clientes` + `POST /api/core/contas` (já existe, expandir para PJ)

---

## 6. Non-goals (MVP)

- Assinatura digital de contrato — `EM_ASSINATURA` como estado terminal até integração real
- Consulta real de CNPJ na Receita (stub existente é suficiente para MVP)
- Integração real de KYC/AML (stubs existentes)
- Notificações push/email — emitir evento internamente apenas
- Onboarding de outros tipos (Instituições Financeiras, Governo) — apenas PF + PJ

---

## 7. Dependências

`aurix-onboarding` já depende de `aurix-shared`. O Cliente PJ consolidado (Sub-project 1) já existe em `aurix-shared`. O módulo `aurix-catalog` (Sub-project 2) não é dependência direta do onboarding — será usado pelo frontend para exibir ofertas durante o fluxo.
