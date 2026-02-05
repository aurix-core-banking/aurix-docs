# Webhooks

Configuracao de URL por tenant e envio de eventos quando uma transacao e concluida (PIX, boleto, etc.). Retry automatico e log de falhas.

---

## Eventos suportados

- `pix.transferencia_concluida` – transferencia PIX concluida
- `pix.transferencia_falhou` – transferencia PIX falhou
- `boleto.pago` – boleto pago
- `boleto.vencido` – boleto vencido
- `transacao.concluida` – transacao generica concluida

A lista de eventos pode ser configurada por tenant; se vazia, todos os eventos sao enviados.

---

## Configurar webhook (por tenant)

**PUT** `/api/webhooks/config/{tenantId}`

Headers: `X-Tenant-Id` (opcional para identificar o tenant no config).

Body:
```json
{
  "url": "https://seu-servidor.com/webhook",
  "eventos": ["pix.transferencia_concluida", "boleto.pago"],
  "ativo": true,
  "secret": "opcional-assinatura"
}
```

- **url**: obrigatorio; endpoint que recebera POST com o payload.
- **eventos**: opcional; lista de eventos a enviar. Se omitido ou vazio, envia todos.
- **ativo**: opcional; default true.
- **secret**: opcional; pode ser usado para assinatura (X-Webhook-Signature em implementacoes futuras).

**GET** `/api/webhooks/config/{tenantId}` – retorna a config atual.

---

## Payload enviado

O servico envia **POST** para a URL configurada com:

```json
{
  "evento": "pix.transferencia_concluida",
  "payload": {
    "id": 123,
    "codigoPix": "abc-...",
    "valor": 100.00,
    "chaveDestino": "...",
    "status": "CONCLUIDO",
    "dataLiquidacao": "2025-02-03T14:00:00"
  }
}
```

O cliente deve responder com **2xx** para sucesso. Qualquer outro codigo ou falha de rede dispara retry.

---

## Retry e log

- **Tentativas**: ate 5 (configuravel por `aureus.webhooks.max-retries`).
- **Intervalo**: 5 minutos entre tentativas (configuravel por `aureus.webhooks.retry-interval-minutes`).
- **Log**: cada envio e registrado em `webhook_log` (tenant, evento, status, tentativas, response_code). Após exceder as tentativas, o status fica EXCEDIDO.

**GET** `/api/webhooks/config/{tenantId}/logs?limit=50` – lista os ultimos envios (sucesso e falha) para o tenant.

---

## Disparar evento (interno)

Servicos como PIX ou core podem disparar webhook chamando:

**POST** `/api/webhooks/dispatch`

Headers: `X-Tenant-Id` (obrigatorio).

Body:
```json
{
  "evento": "pix.transferencia_concluida",
  "payload": { ... }
}
```

O modulo enfileira o envio e retorna 202 Accepted. O envio e feito de forma assincrona; retries sao tratados pelo job agendado.
