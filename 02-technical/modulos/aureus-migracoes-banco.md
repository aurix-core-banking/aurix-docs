# Migrações de Banco de Dados (Flyway)

## Visão Geral

O Aurix usa **Flyway** para versionar e gerenciar o schema do banco de dados
compartilhado (`aurix_db`), que é usado por todos os microsserviços.

Antes desta implementação, o schema era criado/atualizado manualmente via
`spring.jpa.hibernate.ddl-auto` (cada serviço fazia `update`/`validate` sobre o
mesmo banco), o que gerava inconsistência entre ambientes e nenhuma
rastreabilidade das mudanças.

Com o Flyway:

- as migrações ficam versionadas no repositório (`db/migration` do `aurix-shared`);
- o schema é criado de forma determinística e idempotente em qualquer ambiente;
- a execução é registrada na tabela `flyway_schema_history` (dentro do schema `aurix`);
- o schema é **validado** no startup (`validate-on-migrate`).

## Como está configurado

- As dependências `flyway-core` e `flyway-database-postgresql` (necessária para
  suporte a PostgreSQL no Flyway 10+) estão no POM do **`aurix-shared`**, portanto
  chegam transitivamente em todos os serviços que dependem dele.
- Os scripts de migração ficam em:
  `aurix-shared/src/main/resources/db/migration/`
- Os scripts de rollback ficam em:
  `aurix-shared/src/main/resources/db/rollback/`
- Todo serviço com `application.yml` carrega a seguinte configuração:

  ```yaml
  spring:
    flyway:
      enabled: true
      validate-on-migrate: true
      validate-migration-naming: true
      locations: classpath:db/migration
      schemas: aurix
  ```

- Em **testes** (`application-test.yml`) o Flyway é desabilitado
  (`spring.flyway.enabled: false`) porque o H2 usa `ddl-auto: create-drop`
  (o H2 não suporta o tipo `jsonb` e o Hibernate já cria o schema de teste).

### Por que no `aurix-shared`?

Todos os serviços compartilham o mesmo banco (`aurix_db` no docker-compose, ver
`aurix-infrastructure/docker-compose.yml`) e o `aurix-shared` é dependência de
todos. Centralizar as migrações ali garante que qualquer serviço consiga aplicar
e validar o schema completo. O Flyway usa `pg_advisory_lock` do PostgreSQL, então
a execução concorrente de migrações por vários serviços é segura (o primeiro
aplica, os demais validam).

## Migração inicial (V1)

A migração `V1__schema_inicial.sql` contém o DDL de **todas** as entidades JPA
do projeto (aurix-shared + todos os `svc-*`), gerado automaticamente a partir
das próprias entidades via Hibernate (dialeto PostgreSQL).

Pontos importantes do schema gerado:

- Todas as tabelas das entidades que declaram `@Table(schema = "aurix")` são
  criadas no schema `aurix`. O `CREATE SCHEMA IF NOT EXISTS aurix` está no início
  da migração.
- Algumas entidades históricas (ex.: `com.aurix.platform.banking.entity.*`) não
  declaram schema e ficam no schema `public` do banco. Mantidas como estão para
  preservar o comportamento atual do Hibernate.
- Entidades que mapeiam a **mesma tabela física** (ex.: `clientes` definido no
  `aurix-shared` e no `svc-customer`; `liquidez` no `svc-cambio` e no
  `svc-banking`) tiveram suas colunas unificadas em uma única tabela.

O rollback correspondente é
`db/rollback/V1__schema_inicial_rollback.sql` (`DROP TABLE ... CASCADE` de todas
as tabelas, em ordem reversa).

## Como criar uma nova migração

1. **Nunca edite uma migração já aplicada** (nem `V1`). Crie uma nova versão.
2. Nomeie o arquivo seguindo o padrão do Flyway: `V<número>__<descricao>.sql`
   (ex.: `V2__adicionar_coluna_score_credito.sql`), dentro de
   `aurix-shared/src/main/resources/db/migration/`.
3. Escreva o DDL manualmente ou gere a partir do Hibernate. Para gerar a partir
   das entidades:
   - mude temporariamente `ddl-auto` de um serviço para `create` em um banco
     local e capture o DDL, ou
   - use a ferramenta de geração de schema (Hibernate `SchemaCreator`).
4. **Sempre** escreva o rollback correspondente em
   `aurix-shared/src/main/resources/db/rollback/` (`V2__<descricao>_rollback.sql`).
5. Execute as migrações localmente com PostgreSQL antes de abrir o PR:
   ```bash
   cd aurix-backend
   ./mvnw -pl aurix-shared install -DskipTests
   # suba o postgres e rode um serviço que aponta para aurix_db, ex. svc-customer:
   ./mvnw spring-boot:run -pl svc-customer
   ```
   Confira o resultado em:
   ```sql
   SELECT * FROM aurix.flyway_schema_history ORDER BY installed_rank;
   ```
6. Documente a mudança de schema aqui neste arquivo (seção de changelog).

## Regras e boas práticas

- **Banco único**: como todos os serviços compartilham `aurix_db`, alterações em
  tabelas do `aurix-shared` afetam todos os serviços. Seja conservador.
- **Colunas novas**: prefira adicionar colunas `NULLABLE` quando não houver valor
  default definido em todas as entidades que usam a tabela.
- **Nomenclatura**: nomes de tabela e coluna em `snake_case` (exceto heranças
  históricas); enums gravados como `VARCHAR` via `@Enumerated(EnumType.STRING)`.
- **Chaves/índices**: crie índices para as buscas mais comuns dentro da migração.
- **Nunca** use `DROP`/`ALTER` sem o respectivo rollback e sem testar em banco
  real.
- **Testes**: não dependa de migração nos testes (H2 + `create-drop`); valide a
  migração em PostgreSQL.

## Validação no startup

Com `spring.flyway.validate-on-migrate: true`, o Flyway compara a soma de hash
das migrações aplicadas com os arquivos presentes. Se um arquivo aplicado for
alterado, o serviço **não inicia** e lança erro, evitando drift de schema entre
ambientes.

Serviços que usam `spring.jpa.hibernate.ddl-auto: validate` (ex.: svc-cards,
svc-compliance, svc-credit, svc-fraud) ainda validam as entidades contra o banco
depois que o Flyway executa.

## Changelog de migrações

| Versão | Descrição |
|--------|-----------|
| V1 | Schema inicial completo (todas as entidades JPA do projeto) |
