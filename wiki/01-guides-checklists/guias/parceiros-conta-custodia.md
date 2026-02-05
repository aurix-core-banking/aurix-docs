# BaaS – Contas sob custódia para parceiros

Documentação para parceiros que integram com o AUREUS em modo BaaS (Banking as a Service): como usar as APIs de conta sob custódia, consentimento e movimentação.

---

## Visão geral

A instituição (tenant) pode oferecer a parceiros a capacidade de:

- Criar **sub-contas** vinculadas a contas existentes no core, identificadas por um ID externo do parceiro.
- **Consultar saldo** de uma conta, desde que exista consentimento ativo do titular com escopo `CONSULTAR_SALDO`.
- **Movimentar** (PIX) a partir de uma conta, com consentimento ativo com escopo `MOVIMENTAR`.

Todas as requisições exigem o tenant da instituição e o identificador do parceiro (client_id). O consentimento é registrado pela instituição ou pelo fluxo do titular da conta.

---

## Autenticação e headers

- **X-Tenant-Id**: identificador do tenant (instituição). Obrigatório.
- **X-Client-Id**: identificador do parceiro (client_id cadastrado na instituição). Obrigatório nas APIs de custódia.
- **Idempotency-Key**: recomendado em requisições de movimentação (PIX) para evitar duplicidade. Se não enviado, um UUID é gerado.

A instituição cadastra o parceiro (client_id e nome) via `POST /api/baas/parceiros` e repassa as credenciais ao parceiro. Em produção, a autenticação pode ser reforçada com API Key ou OAuth2.

---

## Fluxo típico

1. **Instituição cadastra o parceiro**  
   `POST /api/baas/parceiros` com body `{ "clientId": "parceiro-x", "nome": "Parceiro X" }`. Headers: `X-Tenant-Id`.

2. **Registro de consentimento**  
   O titular da conta autoriza o parceiro a consultar saldo e/ou movimentar. A instituição (ou o próprio fluxo de consentimento) chama  
   `POST /api/baas/custodia/consentimentos` com `contaId`, `parceiroId`, `escopos` (lista com `CONSULTAR_SALDO`, `MOVIMENTAR`) e `dataExpiracao` (ISO-8601).

3. **Parceiro cria sub-conta (vínculo)**  
   O parceiro associa uma conta existente a um identificador próprio:  
   `POST /api/baas/custodia/subcontas` com `{ "contaId": 123, "identificadorExterno": "user-456" }`.  
   A partir daí, o parceiro pode usar `contaId` nas chamadas de saldo e movimentação (desde que o consentimento exista para essa conta e parceiro).

4. **Consultar saldo**  
   `GET /api/baas/custodia/saldo?contaId=123`. Headers: `X-Tenant-Id`, `X-Client-Id`. Resposta: `{ "saldo": 1500.00 }`.

5. **Movimentar (PIX)**  
   `POST /api/baas/custodia/movimentar/pix` com body `{ "contaIdOrigem": 123, "chaveDestino": "cpf/email/pix", "valor": 100.00 }`. Header opcional: `Idempotency-Key`. Resposta: `{ "idTransacao": "..." }`.

---

## Endpoints resumidos

| Método | Path | Descrição |
|--------|------|------------|
| POST | /api/baas/parceiros | Cadastrar parceiro (instituição) |
| GET | /api/baas/parceiros | Listar parceiros do tenant |
| POST | /api/baas/custodia/subcontas | Criar sub-conta (vínculo conta + identificador externo) |
| GET | /api/baas/custodia/subcontas | Listar sub-contas do parceiro |
| POST | /api/baas/custodia/consentimentos | Registrar consentimento (conta, parceiro, escopos, expiração) |
| GET | /api/baas/custodia/saldo?contaId= | Consultar saldo (com consentimento CONSULTAR_SALDO) |
| POST | /api/baas/custodia/movimentar/pix | Enviar PIX (com consentimento MOVIMENTAR) |

---

## Escopos de consentimento

- **CONSULTAR_SALDO**: permite ao parceiro consultar o saldo da conta.
- **MOVIMENTAR**: permite ao parceiro iniciar transferência PIX a partir da conta.

O consentimento possui data de expiração e status (ATIVO, REVOGADO, EXPIRADO). Apenas consentimentos ATIVOS e não expirados são aceitos.

---

## Branding e termos

A instituição configura branding (logo, cores) e **termos de uso** no módulo de provisioning (`TenantConfig`). O campo `termosUsoUrl` pode apontar para a URL dos termos exibidos ao titular da conta no fluxo de consentimento. Consulte a API de config do tenant para obter ou alterar essa URL.

---

## Referências

- [Portal do desenvolvedor](../../03-development/portal-desenvolvedor/README.md): APIs gerais do AUREUS.
- [Templates (termos, privacidade)](../01-guides-checklists/kit-implementacao/templates.md)

[Voltar a Guias](README.md) | [Índice da wiki](../README.md)
