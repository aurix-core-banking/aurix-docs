# 14.1 Alta disponibilidade

Replicas dos servicos criticos e load balancer para resiliencia e SLA (item 14.1 do roadmap).

## Componentes

### Load balancer

- **Nginx**: configurado em `infra/nginx/nginx.conf` com upstreams para gateway, core, pix, credit, treasury, security, compliance, analytics, audit, organization. Usado na pilha Docker Compose.
- **Producao**: em cloud (AWS ALB, Azure Load Balancer, GCP Load Balancer) ou ingress Kubernetes (ex.: NGINX Ingress Controller) com health checks no path `/health` ou `/actuator/health` de cada servico.

### Replicas dos servicos criticos

- **Kubernetes**: o deployment do core ja define `replicas: 3` em `infra/kubernetes/aurix-core-deployment.yaml`. Replicar o padrao para gateway e pix (copiar o deployment e ajustar nome/imagem/porta).
- **Docker Compose**: por padrao cada servico roda uma unica instancia. Para HA em Compose:
  - Escalar: `docker-compose up -d --scale aurix-gateway=2 --scale aurix-core=2 --scale aurix-pix=2`.
  - O Nginx faz balanceamento entre os containers pelo nome do servico (ex.: `aurix-gateway:8080` resolve para todas as tarefas do servico no mesmo network).
- **Banco de dados**: PostgreSQL em HA exige configuracao externa (streaming replication, Patroni, ou servico gerenciado RDS/Cloud SQL com multi-AZ). Fora do escopo dos scripts atuais; documentar no runbook quando usar RDS/Cloud SQL.

### Health checks

- Todos os servicos Spring Boot expoem `/actuator/health`. O load balancer ou orquestrador deve usar esse endpoint para liveness/readiness.
- Nginx: configurar `proxy_next_upstream error timeout http_502 http_503` para retentar em outro backend em caso de falha.

## Checklist de implementacao

- [ ] Nginx ou ingress com health check e retry configurados.
- [ ] Gateway, core e pix com pelo menos 2 replicas em staging/producao (K8s ou Compose scale).
- [ ] PostgreSQL: usar instancia gerenciada com multi-AZ ou replicacao quando SLA exigir.
- [ ] Redis: em producao considerar Redis Cluster ou ElastiCache com failover.

## Referencias

- Roadmap e status: [../roadmap.md](../roadmap.md)
- Runbook: [aurix-cloud-runbook.md](aurix-cloud-runbook.md)
- Kubernetes: `infra/kubernetes/`
- Nginx: `infra/nginx/nginx.conf`
