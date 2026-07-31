# Runbook – Lentidão no banco

Diagnóstico e tratamento de lentidão no PostgreSQL compartilhado (`aurix_db`): queries lentas, locks, conexões esgotadas e failover.

> Todos os serviços `svc-*` compartilham o mesmo PostgreSQL. Uma query cara de um serviço afeta os demais. Ações devem ser cautelosas.

## Sintomas

- Latência alta (p95) em APIs de vários serviços ao mesmo tempo.
- Conexões no limite (`max_connections`); erros `too many clients`.
- Alertas: `DatabaseSlowQueries`, `DatabaseConnectionsHigh`, disco alto.
- Timeout em consultas (`statement timeout`, `canceling statement due to user request`).

## Passo a passo

### 1. Identificar queries lentas e locks

```bash
docker exec aurix-postgres psql -U aurix_user -d aurix_db -c "
SELECT pid, now()-query_start AS duracao, state, wait_event_type, left(query, 120)
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start LIMIT 20;"
```

### 2. Corrigir o padrão de acesso

- **Otimizar a query**: adicionar/ajustar índice, reescrever a query, reduzir dados retornados (paginacao).
- **Adicionar índice** (exemplo):
  ```sql
  CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_transacoes_conta_data
    ON transacoes (conta_id, criado_em DESC);
  ```
- **Limitar tempo de execução** por transação/sessão no driver JDBC (`statement_timeout`) para evitar que queries ruins travem o banco.

### 3. Tratar locks

```sql
-- Ver bloqueios e o que está bloqueando
SELECT blocked.pid AS bloqueado, blocking.pid AS bloqueador,
       left(blocked.query, 60) AS query_bloqueada
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.state <> 'idle';
```

- Identificar a transação longa que segura o lock; finalizar com `SELECT pg_terminate_backend(<pid>);` se confirmado como transação travada.
- Corrigir a causa raiz no serviço (transação sem commit, lock de linha demorado, migração no horário errado).

### 4. Ajustar conexões e recursos

- Verificar uso de conexões:
  ```bash
  docker exec aurix-postgres psql -U aurix_user -d aurix_db -c \
    "SELECT state, count(*) FROM pg_stat_activity GROUP BY state;"
  ```
- Ajustar `max_connections` e `shared_buffers` (preferir pool de conexões na aplicação, ex.: HikariCP com tamanho por serviço).
- Verificar disco e CPU do container: `docker stats aurix-postgres`.

### 5. Failover (instância primária comprometida)

- **Managed (RDS/Cloud SQL)**: acionar failover automático (multi-AZ) e monitorar o novo primário.
- **On-prem/Compose**: em caso de corrupção ou indisponibilidade prolongada, subir a réplica/PITR:
  1. Promover a réplica ou restaurar último backup consistente — ver [DR](../dr-procedimento.md).
  2. Atualizar `POSTGRES_HOST`/URL de conexão nos serviços.
  3. Validar health de todos os serviços.

## Pós-incidente

- Registrar a query/índice que causou o incidente.
- Adicionar índice no próximo deploy de migração (nunca em produção sem validação, preferir `CREATE INDEX CONCURRENTLY`).
- Revisar limites de conexão por serviço no HikariCP.
