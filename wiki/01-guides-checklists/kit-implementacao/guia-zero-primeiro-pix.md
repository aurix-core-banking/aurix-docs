# Do zero ao primeiro PIX em poucos dias

Passo a passo para novo cliente colocar o primeiro PIX no ar com o AUREUS. Item 13.4 do roadmap.

---

## Dia 1 – Conta e ambiente

1. **Contrato e acesso**: fechar contrato (AUREUS Cloud ou self-hosted); receber credenciais de acesso (portal, API Key ou OAuth) e o `tenant_id`.
2. **Checklist de config**: preencher o [checklist go-live](../01-guides-checklists/checklists/go-live.md) (BACEN, SPI/STR, KYC, domínios). Em sandbox/homolog, pode-se usar modo simulado para PIX.
3. **Provisioning**: se AUREUS Cloud, a instituição já estará cadastrada e provisionada; conferir config do tenant (branding, limites) no painel ou via API de provisioning.
4. **Primeira conta**: seguir o guia [Primeira conta](../../03-development/portal-desenvolvedor/01-guides-01-guides-checklists/checklists/guias/primeira-conta.md) para criar cliente e conta de teste; anotar o ID da conta e o número da conta.

---

## Dia 2 – Chave PIX e transferência

1. **Cadastrar chave PIX**: via API ou internet banking, cadastrar uma chave PIX (CPF, e-mail, telefone ou chave aleatória) vinculada à conta criada. Ver [APIs PIX](../../03-development/portal-desenvolvedor/apis/pix.md).
2. **Gerar QR PIX (opcional)**: usar `POST /api/pix/qr/gerar/cobranca` ou `gerar/transferencia` para obter o payload e exibir o QR no front-end ou em tela de caixa.
3. **Primeira transferência**: seguir o guia [Primeiro PIX](../../03-development/portal-desenvolvedor/01-guides-01-guides-checklists/checklists/guias/primeiro-pix.md); enviar uma transferência de teste (valor mínimo) para uma chave de destino válida. Em homologação, usar ambiente simulado do BACEN se disponível.
4. **Consultar status**: usar a API de transferências para verificar se a transferência foi concluída ou está pendente.

---

## Dia 3 – Integração e webhook

1. **Integrar no seu sistema**: usar a API Key ou OAuth para chamar as APIs do AUREUS a partir do seu backend (criar conta, consultar saldo, enviar PIX). Ver [Portal do desenvolvedor](../../03-development/portal-desenvolvedor/README.md) e documentação de cada API.
2. **Webhook**: configurar a URL de webhook no módulo webhooks (`PUT /api/webhooks/config/{tenantId}`) para receber eventos como `pix.transferencia_concluida` e atualizar seu sistema quando um PIX for liquidado.
3. **Teste end-to-end**: simular fluxo completo: abrir conta, cadastrar chave, receber um PIX (ou enviar) e verificar o webhook e o saldo.

---

## Dia 4–5 – Validação e go live

1. **Validação com BACEN**: em homologação, executar transações no ambiente do BACEN (SPI/STR) quando certificados estiverem prontos; validar mensagens e liquidação.
2. **Checklist final**: revisar [checklist go-live](../01-guides-checklists/checklists/go-live.md) (certificados, KYC, domínios, RegTech); confirmar que relatórios obrigatórios estão agendados.
3. **Go live**: trocar para produção (certificados e URLs de produção); comunicar usuários e monitorar primeiras transações; acionar suporte em caso de incidente conforme [Processo de suporte](suporte-tecnico.md).

---

## Resumo de APIs usadas

| Etapa | API / recurso |
|-------|----------------|
| Conta | POST/GET /api/core/clientes, /api/core/contas |
| Chave PIX | POST/GET /api/pix/chaves |
| Transferência PIX | POST /api/pix/transferencias |
| QR PIX | POST /api/pix/qr/gerar/cobranca ou /gerar/transferencia |
| Webhook | PUT /api/webhooks/config/{tenantId} |

Headers obrigatórios: `X-Tenant-Id`, `Authorization` (API Key ou Bearer) conforme contrato.

[Voltar ao kit](README.md) | [Índice da wiki](../README.md)
