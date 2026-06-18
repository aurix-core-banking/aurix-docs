# ADR-0002: JPA para cadastro, SQL nativo/locking explícito para fluxos financeiros

**Status**: Aceito
**Data**: 2026-06-17

---

## Contexto

A plataforma usa JPA/Hibernate (Spring Data JPA) de forma uniforme em todos os 27+ módulos, sem distinção entre
entidades de cadastro (cliente, empresa, produto) e operações que movem dinheiro (débito/crédito de saldo,
liquidação, PIX). Ao construir e depurar o skill de execução do `aureus-core` (ver `docs/wiki/01-guides-checklists/checklists/plataforma-completude.md`), apareceram bugs concretos que expõem onde essa uniformidade já causou problemas reais, e a leitura dos serviços financeiros mostrou um padrão de concorrência que é uma falha de design, não só de implementação.

**Bugs já encontrados causados por abstração do JPA mal usada** (corrigidos durante a construção do skill de run):
- `OutboxEvent.payload` e `Transacao.dadosPix`/`dadosTed`: campos `String` com `@Column(columnDefinition =
  "jsonb")` mas sem `@JdbcTypeCode(SqlTypes.JSON)` — Hibernate vinculava o parâmetro como `varchar`, e o
  Postgres rejeitava a query. Isso bloqueava **toda criação de conta e de transação** sem nenhum teste
  detectando (porque não há testes nesses módulos).

**Padrão de concorrência sem proteção real nos fluxos que movem dinheiro:**
- `aureus-core/.../service/ContaService.java` (`atualizarSaldo`, `atualizarLimiteUtilizado`): lê a `Conta`, faz
  `setSaldo(novoSaldo)`, salva. Não há `@Lock(PESSIMISTIC_WRITE)` nem SQL atômico (`UPDATE ... SET saldo =
  saldo - ? WHERE ...`). A única rede de segurança é o `@Version` herdado de `BaseEntity` (optimistic locking
  automático do Hibernate) — protege contra *lost update* dentro do próprio `aureus-core`, mas lança excessão
  sob concorrência em vez de resolver o conflito, e nada no código trata/retenta esse erro.
- `aureus-settlement/.../service/SettlementService.java` (`validarSaldoDisponivel` + `atualizarSaldoConta`,
  linhas ~175-238): clássico TOCTOU (time-of-check-to-time-of-use) — valida saldo disponível, e só depois,
  numa chamada separada, debita/credita. Entre as duas chamadas duas liquidações concorrentes podem aprovar
  com base no mesmo saldo e gerar saldo negativo sem detecção. `SaldoConta` (a entidade que guarda esse saldo)
  não estende `BaseEntity`, mas declara seu próprio campo `@Version private Long versao` — então o optimistic
  locking básico existe (correção a este ADR: a versão original deste documento afirmou que não existia), mas
  isso não fecha a janela de TOCTOU nem evita o problema mais grave encontrado ao ler o código completo: o
  método que chama `atualizarSaldos` o fazia **incondicionalmente**, mesmo quando a liquidação era rejeitada
  por regra de negócio (ex.: TED fora do horário de funcionamento) — ou seja, dinheiro era debitado/creditado
  mesmo em operações rejeitadas.
- `SaldoConta` (`aureus-settlement`) e `Conta.saldo` (`aureus-core`, usado por `aureus-pix`) são **fontes de
  saldo completamente separadas e não sincronizadas** — o mesmo tipo de duplicação de entidade já identificado
  na auditoria de completude da plataforma (`ConciliacaoBancaria` e `Cliente` duplicados entre módulos).
- `aureus-pix/.../service/PixTransferenciaService.java` (`processarTransferencia`): muda o status da
  `PixTransferencia` para `PROCESSADA` e publica um evento, mas **nunca chama nada que debite a conta origem
  ou credite o destino**. Não é um problema de locking — é a ausência completa da movimentação de saldo no
  PIX, reforçando o achado já registrado no checklist de completude ("PIX é simulação").

Esses três pontos (core, settlement, pix) são exatamente os candidatos que o usuário pediu para mapear.

## Decisão

**Não substituir JPA na plataforma inteira.** A maioria dos módulos é CRUD de cadastro (cliente, empresa,
produto, configuração) onde JPA é produtivo e os bugs encontrados foram de uso incorreto (anotação faltante),
não limitação inerente do JPA — trocar tudo por SQL nativo custaria meses de retrabalho sem ganho proporcional.

**Para os fluxos que debitam/creditam saldo ou decidem com base em saldo, usar acesso a dados explícito em vez
de read-modify-write via JPA:**

1. **Atualização atômica via SQL.** Substituir o padrão `findById` → `setSaldo(x)` → `save()` por uma única
   instrução `UPDATE conta SET saldo = saldo - :valor WHERE id = :id AND saldo >= :valor` (via
   `@Modifying @Query` do Spring Data ou `JdbcTemplate`), verificando o número de linhas afetadas para detectar
   saldo insuficiente — elimina a janela de TOCTOU porque a validação e a escrita são a mesma operação atômica
   no banco, não duas chamadas separadas da aplicação.
2. **Onde múltiplas linhas precisam ser lidas e decididas antes de escrever** (ex.: liquidação que depende de
   regras além de saldo simples), usar `@Lock(LockModeType.PESSIMISTIC_WRITE)` no `findById` do Spring Data
   (gera `SELECT ... FOR UPDATE`) para serializar o acesso àquela linha durante a transação, em vez de confiar
   em optimistic locking + exceção.
3. **Toda entidade que representa saldo precisa de `@Version`** (estender `BaseEntity` ou declarar o campo
   diretamente, como `SaldoConta` já faz) como segunda camada de proteção, mesmo quando a atualização já é
   atômica.
4. **Uma única fonte de saldo por conta.** Eliminar a duplicação `Conta.saldo` (core) vs `SaldoConta`
   (settlement) — settlement deve consumir/atualizar o saldo do core (via evento ou chamada), não manter sua
   própria cópia divergente. Isso depende da consolidação de domínio já apontada na seção 12 do checklist de
   completude (ERP/duplicação de entidades).
5. **PIX precisa implementar a movimentação de saldo de fato** como parte da composição dos itens acima —
   hoje não há nada para corrigir a concorrência porque não há nenhuma escrita de saldo a proteger.

Esse acesso explícito (`@Modifying @Query` nativo ou `JdbcTemplate`) convive na mesma base de código com JPA
normal para as demais entidades — não é uma migração de framework, é uma escolha de ferramenta por tipo de
operação dentro dos mesmos módulos.

## Consequências

**Positivas**
- Elimina a classe de bug mais grave para um banco (saldo inconsistente/negativo sob concorrência) com mudança
  localizada em poucos métodos, não uma reescrita.
- `UPDATE ... WHERE saldo >= :valor` é auditável e testável de forma direta (basta verificar linhas afetadas),
  mais simples de raciocinar sob concorrência do que confiar em exceções de optimistic locking.

**Negativas / trade-offs**
- Duas formas de acessar dados convivendo no mesmo módulo (JPA para entidade, SQL nativo para o saldo da mesma
  entidade) exige disciplina para não voltar a usar `save()` ingenuamente no campo protegido — vale um teste
  de regressão específico para isso por módulo afetado.
- `@Lock(PESSIMISTIC_WRITE)` aumenta contenção sob alto volume (linhas bloqueadas até o fim da transação);
  aceitável para o volume atual, mas revisar se o produto escalar para alto throughput de PIX.
- Consolidar `SaldoConta` e `Conta.saldo` numa única fonte é uma migração de dado real (não só código),
  precisa de plano de migração coordenado com a seção 12 do checklist de completude.

## Alternativas consideradas

- **Manter JPA puro com `@Version` em tudo e retry automático em `OptimisticLockException`**: mais simples de
  implementar (decorator/aspect genérico de retry), mas sob alta contenção em uma única conta (ex.: conta de
  liquidação central) gera muitas tentativas descartadas; para o padrão de acesso de uma conta bancária
  (poucas escritas concorrentes por conta individual, não um hot-row global) isso seria aceitável como
  *complemento* ao `@Version` já existente, mas não resolve o TOCTOU de `SettlementService` nem a ausência de
  `@Version` em `SaldoConta` — por isso a decisão prioriza a atualização atômica como primeira linha de defesa.
- **Migrar tudo para SQL nativo/JDBC, abandonando JPA**: rejeitado — custo de reescrita desproporcional ao
  problema; a maioria dos módulos não tem o padrão de concorrência problemático.
- **Event sourcing para saldo (saldo = soma de eventos, nunca um campo mutável)**: mais robusto a longo prazo e
  combina bem com o outbox/Kafka já adotado no [ADR-0001](0001-comunicacao-entre-servicos.md), mas é uma
  mudança de modelo de dados maior; registrar como candidato para revisão futura, não decisão deste ADR.

## Plano de adoção por módulo/fluxo

| Módulo / fluxo | Situação atual | Ação | Status |
|---|---|---|---|
| `aureus-core` `ContaService.atualizarSaldo` / `atualizarLimiteUtilizado` | Read-modify-write, só `@Version` como rede de segurança | Trocar por `UPDATE` atômico condicional (`saldo >= :valor` no débito) | ✅ Feito — `ContaRepository.debitarSaldoAtomico`/`creditarSaldoAtomico`/`utilizarLimiteAtomico`/`liberarLimiteAtomico` |
| `aureus-settlement` `SettlementService.validarSaldoDisponivel` + `atualizarSaldoConta` | TOCTOU explícito; saldo movimentado mesmo em liquidação rejeitada | Unificar validação+escrita numa operação atômica; só movimentar saldo quando status final for LIQUIDADO; constraint única (`conta`, `data_referencia`) | ✅ Feito — `SaldoContaRepository.debitarSaldoAtomico`/`creditarSaldoAtomico`, `SettlementService.movimentarSaldos` |
| `aureus-settlement` `SaldoConta` vs `aureus-core` `Conta.saldo` | Fontes de saldo duplicadas e dessincronizadas | Consolidar em uma única fonte (acompanhar seção 12 do checklist de completude) | ⏳ Pendente — é migração de dado real, não só código |
| `aureus-pix` `PixTransferenciaService.processarTransferencia` | Não debita/credita saldo nenhum | Implementar a movimentação real usando o padrão de atualização atômica acima como parte da implementação, não depois | ✅ Feito — debita origem atomicamente, credita destino se a chave PIX resolver para conta local |
| `aureus-credit`, `aureus-treasury` | Não auditado neste ADR | Revisar com o mesmo critério (debita/credita saldo ou decide com base em saldo?) antes de assumir que está protegido | ⏳ Pendente |

[Voltar ao índice de ADRs](README.md)
