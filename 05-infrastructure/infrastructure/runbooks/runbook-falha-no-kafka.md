# Runbook – Falha no Kafka

Diagnóstico e recuperação de problemas de mensageria: consumidor parado, lag crescente e replay de mensagens.

## Contexto

- Broker: Kafka (Confluent) em `localhost:9092` (dev).
- Produtores: serviços `svc-*` publicam eventos (ex.: `ClienteCriadoEvent`, `TransacaoAutorizadaEvent`) e o `svc-fraud` consome para pontuar fraude. O `svc-banking` usa outbox (`OutboxEvent` → `OutboxRelay`).

## Sintomas

- `consumer lag` crescente (métricas `kafka_consumer_lag`).
- Consumidor parado (offset não avança).
- Erros `GroupCoordinator not available`, `LeaderNotAvailable`, timeouts de rede.
- Alertas: `KafkaConsumerLag`, `KafkaDown`.

## Passo a passo

### 1. Verificar o broker

```bash
docker compose -f docker-compose.v2.yml ps kafka zookeeper
docker compose -f docker-compose.v2.yml logs --tail 100 kafka
```

- Broker com `not available`/`LeaderNotAvailable`: aguardar retry ou reiniciar o broker; verificar Zookeeper (ou KRaft) e rede.
- Confirmar topicos existentes:
  ```bash
  docker exec aurix-kafka kafka-topics --bootstrap-server localhost:9092 --list
  ```

### 2. Verificar lag dos consumidores

```bash
docker exec aurix-kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 --describe --all-groups
```

- Observar coluna `LAG`. Lag alto e estável = consumidor processando devagar; lag crescente = consumidor parado ou produtor acelerando.

### 3. Consumidor parado ou com erro

1. Verificar logs do serviço consumidor (ex.: `svc-fraud`): exceções de desserialização, commit de offset falhando, mensagem poison pill.
2. Se uma mensagem inválida trava o processamento: identificar o offset e **pular**:
   ```bash
   docker exec aurix-kafka kafka-consumer-groups \
     --bootstrap-server localhost:9092 --group <grupo> --topic <topico> \
     --reset-offsets --to-offset <offset+1> --execute
   ```
3. Corrigir a causa no código (validação/tratamento de mensagem) e redeploy.

### 4. Replay de mensagens

Usado quando o consumidor precisou reprocessar mensagens (correção de bug ou dados perdidos):

```bash
docker exec aurix-kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 --group <grupo> \
  --reset-offsets --to-earliest --execute
```

> Cuidado: reprocessamento pode gerar duplicidade no destino. Se o consumidor não for **idempotente**, coordenar com o time antes do replay, ou usar `--to-datetime` para reprocessar só a janela do incidente.

### 5. Produtor falhando (outbox)

- O `svc-banking` publica via **outbox**: eventos ficam na tabela `outbox_events` e o `OutboxRelay` envia ao Kafka. Se o broker ficar fora por muito tempo, a fila outbox cresce.
- Verificar: `SELECT count(*) FROM outbox_events WHERE publicado = false;`
- Após o broker voltar, o relay retoma automaticamente; monitorar até o backlog zerar.

## Pós-incidente

- Registrar grupo/tópico, lag máximo, causa raiz.
- Adicionar alerta de lag (ex.: > 10.000 por 5 minutos) e monitorar métricas Kafka no Prometheus.
- Se houver poison pill recorrente, implementar DLQ (`DeadLetterQueue` do EventHub) para isolar mensagens inválidas.
