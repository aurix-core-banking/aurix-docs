# Runbook: Disaster Recovery - Aurix Platform

## Visao Geral

Procedimento de Disaster Recovery (DR) para a plataforma Aurix.

- **RTO (Recovery Time Objective)**: 4 horas
- **RPO (Recovery Point Objective)**: 1 hora
- **Estrategia**: Pilot Light (ambiente secundario com recursos minimos prontos)

**Responsavel**: SRE / Squad Platform
**Slack**: #aurix-incidents
**Aprovacao necessaria**: Gerencia de TI + CTO

---

## Cenarios de DR

| Cenario | Severidade | Tempo estimado |
|---------|------------|----------------|
| Falha de AZ inteira | Alta | 1-2h |
| Falha de regiao inteira | Critica | 2-4h |
| Corrupcao de dados | Alta | 1-3h |
| Ataque cibernetico | Critica | 2-4h |

---

## Pre-Requisitos para DR

### Ambiente Secundario
- [ ] Cluster EKS/AKS/GKE standby na regiao secundaria
- [ ] RDS/Azure SQL com read replica em regiao secundaria
- [ ] Redis com replicacao cross-region
- [ ] S3/Storage com replicacao cross-region (ou cross-account)
- [ ] Kafka MirrorMaker configurado
- [ ] DNS failover configurado (Route53/Azure DNS/Cloud DNS)
- [ ] Certificados TLS validos na regiao secundaria

### Dados
- [ ] Ultimo backup verificado (cronograma: a cada 1 hora)
- [ ] WAL archiving ativo e verificado
- [ ] Snapshots Elasticsearch atualizados
- [ ] Backups ClickHouse disponiveis
- [ ] Offsets Kafka replicados

---

## Procedimento de DR

### Fase 1: Avaliacao (15 minutos)

#### 1.1 Confirmar incidente
```bash
# Verificar status da regiao primaria
aws cloudwatch get-metric-statistics \
  --namespace AWS/ELB \
  --metric-name HealthyHostCount \
  --dimensions Name=LoadBalancer,Value=aurix-alb-prod \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average
```

#### 1.2 Notificar stakeholders
```bash
# Postar no Slack #aurix-incidents
echo "INCIDENTE DR: Regiao primaria indisponivel. Iniciando avaliacao.
Status: Em andamento
Responsavel: @sre-oncall
Proxima atualizacao: 15 minutos"
```

#### 1.3 Decidir sobre DR
- Se falha de AZ: migrar para outra AZ (failover interno)
- Se falha de regiao: ativar DR cross-region
- Se corrupcao: ativar DR + restore point-in-time

### Fase 2: Ativacao do Ambiente Secundario (30-60 minutos)

#### 2.1 Promover read replica para escrita
```bash
# AWS Aurora
aws rds failover-db-cluster \
  --db-cluster-identifier aurix-postgres-dr \
  --target-db-instance-identifier aurix-postgres-dr-writer

# Aguardar propagacao
aws rds describe-db-clusters \
  --db-cluster-identifier aurix-postgres-dr \
  --query 'DBClusters[0].Status'

# Azure SQL
az sql db failover \
  --name aurix_db \
  --server aurix-sql-dr \
  --resource-group aurix-dr

# GCP Cloud SQL
gcloud sql instances failover aurix-postgres-dr \
  --region=southamerica-east1
```

#### 2.2 Atualizar DNS para apontar para regiao secundaria
```bash
# Route53
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.aurix.com.br",
        "Type": "A",
        "SetIdentifier": "primary",
        "Failover": "PRIMARY",
        "TTL": 60,
        "ResourceRecords": [{"Value": "DR_LOAD_BALANCER_IP"}]
      }
    }]
  }'
```

#### 2.3 Ativar cluster Kubernetes DR
```bash
# AWS - cambiar nodegroup para on-demand
aws eks update-nodegroup-config \
  --cluster-name aurix-eks-dr \
  --nodegroup-name aurix-main-dr \
  --scaling-config minSize=5,maxSize=20,desiredSize=8

# Azure
az aks nodepool scale \
  --resource-group aurix-dr \
  --cluster-name aurix-aks-dr \
  --name system \
  --node-count 5

# GCP
gcloud container clusters update aurix-gke-dr \
  --region southamerica-east1 \
  --node-pool aurix-main-dr \
  --num-nodes 5
```

#### 2.4 Configurar Kafka DR
```bash
# Verificar MirrorMaker esta sincronizando
kafka-consumer-groups.sh \
  --bootstrap-server kafka-dr:9092 \
  --list

# Se necessario, recriar topicos
cat topics-backup.json | kafka-topics.sh \
  --bootstrap-server kafka-dr:9092 \
  --create --if-not-exists \
  --replication-factor 3
```

#### 2.5 Restaurar Elasticsearch
```bash
# Verificar snapshots disponiveis
curl -s "https://es-dr:9200/_snapshot/aurix-backups/_all" | jq .

# Restaurar ultimo snapshot
curl -X POST "https://es-dr:9200/_snapshot/aurix-backups/snapshot_latest/_restore" \
  -H "Content-Type: application/json" \
  -d '{"ignore_unavailable": true}'
```

### Fase 3: Validacao (30 minutos)

#### 3.1 Health checks
```bash
# Todos os servicos
for SVC in svc-banking svc-payments svc-credit svc-customer svc-platform; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" "https://api-dr.aurix.com.br/${SVC}/actuator/health")
  echo "${SVC}: ${STATUS}"
done
```

#### 3.2 Smoke tests
```bash
./scripts/smoke-tests.sh dr
```

#### 3.3 Verificar integridade de dados
```bash
# TimescaleDB
psql -h dr-db.aurix.com.br -U aurix -d aurix_timeseries \
  -c "SELECT hypertable_name, 
             pg_size_pretty(hypertable_size(format('%I', hypertable_name)::regclass)) as tamanho
      FROM timescaledb_information.hypertables;"

# ClickHouse
clickhouse-client --host dr-ch.aurix.com.br --query "
  SELECT database, table, formatReadableSize(sum(bytes_on_disk)) as tamanho
  FROM system.parts
  WHERE active
  GROUP BY database, table
  ORDER BY sum(bytes_on_disk) DESC
  LIMIT 20"
```

#### 3.4 Verificar latencia
```bash
# Teste de latencia
curl -w "Tempo total: %{time_total}s\n" -o /dev/null -s \
  https://api-dr.aurix.com.br/svc-banking/actuator/health
# Esperado: < 500ms
```

### Fase 4: Comunicacao e Monitoramento

#### 4.1 Atualizar status
```bash
echo "DR ATIVADO com sucesso.
Ambiente secundario operacional.
RTO alcancado: $(date -u -H:%M:%S)
Proxima acao: Monitoramento continuo"
```

#### 4.2 Monitoramento continuo
```bash
# Configurar alertas no Grafana
# - CPU > 80%
# - Memoria > 85%
# - Erro rate > 1%
# - Latencia p99 > 1s
# - Pod restarts > 3 em 5 min
```

---

## Rollback (Retorno a Regiao Primaria)

### Quando retomar
- Regiao primaria saudavel por 30 minutos continuos
- Todos os servicos respondendo
- Dados sincronizados (verificar replicacao)

### Procedimento
1. **Verificar integridade** da regiao primaria
2. **Sincronizar dados** (trigger manual de replicacao se necessario)
3. **Migrar escrita** para regiao primaria gradualmente (canary 10% -> 50% -> 100%)
4. **Atualizar DNS** para apontar para regiao primaria
5. **Monitorar** por 2 horas apos retorno
6. **Documentar** timeline do incidente

---

## RPO - Recuperacao de Dados

### Se perda de dados < 1 hora
```bash
# Restore point-in-time do Aurora
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier aurix-postgres-prod \
  --target-db-instance-identifier aurix-postgres-restore \
  --restore-time "2024-01-15T10:30:00Z" \
  --use-latest-restorable-time
```

### Se perda de dados > 1 hora
1. Verificar backup mais recente no S3
2. Restaurar para instancia temporaria
3. Validar dados com equipe de negocio
4. Promover instancia restaurada

---

## Post-Incidente

### Checklist
- [ ] Timeline documentada
- [ ] Root cause identificada
- [ ] Acoes corretivas definidas
- [ ] Report de incidente gerado
- [ ] Retro agendada (ate 48h apos resolucao)
- [ ] Atualizacao de runbook se necessario
- [ ] Backup adicional realizado

---

## Contato de Emergencia

| Papel | Contato | Disponibilidade |
|-------|---------|-----------------|
| SRE On-Call | PagerDuty #aurix-sre | 24/7 |
| Tech Lead | @tech-lead-slack | Horario comercial + on-call |
| Gerencia de TI | @gerencia-slack | Emergencia |
| CTO | @cto-slack | Apenas decisoes de negocio |
| DBA | @dba-slack | Migracao de dados |
