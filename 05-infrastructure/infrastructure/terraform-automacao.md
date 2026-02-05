# Automação de implantação e Terraform

## A automação está funcional e integrada ao Terraform?

**Sim, de forma limitada.** O script `infrastructure/scripts/deploy-terraform.sh` está integrado ao Terraform (chama `init`, `plan`, `apply` e opcionalmente aplica manifests Kubernetes). Porém:

1. **Módulos são stubs:** Os módulos em `infrastructure/terraform/modules/` (AWS, Azure, GCP) não criam recursos reais; usam `null_resource` e retornam outputs fictícios (ex.: `vpc-stub-*`, `https://stub.eks.local`). Servem para validar a estrutura e o fluxo (`terraform init` e `terraform validate` funcionam); **não provisionam** VPC, EKS, RDS, Redis etc. de fato.
2. **State local:** Não há bloco `backend` em `main.tf`; o state fica local por padrão. Para state remoto (ex.: S3), é preciso adicionar um `backend "s3"` em `main.tf` e passar `-backend-config` no `terraform init` (ou usar arquivo de config).
3. **Um cloud por vez:** O `main.tf` declara recursos para AWS, Azure e GCP ao mesmo tempo. Para deploy real, use o script com `CLOUD_PROVIDER=aws` (ou `azure`/`gcp`); o script aplica só os módulos daquele cloud com `-target`. A opção `all` faz plan/apply de tudo (exige credenciais dos três clouds).

## O que funciona hoje

| Item | Status |
|------|--------|
| `./infrastructure/scripts/deploy-terraform.sh` | Chama Terraform no diretório correto; suporta `--environment`, `--cloud-provider`, `--deploy-k8s` |
| `terraform init -upgrade` | Funciona (state local) |
| `terraform validate` | Funciona |
| `terraform plan` / `apply` | Executam; criam apenas `null_resource` (stubs) |
| Deploy por cloud (`CLOUD_PROVIDER=aws`) | Aplica só módulos AWS com `-target` |
| Deploy K8s após Terraform | Atualiza kubeconfig (aws/azure/gcp) e aplica `kubernetes/namespace.yaml` e `aureus-core-deployment.yaml` |
| Frontend na automação | Build admin + web com apontamento configurável; deploy junto ao mesmo cloud ou separado (ver secao Deploy frontend abaixo) |

## Deploy frontend

O gerenciador de deploy (`deploy-terraform.sh` / `deploy-terraform.bat`) inclui o frontend na automação de duas formas:

1. **Subir junto no mesmo cloud:** use `--deploy-frontend` (e opcionalmente `--frontend-upload` se tiver S3 e AWS CLI). O apontamento (URL da API/gateway) pode ser definido por `--frontend-api-url` ou pela variável de ambiente `FRONTEND_API_URL`.
2. **Subir frontend separado:** use `--deploy-frontend-only` e informe o apontamento com `--frontend-api-url URL` ou `FRONTEND_API_URL=URL`. O script só executa o build (e opcionalmente upload para S3) dos frontends, sem rodar Terraform/K8s.

Scripts: `build-frontend.sh`/`.bat` (build com `FRONTEND_API_URL`); `deploy-frontend.sh`/`.bat` (build + upload S3 com `FRONTEND_S3_BUCKET_ADMIN`, `FRONTEND_S3_BUCKET_WEB`, `FRONTEND_UPLOAD=true`). Variáveis Terraform: `deploy_frontend`, `frontend_api_url`, `frontend_s3_bucket_admin`, `frontend_s3_bucket_web`.

Exemplos: `./deploy-terraform.sh --deploy-frontend --frontend-api-url https://api.meudominio.com` (mesmo cloud); `./deploy-terraform.sh --deploy-frontend-only --frontend-api-url https://outro-backend.com` (frontend separado com apontamento ajustado).

## Como tornar o deploy “real”

1. **Implementar os módulos:** Substituir o conteúdo dos módulos em `modules/aws`, `modules/azure`, `modules/gcp` por recursos reais (aws_vpc, aws_eks, aws_rds, etc.) em vez de `null_resource` e outputs fictícios.
2. **State remoto (recomendado para produção):** Em `main.tf`, dentro do bloco `terraform { ... }`, adicionar por exemplo:
   ```hcl
   backend "s3" {
     bucket  = "aureus-terraform-state"
     key     = "aureus-platform.tfstate"
     region  = "us-east-1"
     encrypt = true
   }
   ```
   (Ou deixar bucket/key/region para serem preenchidos via `-backend-config` no init.) Criar o bucket (e opcionalmente DynamoDB para lock) antes do primeiro `terraform init`.
3. **Variáveis sensíveis:** Não commitar `db_password`; usar `TF_VAR_db_password` ou backend de secrets (ex.: Vault, AWS Secrets Manager) e passar por `-var` ou arquivo `.tfvars` ignorado pelo Git.

## Uso rápido do script

```bash
cd infrastructure/scripts
./deploy-terraform.sh --environment dev --cloud-provider aws
```

Para aplicar também os manifests Kubernetes após o Terraform:

```bash
DEPLOY_K8S=true ./deploy-terraform.sh --environment dev --cloud-provider aws --deploy-k8s
```

Para incluir o frontend no mesmo deploy (build com apontamento para o gateway/API):

```bash
./deploy-terraform.sh --environment dev --cloud-provider aws --deploy-frontend --frontend-api-url https://api.meudominio.com
```

Para deploy apenas do frontend (apontamento ajustado para outro backend):

```bash
./deploy-terraform.sh --deploy-frontend-only --frontend-api-url https://backend-outro-cloud.com
```

## Referências

- Status e roadmap: [../roadmap.md](../roadmap.md); indice infra: [index.md](./index.md)
- Terraform root: `infrastructure/terraform/main.tf`, `variables.tf`
- Módulos: `infrastructure/terraform/modules/{aws,azure,gcp}/`
