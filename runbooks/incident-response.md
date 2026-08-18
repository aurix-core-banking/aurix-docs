# Runbook: Resposta a Incidentes - Aurix Platform

## Visao Geral

Procedimentos padronizados para triagem, escalação e comunicacao durante incidentes na plataforma Aurix.

**SLA de Resposta**: 15 minutos (P1), 30 minutos (P2), 2 horas (P3)
**Canal principal**: Slack #aurix-incidents
**Ferramenta de incidentes**: PagerDuty

---

## Classificacao de Severidade

| Nivel | Descricao | Tempo de Resposta | Exemplos |
|-------|-----------|-------------------|----------|
| **P1 - Critico** | Servico indisponivel para todos os usuarios | 15 min | Cluster caiu, banco indisponivel, breach de seguranca |
| **P2 - Alto** | Funcionalidade critica degradada | 30 min | PIX lento, pagamentos falhando, erro > 5% |
| **P3 - Medio** | Funcionalidade nao-critica afetada | 2 horas | Dashboard com erro, relatorio atrasado |
| **P4 - Baixo** | Bug nao-urgente, melhoria | proximo sprint | UI com detalhe, log incorreto |

---

## Fluxo de Resposta

### 1. Deteccao e Alerta
```
Alerta automatico (Grafana/PagerDuty)
  → Notificacao no Slack #aurix-incidents
  → SRE On-Call acionado
```

### 2. Triagem (primeiros 15 minutos)
```bash
# 1. Confirmar o incidente
# Verificar se o alerta e real (evitar falsos positivos)
curl -s https://api.aurix.com.br/actuator/health | jq .

# 2. Classificar severidade
# Usar a tabela acima

# 3. Notificar no Slack
cat << EOF
🚨 INCIDENTE P${SEVERITY}

Servico afetado: ${SERVICE}
Impacto: ${DESCRIPTION}
Severidade: P${SEVERITY}
Responsavel: @${ONCALL}
Status: Investigando

Timeline:
- $(date +%H:%M) - Incidente detectado
EOF

# 4. Abrir canal de guerra (se P1/P2)
# Criar canal Slack: #aurix-incident-${TIMESTAMP}
```

### 3. Escalacao

#### Escalacao Automatica
| Tempo sem resolucao | Acao |
|---------------------|------|
| 15 min (P1) | Notificar Tech Lead |
| 30 min (P1) | Notificar Gerencia de TI |
| 1 hora (P1) | Notificar CTO |
| 30 min (P2) | Notificar Tech Lead |
| 2 horas (P3) | Notificar squad responsavel |

#### Escalacao Manual
```bash
# Escalacao para SRE senior
pagerduty escalate --incident INC-XXXX --user sre-senior

# Escalacao para gerencia
slack send --channel #aurix-leadership --message "Incidente P1 requer atencao da gerencia"
```

### 4. Diagnostico

#### Comandos de Diagnostico Rapida

```bash
# === KUBERNETES ===
# Pods com problema
kubectl get pods -n aurix --field-selector=status.phase!=Running
kubectl get pods -n aurix -o wide | grep -v Running

# Logs de erro
kubectl logs -n aurix -l app=$SERVICE --tail=100 | grep -i "error\|exception\|fatal"

# Eventos recentes
kubectl get events -n aurix --sort-by=.lastTimestamp | tail -20

# Uso de recursos
kubectl top pods -n aurix --sort-by=memory

# === BANCO DE DADOS ===
# Conexoes ativas
psql -h $DB_HOST -U aurix -d aurix_db \
  -c "SELECT count(*) as connexoes, state 
       FROM pg_stat_activity 
       GROUP BY state;"

# Queries lentas
psql -h $DB_HOST -U aurix -d aurix_db \
  -c "SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
       FROM pg_stat_activity
       WHERE state != 'idle' AND query NOT LIKE '%pg_stat_activity%'
       ORDER BY duration DESC
       LIMIT 10;"

# Locks
psql -h $DB_HOST -U aurix -d aurix_db \
  -c "SELECT blocked.pid AS blocked_pid,
              blocked.query AS blocked_query,
              blocking.pid AS blocking_pid,
              blocking.query AS blocking_query
       FROM pg_stat_activity AS blocked
       JOIN pg_locks AS bl ON bl.pid = blocked.pid
       JOIN pg_locks AS kl ON kl.locktype = bl.locktype
         AND kl.database IS NOT DISTINCT FROM bl.database
         AND kl.relation IS NOT DISTINCT FROM bl.relation
         AND kl.page IS NOT DISTINCT FROM bl.page
         AND kl.tuple IS NOT DISTINCT FROM bl.tuple
         AND kl.transactionid IS NOT DISTINCT FROM bl.transactionid
         AND kl.pid != bl.pid
       JOIN pg_stat_activity AS blocking ON blocking.pid = kl.pid
       WHERE NOT bl.granted;"

# === REDIS ===
# Status do Redis
redis-cli -h $REDIS_HOST INFO memory
redis-cli -h $REDIS_HOST INFO clients
redis-cli -h $REDIS_HOST INFO replication

# Chaves com TTL alto
redis-cli -h $REDIS_HOST --scan --pattern "*" | head -20 | xargs -I {} redis-cli -h $REDIS_HOST TTL {}

# === KAFKA ===
# Status dos brokers
kafka-broker-api-versions.sh --bootstrap-server $KAFKA_BOOTSTRAP

# Lag dos consumers
kafka-consumer-groups.sh --bootstrap-server $KAFKA_BOOTSTRAP --describe --all-groups

# Status dos topicos
kafka-topics.sh --bootstrap-server $KAFKA_BOOTSTRAP --describe --under-replicated-partitions

# === ELASTICSEARCH ===
# Status do cluster
curl -s "https://$ES_HOST:9200/_cluster/health?pretty"

# Indices com problema
curl -s "https://$ES_HOST:9200/_cat/indices?v&health=red"
```

### 5. Mitigacao

#### Acoes Imediatas (antes de encontrar a causa raiz)
```bash
# Reiniciar pod com problema
kubectl delete pod -n aurix $POD_NAME --grace-period=10

# Escalar pods se necessario
kubectl scale deployment -n aurix $DEPLOYMENT --replicas=10

# Ativar modo degradado (circuit breaker)
# Se servico X esta caido, habilitar fallback
kubectl set env -n aurix deployment/$SERVICE CIRCUIT_BREAKER_ENABLED=true

# Bloquear requests problematicos (rate limiting)
kubectl set env -n aurix deployment/$SERVICE RATE_LIMIT_ENABLED=true

# Reduzir trafego (canary rollback)
kubectl patch virtualservice -n istio-system $SERVICE \
  -p '{"spec":{"http":[{"route":[{"destination":{"host":"'$SERVICE'","subset":"stable"},"weight":100}]}]}}'
```

#### Mitigacao por Tipo de Problema

**Banco de dados lento:**
```bash
# Cancelar queries longas
psql -h $DB_HOST -U aurix -d aurix_db \
  -c "SELECT pg_terminate_backend(pid) 
       FROM pg_stat_activity 
       WHERE state = 'active' 
         AND query_start < now() - interval '5 minutes';"
```

**Redis com muita memoria:**
```bash
# Flush se necessario (ULTIMO RECURSO)
redis-cli -h $REDIS_HOST FLUSHDB

# Ou apenas chaves com TTL alto
redis-cli -h $REDIS_HOST --scan --pattern "temp:*" | xargs redis-cli -h $REDIS_HOST DEL
```

**Kafka com lag alto:**
```bash
# Aumentar partitions (se necessario, com cuidado)
kafka-topics.sh --bootstrap-server $KAFKA_BOOTSTRAP \
  --alter --topic $TOPIC --partitions 12
```

### 6. Comunicacao durante o incidente

#### Template de Status (a cada 30 minutos)

```
📋 STATUS DO INCIDENTE - $(date +%d/%m/%Y\ %H:%M)

Servico: ${SERVICE}
Severidade: P${SEVERITY}
Status: ${STATUS}

Timeline:
- HH:MM - Incidente detectado
- HH:MM - Investigacao iniciada
- HH:MM - Causa raiz identificada (quando aplicavel)
- HH:MM - Mitigacao aplicada

Proximo update em: 30 minutos
Responsavel: @${ONCALL}
```

#### Canais de Comunicacao
| Severidade | Canal | Frequencia |
|------------|-------|------------|
| P1 | Slack #aurix-incidents | A cada 30 min |
| P1 | E-mail lideranca | Apos 1 hora |
| P2 | Slack #aurix-incidents | A cada 1 hora |
| P3 | Slack #aurix-incidents | Apos resolucao |

### 7. Resolucao

```bash
# Verificar se servicos estao saudaveis
for SVC in svc-banking svc-payments svc-credit svc-customer svc-platform; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" "https://api.aurix.com.br/${SVC}/actuator/health")
  echo "${SVC}: HTTP ${STATUS}"
done

# Verificar latencia
curl -w "Tempo: %{time_total}s\n" -o /dev/null -s \
  https://api.aurix.com.br/actuator/health

# Verificar error rate
kubectl logs -n aurix -l app=$SERVICE --since=30m | grep -c "ERROR"
```

### 8. Post-Incidente

#### Timeline Documentada
```markdown
# Incidente $(date +%Y%m%d)-${SEVERITY}

## Resumo
- **Inicio**: HH:MM
- **Fim**: HH:MM
- **Duracao**: Xh Xm
- **Impacto**: Descrição
- **Servicos afetados**: Lista
- **Usuarios impactados**: Estimativa

## Timeline
| Hora | Evento |
|------|--------|
| HH:MM | Alerta disparado |
| HH:MM | Investigacao iniciada |
| HH:MM | Causa raiz identificada |
| HH:MM | Mitigacao aplicada |
| HH:MM | Servico restaurado |
| HH:MM | Incidente fechado |

## Causa Raiz
Descricao detalhada da causa raiz.

## Acoes Corretivas
- [ ] Acao 1 - Responsavel - Prazo
- [ ] Acao 2 - Responsavel - Prazo

## Aprendizados
- O que funcionou bem
- O que pode melhorar
```

#### Retro (ate 48h apos resolucao)
- Participantes: SRE, squad responsavel, gerencia
- Formato: 45 minutos
- Resultado: Documento de acoes corretivas com owners e prazos

---

## Templates Rapidos

### Para criar canal de incidente
```bash
slack-cli channels create aurix-incident-$(date +%Y%m%d-%H%M) \
  --topic "Incidente P${SEVERITY} - ${SERVICE}"
```

### Para page SRE senior
```bash
pagerduty trigger \
  --summary "Incidente P${SEVERITY}: ${SERVICE} - ${DESCRIPTION}" \
  --source "aurix-platform" \
  --severity "critical" \
  --pagerduty-service "aurix-sre"
```

---

## Contatos

| Papel | Nome | Slack | Telefone |
|-------|------|-------|----------|
| SRE On-Call | Roda | @sre-oncall | PagerDuty |
| SRE Senior | Equipe | @sre-senior | +55 11 9XXXX-XXXX |
| Tech Lead | Equipe | @tech-lead | +55 11 9XXXX-XXXX |
| Gerencia TI | Equipe | @gerencia-ti | +55 11 9XXXX-XXXX |
| CTO | Equipe | @cto | +55 11 9XXXX-XXXX |
| DBA | Equipe | @dba | +55 11 9XXXX-XXXX |
