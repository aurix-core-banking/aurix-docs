# Infrastructure as Code — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Finalize infrastructure-as-code: Terraform backend state + module completion, ArgoCD GitOps bootstrap, 37 Helm charts, CI/CD pipelines, and full observability/security/DR stack.

**Architecture:** Multi-cloud (AWS EKS, Azure AKS, GCP GKE) provisioned by Terraform. ArgoCD app-of-apps manages all Kubernetes resources via Helm umbrella chart. GitHub Actions builds images, ArgoCD Image Updater detects and syncs new tags. Istio for service mesh, Vault+ESO for secrets, Prometheus+Grafana+ELK for observability, Velero for DR.

**Tech Stack:** Terraform 1.x, Helm 3, ArgoCD 2.x, Istio 1.20+, Prometheus Operator, ECK (Elasticsearch Operator), cert-manager, Vault, External Secrets Operator, Velero, GitHub Actions, Docker buildx

## Global Constraints

- All Terraform: `required_version = ">= 1.0"`, use `hashicorp/aws ~> 5.0`, `hashicorp/azurerm ~> 3.0`, `hashicorp/google ~> 4.0`
- All Helm charts: `apiVersion: v2`, AppVersion matches service version
- All K8s manifests: namespaced under `aurix-platform` unless cluster-scoped
- All deployments: `securityContext.runAsNonRoot: true`, `allowPrivilegeEscalation: false`
- All GitHub Actions: OIDC-based auth to cloud providers (no long-lived secrets)
- All paths relative to repo root unless specified
- Commit messages: conventional commits (`feat(infra):`, `fix(infra):`, etc.)

---

## File Map

### Terraform
- Create: `infrastructure/terraform/backend.tf`
- Modify: `infrastructure/terraform/modules/aws/vpc/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/aws/eks/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/aws/rds/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/aws/redis/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/aws/alb/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/aws/cloudfront/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/aws/s3/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/aws/cloudwatch/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/aws/iam/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/aws/security/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/azure/resource-group/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/azure/aks/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/azure/postgresql/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/azure/redis/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/gcp/vpc/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/gcp/gke/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/gcp/cloud-sql/main.tf` (add outputs)
- Modify: `infrastructure/terraform/modules/gcp/memorystore/main.tf` (add outputs)

### ArgoCD
- Create: `infrastructure/kubernetes/argocd/projects/infra.yaml`
- Create: `infrastructure/kubernetes/argocd/projects/platform.yaml`
- Create: `infrastructure/kubernetes/argocd/bootstrap/infra-components.yaml`
- Create: `infrastructure/kubernetes/argocd/applicationsets/microservices.yaml`

### Helm Charts (186 files — 37 services × ~5 templates each)
- Create: `infrastructure/kubernetes/charts/<service>/Chart.yaml`
- Create: `infrastructure/kubernetes/charts/<service>/values.yaml`
- Create: `infrastructure/kubernetes/charts/<service>/templates/deployment.yaml`
- Create: `infrastructure/kubernetes/charts/<service>/templates/service.yaml`
- Create: `infrastructure/kubernetes/charts/<service>/templates/hpa.yaml`
- Create: `infrastructure/kubernetes/charts/<service>/templates/_helpers.tpl`
- Create: `infrastructure/kubernetes/charts/<service>/templates/serviceaccount.yaml`
- Create: `infrastructure/kubernetes/charts/umbrella/Chart.yaml`
- Create: `infrastructure/kubernetes/charts/umbrella/values.yaml`
- Create: `infrastructure/kubernetes/charts/umbrella/values-dev.yaml`
- Create: `infrastructure/kubernetes/charts/umbrella/values-staging.yaml`
- Create: `infrastructure/kubernetes/charts/umbrella/values-prod.yaml`
- Create: `infrastructure/kubernetes/charts/umbrella/values-aws.yaml`
- Create: `infrastructure/kubernetes/charts/umbrella/values-azure.yaml`
- Create: `infrastructure/kubernetes/charts/umbrella/values-gcp.yaml`

### Infra Charts
- Create: `infrastructure/kubernetes/charts/infra/istio-operator/Chart.yaml`
- Create: `infrastructure/kubernetes/charts/infra/istio-operator/values.yaml`
- Create: `infrastructure/kubernetes/charts/infra/istio-operator/templates/`
- Create: `infrastructure/kubernetes/charts/infra/cert-manager/Chart.yaml`
- Create: `infrastructure/kubernetes/charts/infra/cert-manager/values.yaml`
- Create: `infrastructure/kubernetes/charts/infra/external-secrets/Chart.yaml`
- Create: `infrastructure/kubernetes/charts/infra/external-secrets/values.yaml`
- Create: `infrastructure/kubernetes/charts/infra/velero/Chart.yaml`
- Create: `infrastructure/kubernetes/charts/infra/velero/values.yaml`
- Create: `infrastructure/kubernetes/charts/infra/prometheus/Chart.yaml`
- Create: `infrastructure/kubernetes/charts/infra/prometheus/values.yaml`
- Create: `infrastructure/kubernetes/charts/infra/elk/Chart.yaml`
- Create: `infrastructure/kubernetes/charts/infra/elk/values.yaml`

### CI/CD
- Create: `.github/workflows/ci-aurix-core.yml`
- Create: `.github/workflows/ci-aurix-gateway.yml`
- Create: `.github/workflows/ci-aurix-onboarding.yml`
- Create: `.github/workflows/ci-aurix-pix.yml`
- Create: `.github/workflows/ci-aurix-bacen.yml`
- Create: `.github/workflows/ci-frontend-admin.yml`
- Create: `.github/workflows/ci-frontend-web.yml`
- Create: `.github/workflows/ci-frontend-mobile.yml`
- Create: `.github/workflows/terraform-plan.yml`
- Create: `.github/workflows/terraform-apply.yml`

### Security
- Create: `infrastructure/kubernetes/base/network-policies/default-deny.yaml`
- Create: `infrastructure/kubernetes/base/network-policies/allow-monitoring.yaml`
- Create: `infrastructure/kubernetes/base/network-policies/allow-mesh.yaml`
- Create: `infrastructure/kubernetes/base/istio/peer-authentication.yaml`
- Create: `infrastructure/kubernetes/base/istio/ingress-gateway.yaml`

### Backup
- Create: `infrastructure/kubernetes/base/velero/schedule.yaml`

### Observability
- Create: `infrastructure/kubernetes/base/monitoring/prometheus-servicemonitors.yaml`
- Create: `infrastructure/kubernetes/base/monitoring/grafana-dashboards.yaml`

---

### Task 1: Terraform backend state + locking

**Files:**
- Create: `infrastructure/terraform/backend.tf`

- [ ] **Step 1: Create backend.tf**

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "aurix-terraform-state-${var.environment}"
    key            = "aurix-platform/terraform.tfstate"
    region         = var.aws_region
    encrypt        = true
    dynamodb_table = "aurix-terraform-locks"
  }
}

# Note: Azure backend uses "azurerm" with storage_account_name and container_name
# Note: GCP backend uses "gcs" with bucket name
#
# For Azure:
# terraform {
#   backend "azurerm" {
#     storage_account_name = "aurixtfstate${var.environment}"
#     container_name       = "terraform-state"
#     key                  = "aurix-platform.tfstate"
#   }
# }
#
# For GCP:
# terraform {
#   backend "gcs" {
#     bucket = "aurix-terraform-state-${var.environment}"
#     prefix = "terraform/state"
#   }
# }
```

- [ ] **Step 2: Run terraform init to verify**

Run: `terraform -chdir=infrastructure/terraform init`
Expected: Backend initialized, modules downloaded.

- [ ] **Step 3: Commit**

```bash
git add infrastructure/terraform/backend.tf
git commit -m "feat(infra): add Terraform backend state + DynamoDB locking"
```

---

### Task 2: Audit and complete Terraform module outputs

**Files:**
- Modify: all `infrastructure/terraform/modules/*/main.tf`

- [ ] **Step 1: Audit VPC module**

Read `infrastructure/terraform/modules/aws/vpc/main.tf` and check for outputs: `vpc_id`, `private_subnet_ids`, `public_subnet_ids`, `database_subnet_ids`. Add any missing outputs.

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "database_subnet_ids" {
  value = aws_subnet.database[*].id
}
```

- [ ] **Step 2: Audit EKS module**

```hcl
output "cluster_name" {
  value = aws_eks_cluster.main.name
}

output "cluster_endpoint" {
  value = aws_eks_cluster.main.endpoint
}

output "cluster_ca_certificate" {
  value = base64decode(aws_eks_cluster.main.certificate_authority[0].data)
}
```

- [ ] **Step 3: Audit RDS module**

```hcl
output "cluster_endpoint" {
  value = aws_rds_cluster.main.endpoint
}

output "cluster_port" {
  value = aws_rds_cluster.main.port
}

output "cluster_arn" {
  value = aws_rds_cluster.main.arn
}
```

- [ ] **Step 4: Audit Redis, ALB, CloudFront, S3, CloudWatch, IAM, Security modules**

Read each module's `main.tf`. For each, add any `output` blocks that are referenced in `infrastructure/terraform/main.tf`. Ensure variable `main.tf` references match module outputs.

- [ ] **Step 5: Audit Azure modules**

Same pattern for `infrastructure/terraform/modules/azure/*/main.tf`: ensure outputs for `name`, `fqdn`, `endpoint` match root module references.

- [ ] **Step 6: Audit GCP modules**

Same pattern for `infrastructure/terraform/modules/gcp/*/main.tf`.

- [ ] **Step 7: Run terraform validate**

```bash
terraform -chdir=infrastructure/terraform validate
```

Expected: No errors, configuration is valid.

- [ ] **Step 8: Commit**

```bash
git add infrastructure/terraform/
git commit -m "feat(infra): complete Terraform module outputs for all cloud providers"
```

---

### Task 3: ArgoCD bootstrap manifests

**Files:**
- Create: `infrastructure/kubernetes/argocd/projects/infra.yaml`
- Create: `infrastructure/kubernetes/argocd/projects/platform.yaml`
- Create: `infrastructure/kubernetes/argocd/bootstrap/infra-components.yaml`
- Create: `infrastructure/kubernetes/argocd/applicationsets/microservices.yaml`

- [ ] **Step 1: Create infra ArgoCD Project**

```yaml
# infrastructure/kubernetes/argocd/projects/infra.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: infra
  namespace: argocd
spec:
  description: Cluster infrastructure components
  sourceRepos:
  - 'https://charts.istio.io'
  - 'https://charts.jetstack.io'
  - 'https://charts.external-secrets.io'
  - 'https://vmware-tanzu.github.io/helm-charts'
  - 'https://prometheus-community.github.io/helm-charts'
  - 'https://helm.elastic.co'
  - 'https://helm.hashicorp.com'
  destinations:
  - namespace: 'istio-system'
    server: 'https://kubernetes.default.svc'
  - namespace: 'cert-manager'
    server: 'https://kubernetes.default.svc'
  - namespace: 'external-secrets'
    server: 'https://kubernetes.default.svc'
  - namespace: 'velero'
    server: 'https://kubernetes.default.svc'
  - namespace: 'monitoring'
    server: 'https://kubernetes.default.svc'
  - namespace: 'elastic-system'
    server: 'https://kubernetes.default.svc'
  - namespace: 'vault'
    server: 'https://kubernetes.default.svc'
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  namespaceResourceWhitelist:
  - group: '*'
    kind: '*'
```

- [ ] **Step 2: Create platform ArgoCD Project**

```yaml
# infrastructure/kubernetes/argocd/projects/platform.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform
  namespace: argocd
spec:
  description: Aurix Platform microservices
  sourceRepos:
  - 'https://github.com/anomalyco/aurix-platform'
  destinations:
  - namespace: 'aurix-platform'
    server: 'https://kubernetes.default.svc'
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  namespaceResourceWhitelist:
  - group: '*'
    kind: '*'
  orphanedResources:
    warn: true
```

- [ ] **Step 3: Create infra-components bootstrap Application**

```yaml
# infrastructure/kubernetes/argocd/bootstrap/infra-components.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: infra-components
  namespace: argocd
spec:
  project: infra
  source:
    repoURL: https://github.com/anomalyco/aurix-platform
    path: infrastructure/kubernetes/charts/infra
    targetRevision: HEAD
    helm:
      valueFiles:
      - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

- [ ] **Step 4: Create microservices ApplicationSet**

```yaml
# infrastructure/kubernetes/argocd/applicationsets/microservices.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/anomalyco/aurix-platform
      revision: HEAD
      directories:
      - path: infrastructure/kubernetes/charts/aurix-*
  template:
    metadata:
      name: '{{path.basename}}'
      labels:
        app.kubernetes.io/part-of: aurix-platform
    spec:
      project: platform
      source:
        repoURL: https://github.com/anomalyco/aurix-platform
        targetRevision: HEAD
        path: '{{path}}'
        helm:
          valueFiles:
          - values.yaml
      destination:
        server: https://kubernetes.default.svc
        namespace: aurix-platform
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        retry:
          limit: 3
        syncOptions:
        - CreateNamespace=true
```

- [ ] **Step 5: Commit**

```bash
git add infrastructure/kubernetes/argocd/
git commit -m "feat(infra): add ArgoCD bootstrap manifests + ApplicationSet"
```

---

### Task 4: Infra Helm charts (Istio, cert-manager, ESO, Velero, Prometheus, ELK)

**Files:**
- Create: `infrastructure/kubernetes/charts/infra/Chart.yaml`
- Create: `infrastructure/kubernetes/charts/infra/values.yaml`
- Create: `infrastructure/kubernetes/charts/infra/templates/`

- [ ] **Step 1: Create infra umbrella Chart.yaml**

```yaml
# infrastructure/kubernetes/charts/infra/Chart.yaml
apiVersion: v2
name: infra
description: Aurix Platform infrastructure components
type: application
version: 0.1.0
appVersion: "1.0"
dependencies:
  - name: istio-base
    version: "1.20.2"
    repository: "https://charts.istio.io"
    alias: istio
  - name: cert-manager
    version: "v1.14.0"
    repository: "https://charts.jetstack.io"
  - name: external-secrets
    version: "0.9.0"
    repository: "https://charts.external-secrets.io"
  - name: velero
    version: "5.0.0"
    repository: "https://vmware-tanzu.github.io/helm-charts"
  - name: kube-prometheus-stack
    version: "51.0.0"
    repository: "https://prometheus-community.github.io/helm-charts"
    alias: prometheus
  - name: eck-operator
    version: "2.10.0"
    repository: "https://helm.elastic.co"
    alias: elastic
  - name: vault
    version: "0.27.0"
    repository: "https://helm.hashicorp.com"
```

- [ ] **Step 2: Create infra values.yaml**

```yaml
# infrastructure/kubernetes/charts/infra/values.yaml
istio:
  global:
    meshID: aurix-mesh
    multiCluster:
      clusterName: aurix-primary
    network: aurix-network

cert-manager:
  installCRDs: true
  startupapicheck:
    enabled: false

external-secrets:
  installCRDs: true
  serviceAccount:
    create: true
    name: external-secrets-sa

velero:
  configuration:
    provider: aws
    backupStorageLocation:
      bucket: aurix-velero-backups
    volumeSnapshotLocation:
      config:
        region: us-east-1
  credentials:
    useSecret: false
    existingSecret: velero-credentials
  schedules:
    daily:
      schedule: "0 2 * * *"
      template:
        ttl: "720h"  # 30 days

prometheus:
  grafana:
    adminPassword: changeme
    ingress:
      enabled: true
      hosts: ["grafana.aurix-platform.com"]
  alertmanager:
    enabled: true
  prometheus:
    prometheusSpec:
      retention: 15d
      retentionSize: 50GB

elastic:
  createOperator: true
  createCustomResource: false

vault:
  server:
    dev:
      enabled: false
    ha:
      enabled: true
      replicas: 3
    serviceAccount:
      create: true
    ui:
      enabled: true
```

- [ ] **Step 3: Create infra templates (Istio operator + ClusterIssuer)**

```yaml
# infrastructure/kubernetes/charts/infra/templates/istio-operator.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: aurix-istio
  namespace: istio-system
spec:
  profile: default
  components:
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        service:
          type: LoadBalancer
          ports:
          - port: 80
            name: http
          - port: 443
            name: https
    egressGateways:
    - name: istio-egressgateway
      enabled: false
  meshConfig:
    accessLogFile: /dev/stdout
    enableTracing: true
```

```yaml
# infrastructure/kubernetes/charts/infra/templates/cert-manager-cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: infra@aurixplatform.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: istio
```

```yaml
# infrastructure/kubernetes/charts/infra/templates/elasticsearch-cluster.yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: aurix
  namespace: elastic-system
spec:
  version: 8.11.0
  nodeSets:
  - name: hot
    count: 3
    config:
      node.store.allow_mmap: false
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          env:
          - name: ES_JAVA_OPTS
            value: "-Xms2g -Xmx2g"
          resources:
            requests:
              memory: 4Gi
              cpu: 1
            limits:
              memory: 4Gi
```

```yaml
# infrastructure/kubernetes/charts/infra/templates/kibana.yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: aurix
  namespace: elastic-system
spec:
  version: 8.11.0
  count: 1
  elasticsearchRef:
    name: aurix
  config:
    server.publicBaseUrl: https://kibana.aurix-platform.com
  podTemplate:
    spec:
      containers:
      - name: kibana
        resources:
          requests:
            memory: 1Gi
            cpu: 500m
```

```yaml
# infrastructure/kubernetes/charts/infra/templates/filebeat.yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: filebeat
  namespace: elastic-system
spec:
  type: filebeat
  version: 8.11.0
  elasticsearchRef:
    name: aurix
  config:
    filebeat.inputs:
    - type: container
      paths:
      - /var/log/containers/*.log
      processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers:
          - logs_path:
              logs_path: /var/log/containers/
  daemonSet:
    podTemplate:
      spec:
        serviceAccountName: filebeat
        automountServiceAccountToken: true
        containers:
        - name: filebeat
          volumeMounts:
          - name: varlogcontainers
            mountPath: /var/log/containers
          - name: varlogpods
            mountPath: /var/log/pods
          - name: varlibdockercontainers
            mountPath: /var/lib/docker/containers
        volumes:
        - name: varlogcontainers
          hostPath:
            path: /var/log/containers
        - name: varlogpods
          hostPath:
            path: /var/log/pods
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
```

- [ ] **Step 4: Commit**

```bash
git add infrastructure/kubernetes/charts/infra/
git commit -m "feat(infra): add infra Helm chart with Istio, cert-manager, ESO, Velero, Prometheus, ELK, Vault"
```

---

### Task 5: Create microservice Helm chart template + first 10 charts

**Pattern** — each chart has the same structure. Use `aurix-core` as the reference template.

**Files:**
- Create: `infrastructure/kubernetes/charts/aurix-core/Chart.yaml`
- Create: `infrastructure/kubernetes/charts/aurix-core/values.yaml`
- Create: `infrastructure/kubernetes/charts/aurix-core/templates/_helpers.tpl`
- Create: `infrastructure/kubernetes/charts/aurix-core/templates/deployment.yaml`
- Create: `infrastructure/kubernetes/charts/aurix-core/templates/service.yaml`
- Create: `infrastructure/kubernetes/charts/aurix-core/templates/hpa.yaml`
- Create: `infrastructure/kubernetes/charts/aurix-core/templates/serviceaccount.yaml`
- Create: `infrastructure/kubernetes/charts/aurix-core/templates/pdb.yaml`
- Repeat for: aurix-gateway, aurix-pix, aurix-bacen, aurix-onboarding, aurix-security, aurix-openfinance, aurix-analytics, aurix-catalog, aurix-credit

- [ ] **Step 1: Create aurix-core Chart.yaml**

```yaml
# infrastructure/kubernetes/charts/aurix-core/Chart.yaml
apiVersion: v2
name: aurix-core
description: Aurix Core Banking service
type: application
version: 0.1.0
appVersion: "1.0.0"
```

- [ ] **Step 2: Create _helpers.tpl**

```yaml
# infrastructure/kubernetes/charts/aurix-core/templates/_helpers.tpl
{{- define "aurix-core.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "aurix-core.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{- define "aurix-core.labels" -}}
helm.sh/chart: {{ include "aurix-core.name" . }}-{{ .Chart.Version | replace "+" "_" }}
{{ include "aurix-core.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "aurix-core.selectorLabels" -}}
app.kubernetes.io/name: {{ include "aurix-core.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "aurix-core.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "aurix-core.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

- [ ] **Step 3: Create values.yaml**

```yaml
# infrastructure/kubernetes/charts/aurix-core/values.yaml
replicaCount: 3

image:
  repository: aurix-core
  tag: latest
  pullPolicy: Always

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}
podSecurityContext: {}
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault

service:
  type: ClusterIP
  port: 8080
  managementPort: 8081

ingress:
  enabled: false

resources:
  requests:
    memory: 512Mi
    cpu: 250m
  limits:
    memory: 1Gi
    cpu: 500m

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

pdb:
  enabled: true
  minAvailable: 2

probes:
  liveness:
    path: /actuator/health/live
    port: 8081
    initialDelaySeconds: 60
    periodSeconds: 30
    timeoutSeconds: 10
    failureThreshold: 3
  readiness:
    path: /actuator/health/ready
    port: 8081
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3
  startup:
    path: /actuator/health
    port: 8081
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 30

env:
  SPRING_PROFILES_ACTIVE: production
  SPRING_DATASOURCE_URL: ""
  SPRING_DATASOURCE_USERNAME: ""
  SPRING_DATASOURCE_PASSWORD: ""
  SPRING_REDIS_HOST: aurix-redis
  SPRING_REDIS_PORT: "6379"
  SPRING_KAFKA_BOOTSTRAP_SERVERS: aurix-kafka:9092

envFrom: []
existingSecret: ""

nodeSelector: {}
tolerations: []
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app.kubernetes.io/name
            operator: In
            values:
            - "{{ include "aurix-core.name" . }}"
        topologyKey: kubernetes.io/hostname

extraVolumes: []
extraVolumeMounts: []
```

- [ ] **Step 4: Create deployment.yaml**

```yaml
# infrastructure/kubernetes/charts/aurix-core/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "aurix-core.fullname" . }}
  labels:
    {{- include "aurix-core.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "aurix-core.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "aurix-core.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "aurix-core.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
      - name: {{ .Chart.Name }}
        securityContext:
          {{- toYaml .Values.securityContext | nindent 10 }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.port }}
          name: http
        - containerPort: {{ .Values.service.managementPort }}
          name: management
        env:
        {{- range $key, $value := .Values.env }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}
        {{- if .Values.existingSecret }}
        envFrom:
        - secretRef:
            name: {{ .Values.existingSecret }}
        {{- end }}
        {{- with .Values.extraEnv }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        livenessProbe:
          httpGet:
            path: {{ .Values.probes.liveness.path }}
            port: {{ .Values.probes.liveness.port }}
          initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
          periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
          timeoutSeconds: {{ .Values.probes.liveness.timeoutSeconds }}
          failureThreshold: {{ .Values.probes.liveness.failureThreshold }}
        readinessProbe:
          httpGet:
            path: {{ .Values.probes.readiness.path }}
            port: {{ .Values.probes.readiness.port }}
          initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
          periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
          timeoutSeconds: {{ .Values.probes.readiness.timeoutSeconds }}
          failureThreshold: {{ .Values.probes.readiness.failureThreshold }}
        startupProbe:
          httpGet:
            path: {{ .Values.probes.startup.path }}
            port: {{ .Values.probes.startup.port }}
          initialDelaySeconds: {{ .Values.probes.startup.initialDelaySeconds }}
          periodSeconds: {{ .Values.probes.startup.periodSeconds }}
          timeoutSeconds: {{ .Values.probes.startup.timeoutSeconds }}
          failureThreshold: {{ .Values.probes.startup.failureThreshold }}
        {{- with .Values.extraVolumeMounts }}
        volumeMounts:
          {{- toYaml . | nindent 8 }}
        {{- end }}
      {{- with .Values.extraVolumes }}
      volumes:
        {{- toYaml . | nindent 6 }}
      {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

- [ ] **Step 5: Create service.yaml**

```yaml
# infrastructure/kubernetes/charts/aurix-core/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "aurix-core.fullname" . }}
  labels:
    {{- include "aurix-core.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.port }}
    protocol: TCP
    name: http
  - port: {{ .Values.service.managementPort }}
    targetPort: {{ .Values.service.managementPort }}
    protocol: TCP
    name: management
  selector:
    {{- include "aurix-core.selectorLabels" . | nindent 4 }}
```

- [ ] **Step 6: Create hpa.yaml**

```yaml
# infrastructure/kubernetes/charts/aurix-core/templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "aurix-core.fullname" . }}
  labels:
    {{- include "aurix-core.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "aurix-core.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
{{- end }}
```

- [ ] **Step 7: Create serviceaccount.yaml**

```yaml
# infrastructure/kubernetes/charts/aurix-core/templates/serviceaccount.yaml
{{- if .Values.serviceAccount.create }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "aurix-core.serviceAccountName" . }}
  labels:
    {{- include "aurix-core.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
```

- [ ] **Step 8: Create pdb.yaml**

```yaml
# infrastructure/kubernetes/charts/aurix-core/templates/pdb.yaml
{{- if .Values.pdb.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "aurix-core.fullname" . }}
  labels:
    {{- include "aurix-core.labels" . | nindent 4 }}
spec:
  {{- if .Values.pdb.minAvailable }}
  minAvailable: {{ .Values.pdb.minAvailable }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "aurix-core.selectorLabels" . | nindent 6 }}
{{- end }}
```

- [ ] **Step 9: Run Helm lint**

```bash
helm lint infrastructure/kubernetes/charts/aurix-core/
```

Expected: No errors. Chart with valid template.

- [ ] **Step 10: Copy chart for remaining services**

Copy the entire `aurix-core` chart structure for the first 9 additional services (gateway, pix, bacen, onboarding, security, openfinance, analytics, catalog, credit). Update `Chart.yaml` name and `values.yaml` image.repository for each.

```bash
for service in gateway pix bacen onboarding security openfinance analytics catalog credit; do
  cp -r infrastructure/kubernetes/charts/aurix-core "infrastructure/kubernetes/charts/aurix-${service}"
  sed -i "s/name: aurix-core/name: aurix-${service}/" "infrastructure/kubernetes/charts/aurix-${service}/Chart.yaml"
  sed -i "s/repository: aurix-core/repository: aurix-${service}/" "infrastructure/kubernetes/charts/aurix-${service}/values.yaml"
done
```

- [ ] **Step 11: Lint all copied charts**

```bash
for chart in infrastructure/kubernetes/charts/aurix-*; do
  helm lint "$chart"
done
```

Expected: All charts pass lint successfully.

- [ ] **Step 12: Commit**

```bash
git add infrastructure/kubernetes/charts/aurix-core/ infrastructure/kubernetes/charts/aurix-gateway/ infrastructure/kubernetes/charts/aurix-pix/ infrastructure/kubernetes/charts/aurix-bacen/ infrastructure/kubernetes/charts/aurix-onboarding/ infrastructure/kubernetes/charts/aurix-security/ infrastructure/kubernetes/charts/aurix-openfinance/ infrastructure/kubernetes/charts/aurix-analytics/ infrastructure/kubernetes/charts/aurix-catalog/ infrastructure/kubernetes/charts/aurix-credit/
git commit -m "feat(infra): add Helm charts for core 10 microservices"
```

---

### Task 6: Create remaining 27 microservice Helm charts

**Files:** Same structure as Task 5 for 27 additional services.

Service list: accounting, ai, audit, baas, billing, budget, cambio, cartoes, compliance, consignado, controller, cost, financial, financiamento, internet-banking, investimento, mobile-banking, organization, poupanca, pricing, provisioning, salario, seguros, settlement, shared, tax, treasury, webhooks

- [ ] **Step 1: Copy and customize for each remaining service**

```bash
for service in accounting ai audit baas billing budget cambio cartoes compliance consignado controller cost financial financiamento internet-banking investimento mobile-banking organization poupanca pricing provisioning salario seguros settlement shared tax treasury webhooks; do
  cp -r infrastructure/kubernetes/charts/aurix-core "infrastructure/kubernetes/charts/aurix-${service}"
  sed -i "s/name: aurix-core/name: aurix-${service}/" "infrastructure/kubernetes/charts/aurix-${service}/Chart.yaml"
  sed -i "s/repository: aurix-core/repository: aurix-${service}/" "infrastructure/kubernetes/charts/aurix-${service}/values.yaml"
  helm lint "infrastructure/kubernetes/charts/aurix-${service}"
done
```

- [ ] **Step 2: Commit**

```bash
git add infrastructure/kubernetes/charts/aurix-*/
git commit -m "feat(infra): add Helm charts for remaining 27 microservices"
```

---

### Task 7: Umbrella chart + environment overlays

**Files:**
- Create: `infrastructure/kubernetes/charts/umbrella/Chart.yaml`
- Create: `infrastructure/kubernetes/charts/umbrella/values.yaml`
- Create: `infrastructure/kubernetes/charts/umbrella/values-dev.yaml`
- Create: `infrastructure/kubernetes/charts/umbrella/values-staging.yaml`
- Create: `infrastructure/kubernetes/charts/umbrella/values-prod.yaml`
- Create: `infrastructure/kubernetes/charts/umbrella/values-aws.yaml`
- Create: `infrastructure/kubernetes/charts/umbrella/values-azure.yaml`
- Create: `infrastructure/kubernetes/charts/umbrella/values-gcp.yaml`

- [ ] **Step 1: Create umbrella Chart.yaml**

```yaml
# infrastructure/kubernetes/charts/umbrella/Chart.yaml
apiVersion: v2
name: umbrella
description: Aurix Platform umbrella chart — deploys all microservices
type: application
version: 0.1.0
appVersion: "1.0"
dependencies:
  - name: aurix-core
    version: "0.1.0"
    repository: "file://../aurix-core"
  - name: aurix-gateway
    version: "0.1.0"
    repository: "file://../aurix-gateway"
  - name: aurix-pix
    version: "0.1.0"
    repository: "file://../aurix-pix"
  - name: aurix-bacen
    version: "0.1.0"
    repository: "file://../aurix-bacen"
  - name: aurix-onboarding
    version: "0.1.0"
    repository: "file://../aurix-onboarding"
  - name: aurix-security
    version: "0.1.0"
    repository: "file://../aurix-security"
  - name: aurix-openfinance
    version: "0.1.0"
    repository: "file://../aurix-openfinance"
  - name: aurix-analytics
    version: "0.1.0"
    repository: "file://../aurix-analytics"
  - name: aurix-catalog
    version: "0.1.0"
    repository: "file://../aurix-catalog"
  - name: aurix-credit
    version: "0.1.0"
    repository: "file://../aurix-credit"
  - name: aurix-cartoes
    version: "0.1.0"
    repository: "file://../aurix-cartoes"
  - name: aurix-consignado
    version: "0.1.0"
    repository: "file://../aurix-consignado"
  - name: aurix-financiamento
    version: "0.1.0"
    repository: "file://../aurix-financiamento"
  - name: aurix-cambio
    version: "0.1.0"
    repository: "file://../aurix-cambio"
  - name: aurix-seguros
    version: "0.1.0"
    repository: "file://../aurix-seguros"
  - name: aurix-investimento
    version: "0.1.0"
    repository: "file://../aurix-investimento"
  - name: aurix-salario
    version: "0.1.0"
    repository: "file://../aurix-salario"
  - name: aurix-poupanca
    version: "0.1.0"
    repository: "file://../aurix-poupanca"
  - name: aurix-accounting
    version: "0.1.0"
    repository: "file://../aurix-accounting"
  - name: aurix-billing
    version: "0.1.0"
    repository: "file://../aurix-billing"
  - name: aurix-compliance
    version: "0.1.0"
    repository: "file://../aurix-compliance"
  - name: aurix-audit
    version: "0.1.0"
    repository: "file://../aurix-audit"
  - name: aurix-webhooks
    version: "0.1.0"
    repository: "file://../aurix-webhooks"
  - name: aurix-organization
    version: "0.1.0"
    repository: "file://../aurix-organization"
  - name: aurix-provisioning
    version: "0.1.0"
    repository: "file://../aurix-provisioning"
  - name: aurix-treasury
    version: "0.1.0"
    repository: "file://../aurix-treasury"
  - name: aurix-tax
    version: "0.1.0"
    repository: "file://../aurix-tax"
  - name: aurix-cost
    version: "0.1.0"
    repository: "file://../aurix-cost"
  - name: aurix-budget
    version: "0.1.0"
    repository: "file://../aurix-budget"
  - name: aurix-settlement
    version: "0.1.0"
    repository: "file://../aurix-settlement"
  - name: aurix-pricing
    version: "0.1.0"
    repository: "file://../aurix-pricing"
  - name: aurix-baas
    version: "0.1.0"
    repository: "file://../aurix-baas"
  - name: aurix-internet-banking
    version: "0.1.0"
    repository: "file://../aurix-internet-banking"
  - name: aurix-mobile-banking
    version: "0.1.0"
    repository: "file://../aurix-mobile-banking"
  - name: aurix-ai
    version: "0.1.0"
    repository: "file://../aurix-ai"
  - name: aurix-controller
    version: "0.1.0"
    repository: "file://../aurix-controller"
```

- [ ] **Step 2: Create umbrella values.yaml (dev defaults)**

```yaml
# infrastructure/kubernetes/charts/umbrella/values.yaml
global:
  environment: dev
  replicas: 1
  imageTag: latest
  imagePullPolicy: Always

aurix-core:
  replicaCount: 1
  env:
    SPRING_PROFILES_ACTIVE: dev
  resources:
    requests:
      memory: 256Mi
      cpu: 100m
    limits:
      memory: 512Mi
      cpu: 250m

aurix-gateway:
  replicaCount: 1
  service:
    type: LoadBalancer
  resources:
    requests:
      memory: 256Mi
      cpu: 100m
    limits:
      memory: 512Mi
      cpu: 250m

# All other microservices use same default pattern
# (truncated for brevity — in practice list all 37)
```

- [ ] **Step 3: Commit**

```bash
git add infrastructure/kubernetes/charts/umbrella/
git commit -m "feat(infra): add umbrella Helm chart with all sub-chart dependencies"
```

---

### Task 8: GitHub Actions CI pipelines

**Files:**
- Create: `.github/workflows/ci-aurix-core.yml`
- Create: `.github/workflows/ci-aurix-gateway.yml`
- Create: `.github/workflows/ci-aurix-onboarding.yml`
- Create: `.github/workflows/ci-aurix-pix.yml`
- Create: `.github/workflows/ci-aurix-bacen.yml`
- Create: `.github/workflows/ci-frontend-admin.yml`
- Create: `.github/workflows/ci-frontend-web.yml`
- Create: `.github/workflows/ci-frontend-mobile.yml`
- Create: `.github/workflows/terraform-plan.yml`
- Create: `.github/workflows/terraform-apply.yml`

- [ ] **Step 1: Create CI for aurix-core**

```yaml
# .github/workflows/ci-aurix-core.yml
name: Build aurix-core

on:
  push:
    branches: [main, develop]
    paths: ["backend/aurix-core/**"]
  pull_request:
    branches: [main]
    paths: ["backend/aurix-core/**"]

env:
  REGISTRY: ${{ secrets.REGISTRY_URL }}
  IMAGE_NAME: aurix-core

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-java@v4
      with:
        java-version: '25'
        distribution: 'temurin'
        cache: maven
    - name: Build and test
      run: mvn -pl aurix-core -am compile test -q
    - name: Trivy scan
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: fs
        scan-ref: backend/aurix-core/
        severity: CRITICAL,HIGH
        exit-code: 1
    - name: Docker buildx
      uses: docker/setup-buildx-action@v3
    - name: Login to registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: backend/
        file: backend/aurix-core/Dockerfile
        push: true
        tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        platforms: linux/amd64,linux/arm64
        cache-from: type=gha
        cache-to: type=gha,mode=max
    - name: Update ArgoCD image tag
      uses: actions/github-script@v7
      with:
        script: |
          await github.rest.actions.createWorkflowDispatch({
            owner: context.repo.owner,
            repo: context.repo.repo,
            workflow_id: 'argocd-image-updater.yaml',
            ref: 'main'
          });
```

- [ ] **Step 2: Create frontend CI workflows**

```yaml
# .github/workflows/ci-frontend-web.yml
name: Build Frontend Web

on:
  push:
    branches: [main, develop]
    paths: ["frontend/aurix-web/**"]
  pull_request:
    branches: [main]
    paths: ["frontend/aurix-web/**"]

env:
  REGISTRY: ${{ secrets.REGISTRY_URL }}

jobs:
  build:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
        cache-dependency-path: frontend/package-lock.json
    - name: Install dependencies
      run: npm ci --legacy-peer-deps
    - name: Lint
      run: npm run lint --workspace=aurix-web
    - name: Test
      run: npm test --workspace=aurix-web -- --watchAll=false
    - name: Build
      run: npm run build --workspace=aurix-web
    - name: Docker build and push
      uses: docker/build-push-action@v5
      with:
        context: frontend/
        file: frontend/aurix-web/Dockerfile
        push: true
        tags: ${{ env.REGISTRY }}/aurix-web:${{ github.sha }}
```

- [ ] **Step 3: Create Terraform CI/CD workflows**

```yaml
# .github/workflows/terraform-plan.yml
name: Terraform Plan

on:
  pull_request:
    paths: ["infrastructure/terraform/**"]

jobs:
  plan:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infrastructure/terraform
    steps:
    - uses: actions/checkout@v4
    - uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: 1.6.0
    - name: Terraform fmt
      run: terraform fmt -check
    - name: Terraform init
      run: terraform init
    - name: Terraform validate
      run: terraform validate
    - name: Terraform plan
      run: terraform plan -out=tfplan
    - uses: actions/upload-artifact@v4
      with:
        name: tfplan
        path: infrastructure/terraform/tfplan
```

```yaml
# .github/workflows/terraform-apply.yml
name: Terraform Apply

on:
  push:
    branches: [main]
    paths: ["infrastructure/terraform/**"]

jobs:
  apply:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infrastructure/terraform
    steps:
    - uses: actions/checkout@v4
    - uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: 1.6.0
    - name: Terraform init
      run: terraform init
    - name: Terraform apply
      run: terraform apply -auto-approve
```

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/
git commit -m "feat(infra): add GitHub Actions CI/CD pipelines for backend, frontend, and Terraform"
```

---

### Task 9: Security base (NetworkPolicies, Istio mTLS, Vault+ESO)

**Files:**
- Create: `infrastructure/kubernetes/base/network-policies/default-deny.yaml`
- Create: `infrastructure/kubernetes/base/network-policies/allow-monitoring.yaml`
- Create: `infrastructure/kubernetes/base/network-policies/allow-mesh.yaml`
- Create: `infrastructure/kubernetes/base/istio/peer-authentication.yaml`
- Create: `infrastructure/kubernetes/base/istio/ingress-gateway.yaml`
- Create: `infrastructure/kubernetes/base/external-secrets/secretstore.yaml`
- Create: `infrastructure/kubernetes/base/external-secrets/cluster-secret-store.yaml`

- [ ] **Step 1: Create default-deny NetworkPolicy**

```yaml
# infrastructure/kubernetes/base/network-policies/default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: aurix-platform
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: aurix-platform
spec:
  podSelector: {}
  egress:
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
  policyTypes:
  - Egress
```

- [ ] **Step 2: Create istio PeerAuthentication**

```yaml
# infrastructure/kubernetes/base/istio/peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: aurix-platform
spec:
  mtls:
    mode: STRICT
```

- [ ] **Step 3: Create Istio Ingress Gateway**

```yaml
# infrastructure/kubernetes/base/istio/ingress-gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: aurix-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*.aurix-platform.com"
    tls:
      httpsRedirect: true
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "*.aurix-platform.com"
    tls:
      mode: SIMPLE
      credentialName: aurix-platform-tls
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: aurix-ingress
  namespace: aurix-platform
spec:
  hosts:
  - "api.aurix-platform.com"
  gateways:
  - istio-system/aurix-gateway
  http:
  - match:
    - uri:
        prefix: /api/
    route:
    - destination:
        host: aurix-gateway
        port:
          number: 8080
```

- [ ] **Step 4: Create ClusterSecretStore for ESO + Vault**

```yaml
# infrastructure/kubernetes/base/external-secrets/cluster-secret-store.yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.vault.svc.cluster.local:8200"
      path: "aurix"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "aurix-platform"
          serviceAccountRef:
            name: external-secrets-sa
```

- [ ] **Step 5: Commit**

```bash
git add infrastructure/kubernetes/base/network-policies/ infrastructure/kubernetes/base/istio/ infrastructure/kubernetes/base/external-secrets/
git commit -m "feat(infra): add security base - NetworkPolicies, Istio mTLS, Vault ESO store"
```

---

### Task 10: Monitoring (Prometheus ServiceMonitors + Grafana dashboards + Velero schedule)

**Files:**
- Create: `infrastructure/kubernetes/base/monitoring/prometheus-servicemonitors.yaml`
- Create: `infrastructure/kubernetes/base/monitoring/grafana-dashboards.yaml`
- Create: `infrastructure/kubernetes/base/velero/schedule.yaml`

- [ ] **Step 1: Create ServiceMonitors for all microservices**

```yaml
# infrastructure/kubernetes/base/monitoring/prometheus-servicemonitors.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: aurix-microservices
  namespace: monitoring
  labels:
    release: prometheus
spec:
  jobLabel: app.kubernetes.io/name
  selector:
    matchLabels:
      app.kubernetes.io/part-of: aurix-platform
  namespaceSelector:
    matchNames:
    - aurix-platform
  endpoints:
  - port: management
    interval: 15s
    path: /actuator/prometheus
    relabelings:
    - sourceLabels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
      targetLabel: service
    - sourceLabels: [__meta_kubernetes_namespace]
      targetLabel: namespace
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-proxies
  namespace: monitoring
  labels:
    release: prometheus
spec:
  jobLabel: app
  selector:
    matchLabels:
      app: istio-proxy
  namespaceSelector:
    any: true
  endpoints:
  - port: http-envoy-prom
    interval: 15s
    path: /stats/prometheus
```

- [ ] **Step 2: Create Velero schedule**

```yaml
# infrastructure/kubernetes/base/velero/schedule.yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    ttl: 720h
    includedNamespaces:
    - aurix-platform
    - istio-system
    - monitoring
    - elastic-system
    - cert-manager
    - external-secrets
    - vault
    includedResources:
    - deployment
    - statefulset
    - daemonset
    - configmap
    - secret
    - pvc
    - service
    - ingress
    excludeResources:
    - pods
    snapshotVolumes: true
    storageLocation: default
    volumeSnapshotLocations:
    - default
```

- [ ] **Step 3: Commit**

```bash
git add infrastructure/kubernetes/base/monitoring/ infrastructure/kubernetes/base/velero/
git commit -m "feat(infra): add Prometheus ServiceMonitors, Grafana dashboards, Velero schedule"
```
