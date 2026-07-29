# Perfil de implantacao (Deployment Profile)

Um unico ponto de decisao deve definir **como** e **onde** o AURIX sera implantado. Esse perfil alimenta o provisioning, os scripts de deploy e a configuracao de infraestrutura. A ordem logica das escolhas e a seguinte.

---

## 1. Quem cuida dessas nuancias

Recomendacao: o **provisioning** (roadmap fase 8) ou um componente de **configuracao de plataforma** centraliza essas opcoes. Pode ser:

- Um modulo/servico (ex.: aurix-provisioning ou parte do controller/admin) que persiste o "perfil de implantacao" e o usa no deploy.
- Ou um artefato de config (ex.: `deployment-profile.yml` ou parametros no pipeline) versionado e aplicado no momento do deploy.

Em ambos os casos, as decisoes abaixo sao feitas **uma vez por implantacao** (ou por tenant, no caso multi-tenant) e nao espalhadas em varios arquivos ou servicos.

---

## 2. Ordem das decisoes

### Passo 1: Modelo de tenancy

- **Unico tenant (padrao)**: uma unica instituicao; um banco/SGBD; tenant `default`. E o cenario tipico (self-hosted ou uma instalacao por cliente).
- **Multi-tenant (alternativa)**: multiplas instituicoes no mesmo operador (SaaS); cada tenant com banco totalmente apartado (politica em [multi-tenant.md](./multi-tenant.md)).

Essa escolha define a politica de banco (um banco unico vs um banco por tenant quando multi-tenant).

### Passo 2: Nuvem ou ambiente de execucao

Onde o sistema sera implantado (cloud-native):

- **AWS** (EKS, RDS, etc.)
- **Azure** (AKS, Azure Database, etc.)
- **Google Cloud** (GKE, Cloud SQL, etc.)
- **OpenStack** ou **nuvem privada** (Kubernetes on-prem ou VMs)

O perfil guarda o provider e os parametros minimos (regiao, rede, credenciais por ambiente). Scripts e templates de IaC (Terraform, CloudFormation, etc.) sao selecionados ou parametrizados a partir dessa escolha.

### Passo 3: Topologia de deploy (como o sistema roda)

- **Monolito modular**: uma ou poucas aplicacoes (ex.: um jar que agrupa core + alguns modulos), um processo ou pod; banco unico por tenant/instalacao.
- **Microservicos (pods separados) com banco compartilhado**: cada servico (core, pix, credit, bacen, etc.) em pod/servico proprio, mas **todos conectados ao mesmo banco de dados** por tenant (ou ao unico banco no self-hosted).

Nao ha terceira opcao obrigatoria no perfil; se no futuro houver "microservicos com banco por servico", isso seria um quarto valor (ex.: `topology: microservices-db-per-service`).

---

## 3. Opcao: microservicos com banco compartilhado

Nessa topologia, os servicos sao implantados em pods/containers separados (escala e deploy independentes por servico), porem **compartilham o mesmo SGBD** (por tenant em multi-tenant, ou unico no self-hosted).

### Vantagens

- **Transacoes ACID** entre dominios que usam o mesmo banco; evita saga e consistencia eventual para fluxos criticos.
- **Um banco por tenant** em multi-tenant: provisioning e backup mais simples que N bancos por tenant.
- **Operacao**: um ponto de backup, restore e tuning por tenant; menos conexoes e menos custo de instancias de banco.
- **Migracoes**: schema unico; Flyway/Liquibase rodam em um lugar; menos coordenacao entre servicos.

### Riscos e quando a estrategia e ruim

- **Acoplamento de schema**: todos os servicos dependem do mesmo esquema. Uma migracao que altera tabela usada por varios servicos exige coordenacao de deploy (ou compatibilidade retroativa). Se as equipes forem muito independentes e mudarem o banco sem combinado, a estrategia piora.
- **Escala do banco**: o gargalo passa a ser o SGBD. Se um servico (ex.: PIX) gerar carga desproporcional, afeta todos. Mitigacao: tuning, replicas de leitura, ou no limite evoluir para banco por servico para os servicos mais criticos.
- **Falha unica**: pane no banco derruba todos os servicos. Aceitavel se o banco for o recurso mais bem protegido (HA, backup, DR) e os servicos forem stateless.
- **Quando e estrategia ruim**: (1) quando o objetivo e independencia total entre times (deploy e schema por servico); (2) quando regulacao ou contrato exigirem isolamento de dados por dominio em bancos distintos; (3) quando a carga for tao assimetrica que um banco compartilhado vire gargalo sem saida. Nesses casos, "microservicos + banco compartilhado" e a escolha errada; preferivel monolito modular ou microservicos com banco por servico.

### Quando e estrategia boa

- Fase de evolucao em que se quer **pods separados** (escala e deploy por servico) mas **sem** a complexidade de multiplos bancos, sagas e eventual consistency.
- Multi-tenant com **banco apartado por tenant** ja atendido: dentro de cada tenant, um unico banco compartilhado pelos servicos e simples e transacional.
- Equipe unica ou coordenada no schema; migracoes versionadas e aplicadas em ordem.

---

## 4. Resumo do perfil

| Campo | Opcoes | Usado por |
|-------|--------|-----------|
| **Tenancy** | `single` (padrao) \| `multi-tenant` | Politica de banco; provisioning |
| **Cloud / ambiente** | `aws` \| `azure` \| `gcp` \| `openstack` \| `private` | IaC, scripts, secrets |
| **Topologia** | `modular-monolith` \| `microservices-shared-db` | Deploy (pods, replicas), conexao JDBC |

O componente que "cuidar dessas nuancias" (provisioning ou config de plataforma) le esse perfil e:

- Em **single (padrao)**: um unico tenant (`default`), um banco; deploy unico ou multiplos pods conforme topologia.
- Em **multi-tenant**: provisiona um banco por tenant; roteia conexao por tenant; deploy dos servicos conforme topologia (um pod ou varios pods, mesmo DB por tenant).
- Em **cloud**: usa templates e credenciais do provider escolhido.

---

## 5. Referencia no roadmap

A fase **8. Provisioning e AURIX Cloud** e o lugar natural para implementar a leitura e aplicacao do perfil de implantacao. Os itens 8.1 a 8.5 podem assumir que o perfil (tenancy, cloud, topologia) ja foi definido antes ou no cadastro da instituicao (8.1), e o 8.3 usa esse perfil para decidir criacao de banco(s) e forma de deploy (monolito vs microservicos com banco compartilhado).
