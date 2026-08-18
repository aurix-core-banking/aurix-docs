# Runbook: Backup e Recuperacao - Aurix Platform

## Visao Geral

Procedimentos de backup, restauracao point-in-time e validacao para todos os servicos de dados do Aurix.

**Cronograma de Backups**:

| Servico | Frequencia | Retencao | Metodo |
|---------|-----------|----------|--------|
| PostgreSQL (Aurora) | Diario (03:00 UTC) | 30 dias | Snapshots automaticos + WAL |
| TimescaleDB | Diario (03:30 UTC) | 30 dias | pg_dump custom + WAL archiving |
| ClickHouse | Diario (04:00 UTC) | 30 dias | clickhouse-backup + S3 |
| Elasticsearch | Diario (04:30 UTC) | 30 dias | Snapshot API |
| Kafka | Diario (05:00 UTC) | 30 dias | Topic configs + offsets |

**Armazenamento**: S3/MinIO (`aurix-backups-dev`), com lifecycle para Glacier/Archive apos 90 dias.

---

## 1. PostgreSQL (Aurora) - Backup

### Backup Automatico
O Aurora mantem backups automaticos com retencao configurada para 30 dias.

```bash
# Listar snapshots automaticos
aws rds describe-db-cluster-snapshots \
  --db-cluster-identifier aurix-postgres-prod \
  --query 'DBClusterSnapshots[?starts_with(DBClusterSnapshotIdentifier, `rds:aurix-postgres-prod`)][:5].{ID:DBClusterSnapshotIdentifier,Created:SnapshotCreateTime,Size:AllocatedStorage}' \
  --output table

# Criar snapshot manual (pre-deploy)
aws rds create-db-cluster-snapshot \
  --db-cluster-identifier aurix-postgres-prod \
  --db-cluster-snapshot-identifier "aurix-pre-deploy-$(date +%Y%m%d-%H%M)"
```

### Restore Point-in-Time
```bash
# Restaurar para ponto especifico
aws rds restore-db-cluster-to-point-in-time \
  --db-cluster-identifier aurix-postgres-prod \
  --target-db-cluster-identifier aurix-postgres-restore \
  --restore-time "2024-01-15T10:30:00Z" \
  --use-latest-restorable-time

# Aguardar restauracao
aws rds wait db-cluster-available \
  --db-cluster-identifier aurix-postgres-restore

# Verificar dados restaurados
psql -h aurix-postgres-restore.cluster-xxxx.sa-east-1.rds.amazonaws.com \
  -U aurix -d aurix_db \
  -c "SELECT count(*) FROM contas WHERE created_at > '2024-01-15';"
```

---

## 2. TimescaleDB - Backup e Restore

### Backup
```bash
# Executar backup via script
cd aurix-data-platform/backup/
./backup-timescaledb.sh

# Backup manual
pg_dump \
  --host=localhost --port=5433 --username=aurix \
  --dbname=aurix_timeseries \
  --format=custom --compress=6 \
  --file=/backups/timescaledb/manual_$(date +%Y%m%d).dump
```

### Restore Point-in-Time
```bash
# Restore completo
pg_restore \
  --host=localhost --port=5433 --username=aurix \
  --dbname=aurix_timeseries \
  --clean --if-exists --no-owner \
  --verbose \
  /backups/timescaledb/aurix_timeseries_20240115_030000.dump

# Restore de tabelas especificas
pg_restore \
  --host=localhost --port=5433 --username=aurix \
  --dbname=aurix_timeseries \
  --data-only --table=transacoes_agregadas \
  /backups/timescaledb/aurix_timeseries_20240115_030000.dump
```

### Verificacao de Backup
```bash
# Listar conteudo do dump (sem restaurar)
pg_restore --list /backups/timescaledb/aurix_timeseries_20240115_030000.dump

# Verificar integridade
pg_restore --verbose --list /backups/timescaledb/aurix_timeseries_20240115_030000.dump > /dev/null 2>&1
echo "Status: $?"  # 0 = OK
```

---

## 3. ClickHouse - Backup e Restore

### Backup
```bash
# Executar backup via script
cd aurix-data-platform/backup/
./backup-clickhouse.sh

# Verificar backup
ls -la local-backups/clickhouse/
```

### Restore
```bash
# Via clickhouse-backup (se disponivel)
clickhouse-backup restore clickhouse_full_20240115_030000

# Via TSV manual
for TSV_FILE in local-backups/clickhouse/clickhouse_full_20240115_030000/*.tsv; do
  TABLE=$(basename "${TSV_FILE}" .tsv)
  echo "Importando ${TABLE}..."
  clickhouse-client --host localhost --database aurix_analytics \
    --query "INSERT INTO ${TABLE} FORMAT TabSeparatedWithNamesAndTypes" \
    < "${TSV_FILE}"
done
```

---

## 4. Elasticsearch - Backup e Restore

### Backup
```bash
# Executar backup via script
cd aurix-data-platform/backup/
./backup-elasticsearch.sh

# Backup manual
curl -X PUT "https://localhost:9200/_snapshot/aurix-backups/snapshot_manual?wait_for_completion" \
  -H "Content-Type: application/json" \
  -d '{
    "indices": "*",
    "ignore_unavailable": true,
    "include_global_state": false
  }'
```

### Restore
```bash
# Listar snapshots
curl -s "https://localhost:9200/_snapshot/aurix-backups/_all" | jq '.snapshots[].snapshot'

# Restaurar snapshot especifico
curl -X POST "https://localhost:9200/_snapshot/aurix-backups/snapshot_20240115/_restore" \
  -H "Content-Type: application/json" \
  -d '{
    "indices": "transacoes,transacoes-agregadas",
    "ignore_unavailable": true
  }'

# Fechar e reabrir indices apos restore
curl -X POST "https://localhost:9200/transacoes/_close"
# ... restore ...
curl -X POST "https://localhost:9200/transacoes/_open"
```

---

## 5. Kafka - Backup e Restore

### Backup
```bash
# Executar backup via script
cd aurix-data-platform/backup/
./backup-kafka.sh
```

### Restore de Topicos
```bash
# Recriar topicos a partir do backup de configuracao
for CONFIG_FILE in local-backups/kafka/configs_20240115/*.describe; do
  TOPIC=$(basename "${CONFIG_FILE}" .describe)
  PARTITIONS=$(grep "PartitionCount:" "${CONFIG_FILE}" | awk '{print $2}')
  REPLICAS=$(grep "ReplicationFactor:" "${CONFIG_FILE}" | awk '{print $2}')
  
  kafka-topics.sh --bootstrap-server $KAFKA_BOOTSTRAP \
    --create --if-not-exists \
    --topic "${TOPIC}" \
    --partitions "${PARTITIONS}" \
    --replication-factor "${REPLICAS}"
done

# Restaurar configs
for CONFIG_FILE in local-backups/kafka/configs_20240115/*.configs; do
  TOPIC=$(basename "${CONFIG_FILE}" .configs)
  kafka-configs.sh --bootstrap-server $KAFKA_BOOTSTRAP \
    --entity-type topics --entity-name "${TOPIC}" \
    --alter --add-config $(grep -o '[a-z.]*=[^,]*' "${CONFIG_FILE}" | tr '\n' ',')
done
```

---

## Validacao de Backups

### Checklist de Validacao (executar semanalmente)
- [ ] Listar todos os backups da ultima semana
- [ ] Restaurar 1 backup de cada servico em ambiente de staging
- [ ] Verificar integridade dos dados restaurados
- [ ] Confirmar que backups existem no S3/MinIO
- [ ] Verificar lifecycle de retencao esta funcionando
- [ ] Testar restore point-in-time do Aurora
- [ ] Validar que clickhouse-backup cria snapshots validos

### Script de Validacao Automatica
```bash
#!/usr/bin/env bash
# validacao-backups.sh - Executar toda segunda-feira
set -euo pipefail

echo "=== Validacao Semanal de Backups ==="

# 1. Verificar backups no S3
echo "--- Backups PostgreSQL (Aurora) ---"
aws rds describe-db-cluster-snapshots \
  --db-cluster-identifier aurix-postgres-prod \
  --query 'length(DBClusterSnapshots)'

echo "--- Backups TimescaleDB no S3 ---"
aws s3 ls s3://aurix-backups-dev/timescaledb/ --recursive | wc -l

echo "--- Backups ClickHouse no S3 ---"
aws s3 ls s3://aurix-backups-dev/clickhouse/ --recursive | wc -l

echo "--- Backups Kafka no S3 ---"
aws s3 ls s3://aurix-backups-dev/kafka/ --recursive | wc -l

# 2. Verificar integridade do ultimo backup TimescaleDB
LAST_DUMP=$(aws s3 ls s3://aurix-backups-dev/timescaledb/ --recursive | sort | tail -1 | awk '{print $4}')
aws s3 cp "s3://aurix-backups-dev/timescaledb/${LAST_DUMP}" /tmp/validate.dump
pg_restore --list /tmp/validate.dump > /dev/null 2>&1
if [[ $? -eq 0 ]]; then
    echo "OK: Ultimo backup TimescaleDB integro"
else
    echo "ERRO: Backup TimescaleDB corrompido!"
fi
rm -f /tmp/validate.dump

# 3. Verificar snapshot Elasticsearch
ES_SNAPSHOTS=$(curl -s "https://$ES_HOST:9200/_snapshot/_all" | python3 -c "
import sys, json
data = json.load(sys.stdin)
total = sum(len(r.get('snapshots',[])) for r in data.get('repositories',{}).values())
print(total)
" 2>/dev/null || echo "0")
echo "--- Snapshots Elasticsearch: ${ES_SNAPSHOTS} ---"

echo "=== Validacao concluida ==="
```

---

## Retencao e Limpeza

### Politica de Retencao
| Camada | Dias | Acao |
|--------|------|------|
| Local | 7 | Manter backups locais |
| S3 Standard | 30 | Acesso rapido |
| S3 Standard-IA | 30-90 | Acesso infrequente |
| S3 Glacier | 90-365 | Arquivamento |
| Exclusao | >365 | Deletar |

### Limpeza Manual
```bash
# Forcar limpeza de backups antigos
cd aurix-data-platform/backup/
RETENTION_DAYS=30 ./backup-clickhouse.sh  # Executa limpeza automatica
RETENTION_DAYS=30 ./backup-timescaledb.sh
RETENTION_DAYS=30 ./backup-kafka.sh
```

---

## Contato

| Papel | Contato |
|-------|---------|
| DBA On-Call | PagerDuty #aurix-dba |
| SRE On-Call | PagerDuty #aurix-sre |
| Squad Data | Slack #aurix-data-platform |
