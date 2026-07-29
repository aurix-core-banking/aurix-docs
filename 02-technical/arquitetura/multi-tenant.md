# Tenant no AURIX (single por padrao, multi-tenant como alternativa)

O padrao de operacao e **um unico tenant** por instalacao: uma instituicao, um banco de dados, tenant `default`. O header `X-Tenant-Id` pode ser omitido ou enviado como `default`. **Multi-tenant** e uma **alternativa** para cenarios em que o mesmo deploy atende multiplas instituicoes (SaaS, BaaS, multi-filial); nesses casos o isolamento por tenant e obrigatorio.

## Politica de banco de dados

- **Self-hosted (unico banco)**: quando a instituicao hospeda a plataforma para si mesma, e aceitavel e esperado **um unico banco de dados** (um unico SGBD). O uso de `tenant_id` serve para filiais, unidades de negocio ou ambientes (ex.: default, homolog) dentro da mesma instituicao. Nao ha exigencia de banco separado por tenant nesse cenario.
- **Multi-tenant (SaaS)**: quando a oferta for **multi-tenant** (multiplas instituicoes como clientes do mesmo deploy), cada tenant deve ter **banco de dados totalmente apartado**. Ou seja: um banco (ou instancia/namespace) por instituicao; nenhum compartilhamento de tabelas ou SGBD entre tenants. O provisioning (item 8.3 do roadmap) deve prever a criacao de banco dedicado por novo tenant e o roteamento das conexoes por tenant.

Resumo: **self-hosted = um banco/SGBD; multi-tenant SaaS = banco totalmente separado por tenant.**

## Modelo (identificacao e uso de tenant_id)

- **tenant_id**: identificador do tenant (String, ate 64 caracteres). Convencao: UUID ou slug (ex.: `default`, `banco-alfa`).
- **Header**: `X-Tenant-Id` em toda requisicao HTTP. Ausencia ou vazio usa tenant `default`.
- **Entidades**: Todas as entidades que estendem `BaseEntity` possuem campo `tenantId`. Contas, clientes e transacoes sao sempre filtrados por tenant.

## Componentes

### TenantContext (aurix-shared)

- `TenantContext.setTenantId(String)` / `getTenantId()` / `clear()`
- Constantes: `HEADER_TENANT_ID = "X-Tenant-Id"`, `DEFAULT_TENANT_ID = "default"`
- ThreadLocal; limpo apos cada request.

### TenantFilter (aurix-shared)

- Filtro Servlet que le o header `X-Tenant-Id` e chama `TenantContext.setTenantId(...)` antes da cadeia; no `finally` chama `TenantContext.clear()`.
- Registrado em **aurix-core** via `TenantConfig` (FilterRegistrationBean, ordem mais alta). Outros modulos que precisem de tenant devem registrar o mesmo filtro.

### Gateway

- O Spring Cloud Gateway repassa os headers da requisicao para os backends. O cliente envia `X-Tenant-Id` e o core (e demais servicos) recebem o valor.

### Repositorios e services

- **ContaRepository**: `findByTenantIdAndId`, `findByTenantId`, `findByTenantIdAndNumeroConta`, `findByTenantIdAndClienteId`, `findContasAtivasByTenantIdAndClienteId`, `existsByTenantIdAndNumeroConta`.
- **ClienteRepository**: `findByTenantIdAndId`, `findByTenantId`, `findByTenantIdAndCpf`, `findByTenantIdAndEmail`, `existsByTenantIdAndCpf`, `existsByTenantIdAndEmail`, `findByTenantIdAndStatus`.
- **TransacaoRepository**: `findByTenantIdAndId`, `findByTenantId`, `findByTenantIdAndCodigoTransacao`.
- **ContaService, ClienteService, TransacaoService**: ao criar entidade, setam `entity.setTenantId(TenantContext.getTenantId())`; nas buscas e atualizacoes usam os metodos com tenant.

### Unicidade por tenant

- **Conta**: constraint unica em `(tenantId, numeroConta)`.
- **Cliente**: constraints unicas em `(tenantId, cpf)` e `(tenantId, email)`.

## Banco de dados

- **Self-hosted**: coluna `tenant_id` (VARCHAR(64), nullable) nas tabelas que usam BaseEntity. Um unico banco/SGBD; filtro por tenant_id nas queries. Script de migracao: `infrastructure/data-stack/init-scripts/02-add-tenant-id.sql`. Com `spring.jpa.hibernate.ddl-auto: update`, o Hibernate adiciona a coluna ao subir a aplicacao.
- **Multi-tenant (SaaS)**: conforme politica acima, cada tenant possui banco proprio. A aplicacao (ou gateway de dados) roteia a conexao JDBC/contexto por tenant; nao ha compartilhamento de tabelas entre instituicoes. Implementacao no provisioning (roadmap 8.3).

## Uso

**Padrao (single tenant):** Sem header ou com `X-Tenant-Id: default`. Uma unica instituicao; todos os dados pertencem ao tenant `default`.

**Multi-tenant (alternativa):** Cliente ou gateway envia `X-Tenant-Id: <identificador>` (ex.: `banco-alfa`). O Core e os modulos com TenantFilter leem o tenant e isolam leitura/escrita por esse valor.

## Cenarios em que o multi-tenant e alternativa util

- **AURIX Cloud (SaaS)**: um unico operador (voces) oferece core banking como servico para varias instituicoes (bancos digitais, fintechs, cooperativas). Cada instituicao e um tenant; um mesmo deploy atende a todos, com banco separado por tenant. Custos de infra e desenvolvimento sao diluidos; time-to-market para novo cliente e baixo (provisioning + config).
- **White label / BaaS**: parceiros (empresas que nao sao bancos) oferecem conta, PIX ou cartao usando a plataforma por baixo. Cada parceiro ou cada "marca" pode ser um tenant: isolamento de dados e de config (tarifas, limites, termos) sem deploy separado.
- **Multi-filial ou multi-unidade de negocio**: uma unica instituicao com varias filiais ou unidades (ex.: cooperativa com varias cooperativas singulares). Um tenant por filial/unidade no mesmo banco (self-hosted) permite relatorios e operacao por unidade sem misturar dados.
- **Ambientes por tenant**: em desenvolvimento ou homologacao, usar tenants como ambientes (ex.: `default`, `homolog`, `cliente-demo`) no mesmo banco para testar fluxos e integracoes sem criar multiplas instalacoes.

**Padrao adotado:** uma unica instituicao que hospeda a plataforma para si (self-hosted, um banco, um tenant). Usar sempre o tenant `default`; o header `X-Tenant-Id` pode ser omitido ou enviado como `default`.

## Consideracoes regulatorias

A politica adotada (self-hosted = um banco; multi-tenant = banco apartado por tenant) atende ao risco regulatorio:

- **Multi-tenant**: banco totalmente separado por tenant elimina compartilhamento fisico de dados entre instituicoes, alinhado a exigencias de segregacao (BACEN), LGPD e contratos que exijam isolamento.
- **Self-hosted**: um unico SGBD para a propria instituicao e aceitavel; tenant_id continua util para filiais ou unidades de negocio no mesmo banco.
