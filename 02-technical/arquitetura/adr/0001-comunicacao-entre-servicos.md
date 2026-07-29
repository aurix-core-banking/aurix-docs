# ADR-0001: Padrão de comunicação entre serviços/módulos

**Status**: Aceito
**Data**: 2026-06-17

---

## Contexto

A plataforma tem 27+ módulos Spring Boot (`backend/aurix-*`) e hoje a comunicação entre eles é inconsistente, com dois padrões coexistindo de forma parcial e não documentada:

- **Síncrono ad-hoc**: cada módulo consumidor escreve seu próprio client REST manualmente — `CoreApiClient` (`aurix-onboarding`), `CoreApiClientImpl` (`aurix-openfinance`) — sem contrato gerado, sem versionamento, com risco de drift entre o que o client espera e o que o serviço expõe.
- **Assíncrono parcialmente adotado**: existe um `EventHub` em `aurix-shared` (`eventhub/EventHub.java`) com roteamento, retry exponencial, prioridade e Dead Letter Queue — bem desenhado — e um padrão de outbox transacional em `aurix-core` (`OutboxEventPublisher`/`OutboxRelay`). Mas só `aurix-core` e `aurix-pix` de fato publicam eventos hoje. Os módulos `aurix-analytics`, `aurix-audit`, `aurix-compliance`, `aurix-credit`, `aurix-security`, `aurix-treasury` declaram a dependência do Kafka no `pom.xml` mas não publicam nem consomem nada.
- **Consumidores no lugar errado**: `EventListener` (`@KafkaListener` para `conta-criada`, `transacao-realizada`, `imposto-calculado` etc.) vive dentro de `aurix-shared`, uma biblioteca compartilhada — ou seja, lógica de negócio de um domínio específico roda implicitamente em qualquer serviço que dependa de `aurix-shared`, em vez de pertencer ao serviço que de fato é o dono daquele evento.
- **Sem contrato formal**: `aurix-api-specs` tem apenas 1 OpenAPI (`aurix-core.yaml`) para toda a plataforma; não há AsyncAPI para os tópicos Kafka, nem convenção de nome/versionamento de tópico (`conta-criada` é um nome plano, sem domínio ou versão).
- **Gateway só cobre tráfego norte-sul**: `aurix-gateway` faz autenticação por API key e rate limit na borda, mas não há nada padronizado para tráfego leste-oeste (serviço a serviço) — nem mTLS, nem service mesh.

Isso aumenta o risco de inconsistência de dados em fluxos financeiros multi-módulo (PIX → liquidação → contabilidade → fiscal) e de regressão silenciosa quando um client REST escrito a mão fica desatualizado.

## Decisão

### 1. Síncrono (consulta / validação que precisa de resposta imediata)
REST interno via **clients gerados a partir de OpenAPI**, não mais classes de client escritas à mão. Padronizar em `WebClient` (já usado em `aurix-bacen`). Cada módulo publica seu próprio OpenAPI (expandir `aurix-api-specs` para cobrir todos os módulos, não só `aurix-core`); os specs que faltam devem ser gerados a partir dos controllers existentes (springdoc-openapi) e versionados no repositório.

### 2. Assíncrono (mudança de estado / evento de domínio entre módulos)
Kafka via **outbox transacional**, estendendo o padrão já existente em `aurix-core` para os módulos que hoje mudam estado financeiro crítico sem publicar nada: `aurix-pix` (parcial), `aurix-settlement`, `aurix-billing`, `aurix-bacen`, `aurix-treasury`, `aurix-credit`, `aurix-tax`, `aurix-accounting`. Outbox evita dual-write inconsistente (gravar no banco e falhar ao publicar no Kafka) e dá trilha de auditoria nativa — importante em banking.

**Convenção de tópico**: `<dominio>.<entidade>.<evento>.<versao>` (ex.: `core.conta.criada.v1`, `pix.transferencia.liquidada.v1`) em vez dos nomes planos atuais (`conta-criada`). Evita colisão de nomes entre módulos e permite evolução de schema sem quebrar consumidores antigos.

**Consumidores pertencem ao serviço dono do caso de uso**, nunca a `aurix-shared`. Mover a lógica hoje em `EventListener` (aurix-shared) para os serviços que realmente consomem aquele evento (ex.: lógica de cache de conta deveria estar em quem precisa do cache, não centralizada numa lib compartilhada por todos).

### 3. Fluxos multi-módulo com múltiplas etapas
**Saga coreografada via eventos Kafka**, não transação distribuída (2PC). Cada serviço reage a um evento, executa sua etapa local e publica o evento seguinte (ou um evento de compensação em caso de falha). Ex.: PIX liquidado → evento → lançamento contábil → evento → cálculo de imposto. Falha em qualquer etapa publica evento de compensação para desfazer etapas anteriores.

### 4. Contrato
- **OpenAPI** para toda chamada síncrona (`aurix-api-specs/*.yaml`, um arquivo por módulo).
- **AsyncAPI** para todo tópico Kafka publicado, descrevendo schema do evento, versão e produtor — mesmo diretório `aurix-api-specs`.
- Geração de client/DTO no build (Maven plugin de codegen), eliminando classes de client escritas à mão.

### 5. Segurança leste-oeste
Tráfego serviço-a-serviço deve ser protegido por mTLS quando os manifests de Kubernetes forem criados para os módulos que hoje não têm (ver checklist de infraestrutura). Não é bloqueador para adotar os padrões 1-4, mas deve acompanhar a expansão de cobertura de Kubernetes.

## Classificação arquitetural: híbrido, não EDA puro

Vale deixar explícito para não ser reinterpretado depois como "adotamos EDA": esta decisão é uma **arquitetura de microsserviços orientada a eventos no lado de escrita/propagação de estado, com REST request-response no lado de consulta** — não um Event-Driven Architecture (EDA) completo.

- **O que é EDA aqui**: a parte assíncrona (Kafka + outbox + saga coreografada) segue o padrão — serviços publicam eventos de domínio, outros reagem sem chamada direta acoplada.
- **O que não é EDA puro**: consultas e validações que precisam de resposta imediata (ex.: "esse CPF já tem conta?", "qual o saldo agora?") continuam via REST síncrono. Um EDA puro resolveria isso com **CQRS** — cada serviço mantendo uma *view* materializada local, alimentada pelos eventos dos serviços donos, em vez de perguntar sincronamente. Adotar CQRS agora foi descartado por ser complexidade prematura: exigiria manter réplicas de dados de outros domínios em cada serviço (mais armazenamento, mais consistência eventual para tratar, mais superfície operacional) antes de termos resolvido o básico (PIX/settlement ainda simulados — ver checklist de completude). Revisitar quando escala de leitura for um problema real e mensurável, não antecipado.
- **Não é Event Sourcing**: os eventos no Kafka são notificação de mudança de estado (o banco relacional via outbox continua sendo a fonte da verdade), não um log de eventos como única fonte de verdade reconstruível. Event Sourcing é um salto de complexidade maior, fora de escopo desta decisão.

## Consequências

**Positivas**
- Reaproveita 100% da infraestrutura Kafka/outbox/EventHub já construída e paga, em vez de descartar.
- Reduz drift de contrato entre módulos (clients gerados, não escritos a mão).
- Dá trilha de auditoria e capacidade de replay de eventos — relevante para investigação de incidentes e para os relatórios regulatórios (BACEN, SCR) que hoje dependem de dados consistentes entre módulos.
- Desacopla módulos: um módulo lento ou fora do ar não bloqueia sincronamente os demais em fluxos assíncronos.

**Negativas / trade-offs**
- Mais disciplina de contrato (manter OpenAPI/AsyncAPI atualizados) do que simplesmente chamar um endpoint.
- Consistência eventual em vez de imediata nos fluxos assíncronos — exige que cada módulo trate estados intermediários (ex.: "PIX recebido, aguardando lançamento contábil") na sua modelagem.
- Saga coreografada é mais difícil de depurar que uma chamada síncrona direta — requer correlação de eventos por ID de transação/trace.
- Esforço de migração: mover `EventListener` de `aurix-shared` para os serviços donos é uma refatoração não trivial.

## Alternativas consideradas

- **gRPC para todas as chamadas síncronas**: rejeitado por agora — adiciona ferramental e curva de aprendizado sem que latência seja hoje o problema; REST com client gerado resolve o problema real (drift de contrato).
- **Transação distribuída (2PC) para fluxos multi-módulo**: rejeitado — não escala bem com Kafka/microsserviços e adiciona acoplamento forte exatamente onde se quer desacoplar.
- **Manter o status quo (cada módulo decide)**: rejeitado — é a causa raiz da inconsistência atual (alguns módulos com Kafka não usado, clients duplicados, eventos sem padronização de nome).
- **Service mesh completo (Istio) imediatamente**: postergado — a cobertura de Kubernetes hoje é de 1/29 serviços; adotar mTLS/mesh antes de ter manifests para todos os serviços é prematuro. Decisão registrada para revisão quando a cobertura de k8s avançar.

## Plano de adoção por módulo

| Módulo | Situação atual | Ação | Status |
|--------|-----------------|------|--------|
| `aurix-core` | Outbox + Kafka publicando | Referência — manter, alinhar nome de tópico à nova convenção | ⏳ Nome de tópico ainda não padronizado |
| `aurix-pix` | Publica eventos via EventHub | Migrar para outbox transacional; alinhar nome de tópico | ⏳ Pendente |
| `aurix-settlement` | Não publica nada | Adotar outbox; publicar eventos de liquidação | ⏳ Pendente (saldo já corrigido via ADR-0002) |
| `aurix-billing` | Não publica nada | Adotar outbox; publicar evento de fatura emitida/paga | ⏳ Pendente |
| `aurix-bacen` | Não publica nada | Adotar outbox; publicar evento de relatório gerado/enviado | ⏳ Pendente |
| `aurix-treasury` | Dependência Kafka não usada | Adotar outbox ou remover dependência morta | ⏳ Pendente |
| `aurix-credit` | Dependência Kafka não usada | Adotar outbox; publicar evento de decisão de crédito | ⏳ Pendente |
| `aurix-tax` | Não publica nada | Adotar outbox; publicar evento de imposto calculado | ⏳ Pendente |
| `aurix-accounting` | Não publica nada | Consumir eventos de billing/settlement; publicar lançamento contábil | ⏳ Pendente |
| `aurix-analytics` | Dependência Kafka não usada | Tornar-se consumidor (não produtor) dos eventos de negócio | ⏳ Pendente |
| `aurix-audit` | Dependência Kafka não usada | Tornar-se consumidor de todos os eventos relevantes para trilha de auditoria | ⏳ Pendente |
| `aurix-compliance` | Dependência Kafka não usada | Consumir eventos de transação/conta para checagens AML/PLD | ⏳ Pendente |
| `aurix-security` | Dependência Kafka não usada | Avaliar se precisa de eventing ou se é só dependência morta | ⏳ Pendente |
| `aurix-gateway` | Dependência Kafka não usada | Remover dependência — gateway de borda não deveria precisar de broker | ✅ Feito — `spring-kafka`/`spring-kafka-test` removidos do pom |
| `aurix-onboarding` | Client REST escrito à mão (`CoreApiClient`) | Migrar para client gerado a partir do OpenAPI do `aurix-core` | ⏳ Pendente |
| `aurix-openfinance` | Client REST escrito à mão (`CoreApiClientImpl`) | Migrar para client gerado a partir do OpenAPI do `aurix-core` | ⏳ Pendente |
| `aurix-shared` | Hospeda `EventListener` com lógica de negócio | Remover listeners de domínio daqui; mover para o serviço dono de cada evento | ✅ Feito — `EventListener.java` removido; os 3 listeners com lógica real (`conta-criada`, `conta-atualizada`, `transacao-realizada`) foram para `aurix-core/.../event/ContaTransacaoEventListener.java` com lookups reais (antes eram dados fabricados); os outros 5 listeners (`conta-bloqueada`, `transacao-liquidada`, `transacao-conciliada`, `imposto-calculado`, `imposto-registrado`) foram removidos — não tinham nenhum publicador e o corpo era um stub vazio, então não havia nada de real para realocar |

Também corrigido: `EventHub.publishEventWithDelay` e `publishEventWithRetryInternal` usavam `Thread.sleep` real
dentro de `CompletableFuture.runAsync`, bloqueando uma thread do pool compartilhado pela duração inteira do
delay/backoff. Agora usam um `ScheduledExecutorService` dedicado, sem bloquear nenhuma thread durante a espera.

[Voltar ao índice de ADRs](README.md)
