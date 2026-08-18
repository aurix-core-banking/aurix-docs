# Runbook: Deploy em Producao - Aurix Platform

## Visao Geral

Procedimento completo de deploy em producao da plataforma Aurix, incluindo pre-requisitos, checklist de seguranca, deploy e rollback.

**Responsavel**: Squad Platform / DevOps
**Slack**: #aurix-deployments
**Posicao na fila**: Apenas 1 deploy por vez em producao

---

## Pre-Requisitos

### Ambiente
- [ ] Pipeline CI verde (todas as branches)
- [ ] Tests E2E passando em staging
- [ ] Review de seguranca aprovado (se aplicavel)
- [ ] Performance test OK (latencia p99 < 500ms)
- [ ] Approvals necessarios obtidos

### Infraestrutura
- [ ] Cluster EKS/AKS/GKE saudavel
- [ ] RDS/Azure SQL disponivel e sem manutencao programada
- [ ] Redis operacional
- [ ] Kafka com todos os brokers UP
- [ ] ClickHouse e TimescaleDB operacionais
- [ ] Elasticsearch com espaco livre > 20%
- [ ] Certificados SSL validos (mais de 30 dias para vencimento)

### Equipe
- [ ] Squad responsavel notificado no Slack #aurix-deployments
- [ ] DBA notificado (se houver migracao de schema)
- [ ] SRE em stand-by
- [ ] Janela de deploy comunicada (evitar horario de pico 09h-11h)

---

## Checklist de Deploy

### 1. Pre-Deploy
```
[x] Codigo aprovado no PR
[x] Branch principal atualizada
[x] Build passando em staging
[x] Migracao de banco testada em staging
[x] Rollback testado em staging
```

### 2. Deploy

#### Passo 1: Notificar squad
```bash
# Postar no Slack #aurix-deployments
echo "Iniciando deploy versao X.Y.Z - Responsavel: @equipe"
```

#### Passo 2: Migracao de banco (se necessario)
```bash
# Conectar ao bastion
aws ssm start-session --target i-XXXXXX

# Executar migracao no bastion
psql -h aurix-postgres-prod.cluster-xxxx.sa-east-1.rds.amazonaws.com \
     -U aurix -d aurix_db \
     -f /migrations/V${VERSION}.sql

# Verificar migracao
psql -h aurix-postgres-prod.cluster-xxxx.sa-east-1.rds.amazonaws.com \
     -U aurix -d aurix_db \
     -c "SELECT version FROM schema_migrations ORDER BY version DESC LIMIT 5;"
```

#### Passo 3: Deploy via ArgoCD
```bash
# Atualizar tag da imagem
kubectl patch application aurix-svc-banking \
  -n argocd \
  -p '{"spec":{"source":{"targetRevision":"v'$VERSION'"}}}' \
  --type merge

# Ou via Helm
helm upgrade svc-banking ./charts/svc-banking \
  --namespace aurix \
  --set image.tag=$VERSION \
  --set replicas=5 \
  --wait \
  --timeout 600s
```

#### Passo 4: Verificar rollout
```bash
# Verificar pods
kubectl get pods -n aurix -l app=svc-banking -w

# Verificar health
curl -s http://localhost:8080/actuator/health | jq .

# Verificar logs
kubectl logs -n aurix -l app=svc-banking --tail=50
```

#### Passo 5: Smoke tests
```bash
# Executar smoke tests
./scripts/smoke-tests.sh prod

# Verificar endpoints criticos
curl -s https://api.aurix.com.br/actuator/health
curl -s https://api.aurix.com.br/api/v1/contas/saldo?contaId=test | jq .
```

### 3. Pos-Deploy
- [ ] Todos os pods rodando (ready = desired)
- [ ] Health checks passando
- [ ] Latencia p99 dentro do SLA
- [ ] Sem erros nos logs (ultimo 5 min)
- [ ] Alertas no Grafana estaveis
- [ ] Trafego redistribuido normalmente

---

## Rollback

### Rollback Imediato (primeiros 5 minutos)
```bash
# Reverter Helm
helm rollback svc-banking $PREVIOUS_REVISION --namespace aurix

# Ou reverter ArgoCD
kubectl patch application aurix-svc-banking \
  -n argocd \
  -p '{"spec":{"source":{"targetRevision":"'$PREVIOUS_VERSION'"}}}' \
  --type merge
```

### Rollback de Migracao de Banco
```bash
# SOB CONSULTA DO DBA - executar apenas apos validacao
psql -h aurix-postgres-prod.cluster-xxxx.sa-east-1.rds.amazonaws.com \
     -U aurix -d aurix_db \
     -f /migrations/rollback/V${VERSION}_rollback.sql

# Verificar integridade
psql -h aurix-postgres-prod.cluster-xxxx.sa-east-1.rds.amazonaws.com \
     -U aurix -d aurix_db \
     -c "SELECT table_name FROM information_schema.tables WHERE table_schema = 'public' ORDER BY table_name;"
```

### Criterios para Rollback Automatico
- Mais de 1% de requests retornando 5xx por 3 minutos
- Latencia p99 > 2s por 5 minutos
- Error rate > 5% em qualquer microservico
- Health check falhando por mais de 2 minutos

---

## Migracao de Schema (Banco Unico)

> **ATENCAO**: Todos os servicos compartilham o mesmo PostgreSQL (`aurix_db`). Migracoes devem ser backward-compatible.

### Regras
1. Nunca alterar colunas existentes diretamente - criar colunas novas e migrar
2. Nunca remover colunas sem periodo de transicao (minimo 2 releases)
3. Sempre testar migracao em staging primeiro
4. Migracoes devem ser idempotentes
5. Usar transacoes em todas as migracoes

```sql
-- Exemplo de migracao segura
BEGIN;
-- 1. Adicionar coluna nova
ALTER TABLE contas ADD COLUMN IF NOT EXISTS saldo_disponivel NUMERIC(15,2);
-- 2. Migrar dados
UPDATE contas SET saldo_disponivel = saldo WHERE saldo_disponivel IS NULL;
-- 3. Adicionar constraint (apos validacao)
ALTER TABLE contas ALTER COLUMN saldo_disponivel SET NOT NULL;
COMMIT;
```

---

## Contato de Emergencia

| Papel | Contato | Telefone |
|-------|---------|----------|
| SRE On-Call | PagerDuty | +55 11 9XXXX-XXXX |
| DBA On-Call | PagerDuty | +55 11 9XXXX-XXXX |
| Tech Lead | @equipe-slack | +55 11 9XXXX-XXXX |
| Gerencia | @gerencia-slack | +55 11 9XXXX-XXXX |
