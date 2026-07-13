# Infrastructure as Code — Final Design

## Overview

Complete the infrastructure-as-code layer for the Aureus Platform: Terraform multi-cloud provisioning + GitOps (ArgoCD) + Kubernetes Helm deployment for all 37 microservices, with observability, security, and disaster recovery.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    GitHub Actions (CI)                   │
│   Build → Test → Docker buildx → Push to ECR/ACR/GCR   │
└──────────────────────┬──────────────────────────────────┘
                       │ Image tag update
                       ▼
┌─────────────────────────────────────────────────────────┐
│              ArgoCD Image Updater                        │
│              ArgoCD (GitOps CD)                          │
│         App-of-Apps Pattern + ApplicationSets            │
├─────────────────────────────────────────────────────────┤
│   Umbrella Helm Chart → 37 sub-charts (microservices)   │
├─────────────────────────────────────────────────────────┤
│   Infra base: Istio, ESO, Vault, cert-manager,          │
│   Prometheus, Grafana, ELK, Velero                      │
├─────────────────────────────────────────────────────────┤
│   EKS / AKS / GKE (provisionado por Terraform)          │
└─────────────────────────────────────────────────────────┘
```

## 1. Repository Structure (Monorepo)

```
infrastructure/
├── terraform/
│   ├── main.tf                     # Root module (providers + module calls)
│   ├── variables.tf                # All variables
│   ├── backend.tf                  # S3 (AWS) / Storage Account (Azure) / GCS (GCP) state
│   └── modules/
│       ├── aws/{vpc,eks,rds,redis,alb,cloudfront,s3,cloudwatch,iam,security}
│       ├── azure/{aks,postgresql,redis,resource-group}
│       └── gcp/{gke,cloud-sql,memorystore,vpc}
├── kubernetes/
│   ├── argocd/
│   │   ├── bootstrap/infra-components.yaml
│   │   ├── applicationsets/microservices.yaml
│   │   └── projects/{infra,platform}.yaml
│   ├── charts/
│   │   ├── aureus-core/
│   │   ├── aureus-pix/
│   │   ├── aureus-bacen/
│   │   ├── aureus-onboarding/
│   │   ├── aureus-cartoes/
│   │   ├── aureus-consignado/
│   │   ├── aureus-financiamento/
│   │   ├── aureus-cambio/
│   │   ├── aureus-seguros/
│   │   ├── aureus-investimento/
│   │   ├── aureus-salario/
│   │   ├── aureus-poupanca/
│   │   ├── aureus-credit/
│   │   ├── aureus-catalog/
│   │   ├── aureus-analytics/
│   │   ├── aureus-security/
│   │   ├── aureus-audit/
│   │   ├── aureus-openfinance/
│   │   ├── aureus-gateway/
│   │   ├── aureus-accounting/
│   │   ├── aureus-billing/
│   │   ├── aureus-compliance/
│   │   ├── aureus-webhooks/
│   │   ├── aureus-organization/
│   │   ├── aureus-provisioning/
│   │   ├── aureus-treasury/
│   │   ├── aureus-tax/
│   │   ├── aureus-cost/
│   │   ├── aureus-budget/
│   │   ├── aureus-settlement/
│   │   ├── aureus-pricing/
│   │   ├── aureus-baas/
│   │   ├── aureus-internet-banking/
│   │   ├── aureus-mobile-banking/
│   │   ├── aureus-ai/
│   │   ├── aureus-controller/
│   │   └── aureus-shared/
│   │   └── umbrella/              # Umbrella chart por ambiente
│   ├── base/                       # Cluster bootstrap manifests
│   └── overlays/                   # Values per environment + cloud
│       ├── dev/
│       ├── staging/
│       └── prod/
├── docker-compose.yml
└── .env.example
```

## 2. Terraform — State + Module Audit

### Backend State

- **AWS**: S3 bucket `aureus-terraform-state-{env}` + DynamoDB table `aureus-terraform-locks`
- **Azure**: Storage Account container `terraform-state` + blob lease locking
- **GCP**: GCS bucket `aureus-terraform-state-{env}` + object versioning

### Module Audit

Each module must expose outputs consumed by `main.tf`:
- `aws_vpc`: `vpc_id`, `private_subnet_ids`, `public_subnet_ids`, `database_subnet_ids`
- `aws_eks`: `cluster_name`, `cluster_endpoint`, `cluster_ca_certificate`
- `aws_rds`: `cluster_endpoint`, `cluster_port`, `cluster_arn`
- `aws_redis`: `cluster_endpoint`, `cluster_port`
- `aws_alb`: `dns_name`, `zone_id`
- `aws_cloudfront`: `domain_name`, `distribution_id`
- `aws_s3`: `bucket_id`, `bucket_arn`
- `aws_cloudwatch`: `log_group_arn`
- `aws_iam`: (roles/ policies)
- `aws_security`: `security_group_ids`

Azure/GCP modules follow same pattern.

## 3. GitOps Bootstrap (ArgoCD)

### Bootstrap Order

1. Terraform provisions K8s clusters + outputs kubeconfig
2. **Phase 0**: Install ArgoCD via `helm_release` in Terraform (or manual `kubectl apply`)
3. **Phase 1**: ArgoCD syncs `argocd/bootstrap/infra-components.yaml` (or ApplicationSet)
4. **Phase 2**: Infra app deploys: Istio, External Secrets Operator, cert-manager, Velero, Prometheus stack, ELK
5. **Phase 3**: Microservices ApplicationSet creates 37 Application resources

### ArgoCD ApplicationSet Template

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices
spec:
  generators:
  - git:
      repoURL: https://github.com/anomalyco/aureus-platform
      revision: HEAD
      directories:
      - path: infrastructure/kubernetes/charts/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: platform
      source:
        repoURL: https://github.com/anomalyco/aureus-platform
        path: '{{path}}'
        helm:
          valueFiles:
          - values.yaml
          - values-{{env}}.yaml
      destination:
        server: https://kubernetes.default.svc
        namespace: aureus-platform
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

## 4. Helm Chart Structure

### Individual Chart (`charts/<service>/`)

```
Chart.yaml
values.yaml              # Dev defaults
values-staging.yaml      # Staging overrides
values-prod.yaml         # Prod overrides
values-aws.yaml          # Cloud-specific overrides
values-azure.yaml
values-gcp.yaml
templates/
  deployment.yaml        # Deployment + env vars + resources + probes
  service.yaml           # ClusterIP service
  hpa.yaml               # HorizontalPodAutoscaler
  pdb.yaml               # PodDisruptionBudget
  serviceaccount.yaml    # Per-service SA
  _helpers.tpl           # Common templates
```

### Umbrella Chart (`charts/umbrella/`)

```yaml
# Chart.yaml
dependencies:
  - name: aureus-core
    version: "0.1.0"
    repository: "file://../aureus-core"
  - name: aureus-pix
    version: "0.1.0"
    repository: "file://../aureus-pix"
  # ... 35 more
```

Centralizes shared configuration:
- Database URLs (from cloud provider outputs → Vault → ESO → K8s Secret)
- Redis host/port
- Kafka bootstrap servers
- Istio VirtualService + DestinationRule definitions

## 5. CI/CD — GitHub Actions

### Pipeline per Microservice

```yaml
name: Build aureus-core

on:
  push:
    branches: [main, develop]
    paths: ["backend/aureus-core/**"]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '25', distribution: 'temurin' }
      - run: mvn -pl aureus-core -am compile test
      - uses: docker/setup-buildx-action@v3
      - run: |
          docker buildx build \
            -t ${{ env.REGISTRY }}/aureus-core:${{ github.sha }} \
            --platform linux/amd64,linux/arm64 \
            --push \
            -f backend/aureus-core/Dockerfile backend/
```

### Image Registry Strategy

| Cloud | Registry | Auth |
|---|---|---|
| AWS | ECR (Elastic Container Registry) | IRSA + OIDC |
| Azure | ACR (Azure Container Registry) | Workload Identity |
| GCP | GCR / Artifact Registry | Workload Identity Federation |
| Fallback | Docker Hub | Token via GitHub Secrets |

### ArgoCD Image Updater

Configured per Application to watch registry and auto-update image tag. No manual version bumps.

## 6. Observability

### Metrics — kube-prometheus-stack

- Prometheus Operator (CRDs: ServiceMonitor, PodMonitor, PrometheusRule)
- Grafana with pre-configured dashboards:
  - K8s cluster health (nodes, pods, namespaces)
  - Microservice-specific (JVM metrics, request rate, error rate, latency)
  - Business metrics (onboarding funnel, PIX TPS, credit applications)
- AlertManager rules:
  - Pod crash loop / pending / OOMKill
  - High error rate (>5% 5xx per service)
  - P99 latency > 2s
  - Disk space > 80%

### Logs — ECK (Elastic Cloud on K8s)

- Elasticsearch cluster (3 nodes, production)
- Filebeat daemonset shipping container logs
- Logstash for enrichment (tenant ID, service name, cloud provider)
- Kibana for visualization + alerting
- Retention: 7 days hot, 30 days warm, 90 days cold

## 7. Security

### Istio Service Mesh

- **mTLS**: `PeerAuthentication` STRICT mode in aureus-platform namespace
- **Ingress**: Istio Gateway as single entry point, TLS termination via cert-manager
- **Authorization**: `AuthorizationPolicy` allowlisting service-to-service calls
- **Canary**: `VirtualService` weighted routing for gradual rollouts

### Secrets Management

- **Source of Truth**: Vault (HashiCorp) with auto-unseal via KMS
- **Sync to K8s**: External Secrets Operator reads from Vault and creates K8s Secrets
- **Rotation**: Vault lease TTL 30 days; ESO auto-refreshes before expiry

### Network Policies

Default deny-all ingress/egress per namespace, with explicit allow rules for:
- Istio ingress gateway → microservices
- Microservice A → microservice B (known dependencies)
- Prometheus → all pods (port 8081 /actuator)
- Elasticsearch → specific services that log

### Pod Security

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

### Image Security

- Trivy scan in CI pipeline (fail on CRITICAL/HIGH)
- Distroless base images where possible
- Image signing via cosign (optional phase 2)

## 8. Backup / Disaster Recovery

### Velero

- Backup: K8s resources + PV snapshots (CSI) + restic for file-level
- Schedule: Daily at 02:00 UTC, retained 30 days
- Storage: S3 (AWS) / Blob (Azure) / GCS (GCP)

### Database Snapshots

| Engine | Strategy | RPO |
|---|---|---|
| RDS Aurora | Automated daily + continuous WAL | 5 min |
| Azure PostgreSQL | Geo-redundant backup | 1 hour |
| GCP Cloud SQL | Automated daily + PITR | 5 min |
| Redis | RDB snapshot via Velero | 24 hours |

### DR Plan (Active-Passive)

1. Secondary region has warm EKS cluster (1 node minimum)
2. Velero restores latest backup to secondary cluster
3. RDS read-replica promoted to primary
4. Route53/DNS failover to secondary Istio Gateway
5. Estimated RTO: 2 hours (database promote) + 30 min (Velero restore)
6. Testing: DR drill scheduled quarterly

## 9. DNS + SSL

- **cert-manager**: ACME issuer (Let's Encrypt production) + ClusterIssuer
- **external-dns**: Syncs Ingress/Istio Gateway hostnames to Route53 / Azure DNS / Cloud DNS
- **Istio Gateway**: TLS via cert-manager `Certificate` resource, `credentialName` reference

## Implementation Phases

| Phase | Description | Dependencies |
|---|---|---|
| **1** | Terraform: backend.tf + module audit + fixes | None |
| **2** | Bootstrap: ArgoCD install + infra-components ApplicationSet | Phase 1 |
| **3** | Helm charts: Create 37 charts + umbrella chart | Phase 2 |
| **4** | CI/CD: GitHub Actions pipelines for all services | Phase 3 |
| **5** | Observability: Prometheus + Grafana + ELK deployment | Phase 2 |
| **6** | Security: Istio + Vault + ESO + NetworkPolicies | Phase 2 |
| **7** | Backup: Velero + DB snapshot schedules | Phase 1 |
| **8** | DNS: cert-manager + external-dns + Istio Gateway TLS | Phase 2 |
