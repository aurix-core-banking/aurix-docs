# ADR-0004: Implementação dos serviços svc-products (catálogo de produtos) e svc-contracts (gestão de contratos)

**Status**: Aceito
**Data**: 2026-08-10

---

## Contexto

O backlog da plataforma registrava duas lacunas de domínio que impediam a contratação automatizada de produtos:

- **Issue #2**: não existia um catálogo de produtos com **regras de elegibilidade** (renda mínima, idade, segmento, não negativado etc.) e **tarifas** (manutenção, transferência, saque, taxa de juros etc.) — a precificação e a simulação de crédito dependiam de cadastro manual e de regras espalhadas nos serviços consumidores.
- **Issue #19**: não existia uma gestão de **contratos** com **assinaturas digitais** (com hash de documento, assinante, IP, user agent) e **templates de contrato** versionados — a assinatura de produtos (empréstimo, financiamento, seguros, consignado, cartão, câmbio) não tinha um dono de domínio.

O `svc-products` existia apenas como placeholder no POM e no docker-compose, e o `svc-contracts` era documentado na visão de arquitetura como "em construção, sem porta definida". A implementação desses dois domínios foi decidida dentro das premissas já consolidadas na plataforma:

- **Banco único compartilhado** (`aurix_db`): não há database-per-service; as tabelas dos novos domínios entram no mesmo banco via migração **Flyway V2**, centralizada em `aurix-shared` (`V2__schema_produtos_contratos.sql`), no schema `aurix`, com **8 tabelas**: `produtos`, `regras_elegibilidade`, `tarifas_produto`, `versoes_produto`, `contratos`, `contratos_assinaturas`, `contratos_versoes`, `templates_contrato`.
- **Eventos de domínio via Kafka**, seguindo a convenção `<dominio>.<entidade>.<evento>.<versao>` do [ADR-0001](0001-comunicacao-entre-servicos.md): tópicos `PRODUTO_*` (`products.produto.criado.v1`, `products.produto.atualizado.v1`, `products.produto.descontinuado.v1`) e `CONTRATO_*` (`contracts.contrato.criado.v1`, `contracts.contrato.assinado.v1`, `contracts.contrato.liquidado.v1`, `contracts.contrato.cancelado.v1`), centralizados em `Topics.java` no `aurix-shared`.
- **Integração síncrona com tolerância a falha**: o `svc-contracts` consulta o perfil do cliente (`svc-customer`) e o catálogo de produtos (`svc-products`) via HTTP (`RestClient` + `HttpServiceProxyFactory`, clients declarativos `ClienteClient`/`ProdutoClient`); a camada `IntegracaoContratoService` envolve cada chamada em `try/catch` e degrada graciosamente para `Optional.empty()` quando o serviço parceiro está indisponível — o contrato é criado em `RASCUNHO` mesmo sem o enriquecimento dos dados do cliente/produto.
- **Testes**: o domínio foi entregue com **47 testes automatizados** (unitários de serviço, de integração e contratos entre serviços).
- A implementação foi consolidada no **PR #25** (https://github.com/aurix-core-banking/aurix-backend/pull/25), que resolve as issues #2 e #19, e o catálogo de endpoints foi publicado na spec `aurix-api-specs` (tags `products` e `contracts`).

## Decisão

Implementar os dois domínios como serviços Spring Boot no monorepo Maven, seguindo o padrão Controller → Service → Repository com entidades JPA e DTOs:

1. **`svc-products`** (porta **8084**) — catálogo de produtos: CRUD de produto com versionamento (`versoes_produto`), regras de elegibilidade avaliáveis (`regras_elegibilidade`), tarifas com periodicidade e vigência (`tarifas_produto`), e avaliação de elegibilidade a partir do perfil do cliente.
2. **`svc-contracts`** (porta **8085**) — gestão de contratos: ciclo de vida do contrato (RASCUNHO → AGUARDANDO_ASSINATURA → ASSINADO → ATIVO → LIQUIDADO/CANCELADO/REJEITADO), assinaturas com hash de documento e trilha de dados do assinante (`contratos_assinaturas`), templates de contrato (`templates_contrato`) e versionamento do contrato (`contratos_versoes`).
3. **Schema compartilhado via Flyway V2** em `aurix-shared` — as 8 tabelas vivem no `aurix_db` no schema `aurix`, evoluídas pela migração `V2__schema_produtos_contratos.sql` (com `rollback` em `db/rollback/`).
4. **Eventos de domínio** `PRODUTO_*` e `CONTRATO_*` publicados no Kafka, permitindo que `svc-fraud`, `svc-compliance`, `svc-platform` e demais consumidores reajam à criação/atualização de produtos e contratos sem acoplamento síncrono.
5. **Integração Cliente/Produto via HTTP com tolerância a falha** — leitura síncrona para enriquecimento do contrato, com degradação graciosa (log + `Optional.empty()`) em vez de falhar a operação quando o parceiro está fora. Coerente com o ADR-0001: REST para consulta/validação, Kafka para propagação de estado.

## Consequências

### Positivas

- Fecha as issues **#2** e **#19** e remove o gap documentado de `svc-contracts` "em construção" na arquitetura.
- Catálogo de produtos com elegibilidade e tarifas versionadas dá base para contratação automatizada, simulação de crédito e precificação consistente entre serviços.
- A tolerância a falha na integração Cliente/Produto mantém a disponibilidade da criação de contratos mesmo com `svc-customer` ou `svc-products` fora do ar (dados de enriquecimento podem ser preenchidos posteriormente).
- Migração V2 centralizada em `aurix-shared` preserva um único versionamento de schema e uma única fonte de verdade das entidades compartilhadas.
- Eventos `PRODUTO_*`/`CONTRATO_*` no Kafka dão trilha de auditoria e permitem reação assíncrona dos demais serviços.

### Negativas / Trade-offs

- **Novos módulos Maven** no build do monorepo: aumenta o tempo de build e a superfície de manutenção de dependências.
- **Migração V2 no schema compartilhado** afeta todos os serviços que usam `aurix-shared` — qualquer mudança nas 8 tabelas deve ser coordenada (mesmo banco, mesmo schema).
- Consistência de enriquecimento eventual: quando `svc-customer`/`svc-products` está indisponível, o contrato é criado sem os dados do cliente/produto, exigindo que o consumidor trate estados intermediários (mesma disciplina do ADR-0001).
- O `svc-contracts` foi incluído no docker-compose (porta 8085) e roteado no gateway (`/api/contracts/**`), com health check E2E — a manutenção desses artefatos passa a ser responsabilidade da equipe de plataforma.

## Alternativas consideradas

- **Database-per-service (banco separado para produtos e contratos)**: rejeitado — contraria a premissa de banco único `aurix_db` já adotada na plataforma.
- **Schemas separados por domínio no mesmo banco**: considerado, mas mantido o schema `aurix` único por consistência com as migrações existentes.
- **Integração Cliente/Produto via eventos Kafka em vez de HTTP síncrono**: descartada para esta leitura — a consulta é pontual e precisa de resposta imediata; HTTP com fallback resolve com menos latência e complexidade, alinhado à classificação híbrida do ADR-0001.
- **Clients REST escritos à mão**: rejeitado — usados clients declarativos via `RestClient` + `HttpServiceProxyFactory`, coerente com o ADR-0001 (sem classes de client duplicadas à mão).
- **Transação distribuída (2PC)** para criar contrato + consultar cliente/produto: não aplicável — as integrações são apenas leitura/enriquecimento, não escrita multi-domínio.

## Plano de adoção por módulo/fluxo

| Módulo / fluxo | Situação anterior | Ação | Status |
|---|---|---|---|
| `svc-products` | Placeholder no POM/docker-compose | Implementar catálogo de produtos, elegibilidade e tarifas | ✅ Feito — porta 8084, health `/actuator/health` |
| `svc-contracts` | "Em construção", sem porta definida | Implementar contratos, assinaturas e templates | ✅ Feito — porta 8085, health `/actuator/health` |
| `aurix-shared` Flyway | Sem schema de produtos/contratos | Migração V2 (`V2__schema_produtos_contratos.sql`) com 8 tabelas no schema `aurix` | ✅ Feito |
| Kafka | Sem eventos de produto/contrato | Tópicos `PRODUTO_*` e `CONTRATO_*` centralizados em `Topics.java` | ✅ Feito |
| Integração Cliente/Produto | Inexistente | `IntegracaoContratoService` com HTTP + tolerância a falha | ✅ Feito |
| Testes | Inexistente | 47 testes automatizados nos dois serviços | ✅ Feito |
| Health checks E2E | Não cobria os novos serviços | Adicionar `svc-products` (8084) e `svc-contracts` (8085) a `SERVICE_HEALTH_ENDPOINTS` | ✅ Feito |
| docker-compose / gateway | `svc-contracts` ausente | Incluir `svc-contracts` no docker-compose e no roteamento do gateway | ✅ Feito — porta 8085, rota `/api/contracts/**` |
| OpenAPI | Spec sem products/contracts | Tags, paths e schemas de products/contracts em `aurix-api-specs` | ✅ Feito |

[Voltar ao índice de ADRs](README.md)
