# AURIX Cloud – Runbook e alertas

Documento de operacao para deploy e monitoramento centralizados do AURIX em modo SaaS (AURIX Cloud). Complementa a fase 8.4 do roadmap.

---

## Escopo

- Deploy centralizado dos servicos (gateway, core, pix, credit, bacen, provisioning, etc.) e do plano de controle (provisioning DB).
- Deploy dos frontends: aurix-admin e aurix-web (build estatico servido por Nginx, S3/CloudFront ou container); aurix-mobile via build nativo e lojas.
- Monitoramento e alertas para garantir disponibilidade e detectar falhas.
- Runbook com procedimentos comuns para a equipe de operacao.

---

## Deploy centralizado

- **Orquestracao**: Kubernetes (EKS, AKS, GKE ou cluster on-prem). Cada tenant em multi-tenant possui banco apartado; os pods dos servicos podem ser compartilhados (roteamento por tenant) ou namespace por tenant conforme perfil de implantacao.
- **Pipeline**: CI/CD (GitHub Actions, GitLab CI ou similar) builda imagens, roda testes e deploya em staging/producao. Migrations (Flyway/Liquibase) executadas antes ou durante o deploy dos servicos que usam banco.
- **Secrets**: credenciais de banco por tenant, API keys externas (BACEN, KYC, Stripe) em secret manager do cloud (AWS Secrets Manager, Azure Key Vault, etc.) ou em Kubernetes Secrets. Nao commitar em repo.
- **Config**: perfil de implantacao (tenancy, cloud, topology) e config por ambiente (URLs, feature flags) via ConfigMap ou variaveis de ambiente.

---

## Monitoramento

- **Health checks**: todos os servicos expoem `/health` ou `/actuator/health`. O orchestrator (K8s) usa liveness/readiness probes. Gateway ou load balancer pode fazer health check no endpoint unico.
- **Metricas**: Prometheus scrape dos servicos (Spring Boot Actuator com micrometer-prometheus). Metricas sugeridas:
  - Por servico: requisicoes por segundo, latencia (p50, p95, p99), erros 4xx/5xx.
  - Por tenant (quando aplicavel): contagem de requisicoes por tenant_id para billing e limite.
  - Infra: CPU, memoria, conexoes de banco, filas Kafka.
- **Logs**: agregacao em um único sistema (ELK, Loki, CloudWatch Logs, etc.). Incluir tenant_id e correlation_id nas requisicoes para rastreio. Retencao conforme LGPD e contrato.
- **Dashboard**: Grafana (ou equivalente) com paineis de disponibilidade, latencia, erros e uso por tenant. Alertas derivados das metricas.

---

## Alertas

Definir regras de alerta (Prometheus Alertmanager, PagerDuty, Opsgenie ou integracao com canal Slack/email):

- **Criticos (acordar / acao imediata)**  
  - Servico indisponivel (health down por mais de X minutos).  
  - Taxa de erro acima de Y% por 5 minutos.  
  - Banco de dados inacessivel.  
  - Disco ou conexoes de DB proximos do limite.

- **Altos (resolver no mesmo dia)**  
  - Latencia p95 acima de SLA por 15 minutos.  
  - Filas Kafka com atraso crescente.  
  - Uso de memoria/CPU acima de 80% de forma sustentada.

- **Informativos**  
  - Deploy concluido.  
  - Novo tenant provisionado.  
  - Limite de uso do tenant (billing) atingindo 80%.

Cada alerta deve referenciar um item do runbook (link ou nome do procedimento).

> Runbooks dedicados (queda de serviço, lentidão no banco, falha no Kafka, incidente de segurança, rollback de deploy): ver [runbooks/index.md](runbooks/index.md). Os itens R1–R6 abaixo complementam os cenários do AURIX Cloud (SaaS/multi-tenant).

---

## Runbook – Procedimentos comuns

### R1 – Servico nao sobe apos deploy

1. Verificar logs do pod (kubectl logs ou painel de logs).
2. Verificar eventos do pod (kubectl describe pod).
3. Verificar config e secrets (variaveis, conexao com banco e Kafka).
4. Se migracao falhou: verificar estado do Flyway/Liquibase no banco; corrigir e reexecutar ou rollback da migracao conforme procedimento de banco.
5. Rollback da imagem para versao anterior se necessario; comunicar time de desenvolvimento.

### R2 – Latencia alta ou timeout nas APIs

1. Verificar dashboard de latencia por servico e por endpoint.
2. Verificar carga do banco (conexoes, queries lentas, locks). Se multi-tenant, identificar tenant(s) com mais carga.
3. Verificar filas Kafka (atraso, consumer lag).
4. Escalar horizontalmente o servico afetado se a causa for carga; otimizar queries ou indices se a causa for banco.
5. Se limite de plano do tenant foi ultrapassado, aplicar rate limit ou notificar cliente (billing).

### R3 – Banco do tenant inacessivel

1. Verificar conectividade de rede (pod -> DB) e credenciais.
2. Verificar status da instancia do banco no cloud (RDS, Cloud SQL, etc.): disponibilidade, storage, reinicios.
3. Verificar se o tenant foi provisionado corretamente (provisioning service, tabela instituicoes e config do banco).
4. Restaurar de backup se corrupcao ou exclusao acidental; seguir procedimento de restore do cloud provider.

### R4 – Incidente de seguranca ou vazamento de dados

1. Conter: isolar tenant ou servico afetado se possivel; revogar tokens/chaves comprometidas.
2. Avaliar alcance (quais tenants, quais dados) e registrar evidencia para auditoria.
3. Acionar processo de resposta a incidentes e notificacao (LGPD, BACEN, contratos).
4. Pos-incidente: revisar acessos, logs e config; corrigir vulnerabilidade e atualizar runbook.

### R5 – Novo tenant a provisionar (manual ou complementar)

1. Cadastrar instituicao e config no modulo provisioning (API ou painel admin).
2. Executar provisioning (POST /api/provisioning/instituicoes/{id}/provisionar). Se provider real estiver configurado, banco e namespace serao criados.
3. Validar acesso do tenant (login, health com tenant_id).
4. Registrar no billing e enviar documentacao de acesso ao cliente (ver doc AURIX Cloud para cliente).

### R6 – Backup e restore

1. **Backup**: seguir politica de backup do cloud (RDS automated backups, snapshots, etc.) ou usar scripts do item 14.2: `infra/scripts/backup-postgres.sh` com retencao configurável. Para multi-tenant, cada banco de tenant deve estar incluido. Testar restore periodicamente (ex.: trimestral). Detalhes: [backup-restore.md](backup-restore.md).
2. **Restore**: escolher ponto no tempo conforme RTO; restaurar instancia ou banco; atualizar config de conexao se necessario; validar integridade dos dados e comunicar cliente se impacto. Script: `restore-postgres.sh <arquivo.sql> [database]`.

---

## Contatos e escalacao

- Definir lista de contatos para alertas criticos (dev on-call, infra, seguranca).
- Integrar alertas com canal de comunicacao (Slack, PagerDuty) e rotacao de plantao conforme politica da empresa.

---

## Referencia

- Roadmap fase 8.4: Operacao AURIX Cloud.
- [deployment-profile.md](../arquitetura/deployment-profile.md): tenancy, cloud, topologia.
- [multi-tenant.md](../arquitetura/multi-tenant.md): politica de banco por tenant.
