# AUREUS - Integracao entre Modulos

Integracao entre todos os modulos da plataforma AUREUS: comunicacao sincrona e assincrona, cache compartilhado e APIs unificadas.

## Componentes

1. **IntegrationService** - Comunicacao sincrona entre modulos
2. **EventPublisher** - Publicacao de eventos Kafka
3. **EventListener** - Processamento de eventos assincronos
4. **SharedCacheService** - Cache compartilhado Redis
5. **IntegrationController** - APIs de integracao

## Fluxos

- **Criacao de conta**: aureus-core -> EventPublisher -> Kafka -> EventListener -> SharedCacheService; aureus-financial <- IntegrationService
- **Transacao**: aureus-core -> Kafka/EventListener; aureus-financial, aureus-controller, aureus-tax, aureus-accounting <- IntegrationService
- **Tarifas**: aureus-pricing <- IntegrationService; SharedCacheService -> Redis

## APIs de Integracao

- `GET /api/integration/clientes/{clienteId}`, `GET .../contas/{contaId}`, `GET .../transacoes/{transacaoId}`
- `POST /api/integration/contas/{contaId}/sincronizar`, `POST .../transacoes/{transacaoId}/sincronizar`
- `POST /api/integration/tarifas/calcular`
- `GET /api/integration/dashboard`
- `POST /api/integration/cache/limpar`, `POST .../cache/limpar/{padrao}`

## Eventos Kafka

- Conta: conta-criada, conta-atualizada, conta-bloqueada
- Transacao: transacao-realizada, transacao-liquidada, transacao-conciliada
- Imposto: imposto-calculado, imposto-registrado

## Cache Compartilhado

- Clientes (TTL 1h), Contas (30min), Transacoes (15min), Configuracoes (24h), Tarifas (1h)
- Operacoes: buscar/salvar/remover por cliente, conta, transacao

## Configuracao

URLs dos modulos em `aureus.integration.*` (core, financial, controller, tax, accounting, pricing). Kafka bootstrap e Redis em config padrao.
