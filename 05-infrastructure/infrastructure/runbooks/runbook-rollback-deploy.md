# Runbook – Rollback de deploy

Procedimento para reverter um deploy com defeito, incluindo migrações de banco, Docker Compose e Kubernetes.

## Decisão de rollback

Reverter quando, após o deploy:

- Health check do serviço fica DOWN.
- Taxa de erro 5xx acima do limite por mais de 5 minutos.
- Latência muito acima do SLA ou funcionalidade crítica quebrada (ex.: falha ao autorizar PIX).
- Migração de banco falhou ou corrompeu o schema.

## Passo a passo

### 1. Avaliar o impacto

- Identificar a versão do deploy atual e a última versão estável (tag/branch/commit/imagem).
- Verificar se o defeito é isolável (feature flag) — se sim, preferir desligar a flag ao rollback. Ver [feature-flags](../feature-flags.md).
- Registrar: versão atual, versão alvo, janela, serviços afetados.

### 2. Rollback do código/imagem

**Docker Compose:**

```bash
# Rebuild da stack com a versão estável e reinício
docker compose -f docker-compose.v2.yml up -d --no-deps --build <servico>
# ou restart com a imagem anterior (se tag versionada)
docker compose -f docker-compose.v2.yml stop <servico>
docker compose -f docker-compose.v2.yml up -d --no-deps <servico>
```

**Kubernetes:**

```bash
kubectl rollout undo deployment/<servico> -n <namespace>
kubectl rollout status deployment/<servico> -n <namespace>
```

> Se o deploy usou imagem com tag fixa (`latest`), refazer o deploy apontando para a tag estável anterior.

### 3. Rollback de migração de banco

> **Regra de ouro**: migrações devem ser compatíveis com a versão anterior (colunas novas como nullable; sem drop imediato). Rollback de banco é mais difícil que rollback de código.

- **Flyway**: se a migração `V<n>` do deploy falhou, corrigir o SQL e reexecutar, ou:
  ```bash
  # reverter apenas em último caso (ex.: com flyway repair)
  mvn flyway:repair
  ```
- Se a migração aplicou mudança incompatível e a versão anterior do serviço não funciona com o schema novo:
  1. Restaurar o banco do backup consistente anterior — ver [backup-restore](../backup-restore.md).
  2. Redeplyar o código na versão anterior.
- Sempre validar o schema após o rollback (`SELECT * FROM flyway_schema_history ORDER BY installed_rank;`).

### 4. Validar

```bash
curl -sf http://localhost:<PORTA>/actuator/health
```

- Rodar smoke test do domínio (ex.: consulta de saldo, um PIX de teste, login).
- Confirmar que métricas de erro/latência voltaram ao normal antes de comunicar a conclusão.

### 5. Pós-incidente

- Documentar causa raiz do deploy defeituoso e plano de correção.
- Revisar o processo (testes, canary, validação em staging) para evitar reincidência. Ver [deploy sem downtime](../deploy-sem-downtime.md).
