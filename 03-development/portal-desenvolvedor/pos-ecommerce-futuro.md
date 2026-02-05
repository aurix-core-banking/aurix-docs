# POS e e-commerce (futuro)

Integracao com PDV (POS) e e-commerce esta prevista para uma proxima fase. Este documento resume o escopo futuro.

---

## Escopo previsto

- **QR PIX no PDV**: exibir QR Code de cobranca PIX na tela do caixa ou gerar payload para impressao; consulta de status de pagamento.
- **Webhook de confirmacao**: ao receber pagamento PIX ou boleto, o webhook configurado pelo tenant pode notificar o sistema do parceiro (POS/e-commerce) para liberar pedido ou atualizar status.
- **Exemplos de integracao**: codigo de exemplo (ex.: Node ou Java) mostrando fluxo completo: gerar cobranca, exibir QR, poll ou webhook de confirmacao, concluir venda.

---

## Uso atual

Enquanto a integracao dedicada POS/e-commerce nao for entregue, e possivel:

1. Usar a **API de QR PIX** (`POST /api/pix/qr/gerar/cobranca`) para obter o payload e exibir o QR no front-end (ou em tela do PDV).
2. Configurar **webhook** (docs/portal-desenvolvedor/webhooks.md) com a URL do seu sistema; ao concluir o PIX ou boleto, o AUREUS envia o evento e o payload para essa URL.
3. Implementar no seu sistema o endpoint que recebe o webhook e atualiza o pedido/transacao.

Quando houver documentacao e exemplos especificos para POS/e-commerce, eles serao publicados aqui ou em guia dedicado.
