# Admin PF Pages — Design

## Goal
Rewrite existing `solicitacoes_conta` React Admin resource with tabbed layout matching `solicitacoes_pj` quality.

## Changes

### Resource config
`solicitacoes_conta: { base: '/api/onboarding/contas/pf', path: 'solicitacoes' }`
- Maps to `POST / GET /api/onboarding/contas/pf/solicitacoes`, matching `ControllerPF` (`@RequestMapping("/contas/pf")`)

### Refactored components

1. **`DocumentList`** — accepts optional `resourceName` prop (default `'solicitacoes_pj'`). PF passes `resourceName="solicitacoes_conta"`.

2. **`StatusTimeline`** — accepts optional `statusOrder` / `statusLabels` props. PF passes PF-only order:
   `RECEBIDA → DOCUMENTOS_PENDENTES → EM_ANALISE_KYC → KYC_APROVADO → KYC_REJEITADO → APROVADA → CONTA_CRIADA → REJEITADA`

3. **`WorkflowActionsPF`** (new) — PF-specific workflow:
   | Status | Action | Endpoint |
   |--------|--------|----------|
   | RECEBIDA | Enviar para KYC | `POST /{id}/kyc` |
   | DOCUMENTOS_PENDENTES | Enviar para KYC | `POST /{id}/kyc` |
   | KYC_APROVADO | Aprovar (criar cliente+conta) | `POST /{id}/aprovar` |
   | Any (non-terminal) | Rejeitar | `POST /{id}/rejeitar` |

### Pages

**`SolicitacaoContaList`** — SearchInput (cpf), SelectInput (status with PF-only statuses), Datagrid: id, cpf, nome, status, dataCriacao + ShowButton

**`SolicitacaoContaShow`** — TabbedShowLayout:
- Detalhes: id, cpf, nome, email, telefone, dataNascimento, ocupacao, rendaDeclarada, status, pep, scoreBureau, resultadoKyc, clienteIdCriado, contaIdCriada, contaLimitadaAteKyc, observacoesAnalista, dataCriacao, dataAtualizacao
- Documentos: DocumentList (resourceName="solicitacoes_conta")
- Histórico: StatusTimeline (PF order)
- Ações: WorkflowActionsPF

### PF Status Order (for StatusTimelinePF)
```
RECEBIDA, DOCUMENTOS_PENDENTES, EM_ANALISE_KYC, KYC_APROVADO,
KYC_REJEITADO, APROVADA, CONTA_CRIADA, REJEITADA
```
