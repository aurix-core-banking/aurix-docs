# Runbook – Queda de serviço

Diagnóstico e recuperação quando um serviço `svc-*` está fora do ar, não sobe ou responde com erro.

## Sintomas

- Health check em `/actuator/health` retornando DOWN ou timeout.
- Alertas de indisponibilidade (ex.: regra Prometheus `sla-uptime.yml`).
- Erros 5xx / connection refused no gateway (Traefik, porta 8080).
- Pod/container reiniciando (CrashLoopBackOff) ou status `Exited`.

## Passo a passo

### 1. Confirmar o serviço afetado

- Identificar qual serviço: `svc-banking`, `svc-payments`, `svc-credit`, etc. A porta de cada serviço está no `docker-compose.yml`/`docker-compose.v2.yml`.
- Verificar o health check:
  ```bash
  curl -sf http://localhost:<PORTA>/actuator/health
  ```

### 2. Coletar informações

- **Docker Compose**: `docker compose -f docker-compose.v2.yml ps` e `docker compose -f docker-compose.v2.yml logs --tail 200 <servico>`.
- **Kubernetes**: `kubectl get pods -n <namespace>`; `kubectl logs <pod> --tail 200`; `kubectl describe pod <pod>`.
- Registrar: mensagem de erro, timestamp, mudanças recentes (deploy/config), eventos de infra (reinício de banco, broker, rede).

### 3. Diagnóstico por causa provável

| Causa | Como verificar | Ação |
|-------|----------------|------|
| Config/secrets errados | Logs com `Connection refused`, `Failed to bind`, `Invalid login` | Corrigir variáveis/ConfigMap/Secret e reiniciar |
| Dependência fora (DB/Kafka/Redis) | Health das dependências; logs com timeout de conexão | Subir dependência primeiro; ver runbooks de banco e Kafka |
| Migração falhou | Logs com `FlywayException`/`Migration failed` | Corrigir migração; ver [rollback de deploy](runbook-rollback-deploy.md) |
| Recurso esgotado (memória/disco) | `docker stats`, métricas Prometheus | Escalar/limpar; ajustar limites |
| Deploy com defeito | Comparar versão atual vs anterior | Rollback; ver [rollback de deploy](runbook-rollback-deploy.md) |

### 4. Recuperação

1. Corrigir a causa identificada no passo 3.
2. Reiniciar o serviço:
   ```bash
   docker compose -f docker-compose.v2.yml restart <servico>
   # ou em K8s:
   kubectl rollout restart deployment <servico>
   ```
3. Aguardar o health check voltar a UP:
   ```bash
   until curl -sf http://localhost:<PORTA>/actuator/health; do sleep 5; done
   ```
4. Validar fluxo crítico do domínio (ex.: para `svc-banking`, consultar saldo e executar uma transferência de teste).

### 5. Pós-incidente

- Registrar incidente: causa raiz, duração, ações tomadas, quem acionou.
- Atualizar este runbook se houve passo não documentado.
- Se aplicável, adicionar alerta para detectar a falha antes.

## Escalação

- Sem recuperação em 15 minutos: acionar dev on-call do serviço.
- Banco ou Kafka envolvidos: ver [lentidão no banco](runbook-lentidao-no-banco.md) e [falha no Kafka](runbook-falha-no-kafka.md).
- Indisponibilidade prolongada: acionar procedimento de [DR](../dr-procedimento.md).
