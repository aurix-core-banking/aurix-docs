# Módulo Poupança (aureus-poupanca)

Conta de depósito de poupança — aniversário, crédito automático de TR, IOF na quebra,
extrato PDF, integração com PIX para saque/depósito.

## Stack

| Camada | Tecnologia |
|---|---|
| Framework | Spring Boot 4.1.0 / Spring Framework 7 |
| Linguagem | Java 25 |
| Persistência | JPA + Hibernate 6 + PostgreSQL |
| Cache | Redis (via `spring-boot-starter-data-redis`) |
| Mensageria | Kafka (eventos de transação) |
| Cliente HTTP | `@HttpExchange` (nativo Spring 7, **sem Feign**) |
| Resiliência | `@Retryable` nativo + `@ConcurrencyLimit` + `@CircuitBreaker` (Feign) |
| Observabilidade | `@Observed` + Micrometer + Prometheus |
| Docs API | springdoc-openapi 2.2 |
| Testes | `RestTestClient` (unificado) + Testcontainers + H2 |
| Null Safety | `@NullMarked` (package-level) + `@Nullable` |

## Portas

- **`8111`** — Gateway route `/api/poupanca/*`

## Dependências (pom.xml)

```xml
<parent>
    <groupId>com.aureus.platform</groupId>
    <artifactId>aureus-platform</artifactId>
    <version>1.0.0</version>
</parent>

<artifactId>aureus-poupanca</artifactId>

<dependencies>
    <dependency>
        <groupId>com.aureus.platform</groupId>
        <artifactId>aureus-shared</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>2.2.0</version>
    </dependency>

    <!-- Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers-junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

> **Nota:** `spring-retry` **não** é necessário — Spring Framework 7 inclui `@Retryable` nativo em `spring-core`.  
> `@HttpExchange` é parte do `spring-web`, já incluso no starter web.

## Entidades

### `ContaPoupanca`

| Campo | Tipo | Notas |
|---|---|---|
| `id` | `Long` (PK auto) | |
| `clienteId` | `Long` | FK lógica para `aureus-core.cliente` |
| `contaCorrenteId` | `Long` | FK lógica para `aureus-core.conta` (conta corrente atrelada) |
| `numeroConta` | `String` (unique) | Gerado via `TransacaoUtil.gerarCodigoPix()` |
| `saldo` | `BigDecimal` | |
| `aniversarioDia` | `int` | Dia do mês para crédito de rendimento |
| `dataAbertura` | `LocalDate` | |
| `ultimoAniversario` | `LocalDate` | Último crédito de TR |
| `status` | `Enum(ATIVA, BLOQUEADA, ENCERRADA)` | |
| `tenantId` | `String` | |

### `MovimentacaoPoupanca`

| Campo | Tipo | Notas |
|---|---|---|
| `id` | `Long` (PK auto) | |
| `contaPoupancaId` | `Long` | FK |
| `tipo` | `Enum(DEPOSITO, SAQUE, RENDIMENTO_TR, ESTORNO)` | |
| `valor` | `BigDecimal` | |
| `saldoAnterior` | `BigDecimal` | |
| `saldoPosterior` | `BigDecimal` | |
| `dataMovimentacao` | `LocalDateTime` | |
| `descricao` | `String` | |
| `transacaoOrigemId` | `Long` | Referência à transação origem (PIX, TED, etc) |
| `tenantId` | `String` | |

## API REST

### `/api/poupanca/contas`

| Método | Path | Descrição |
|---|---|---|
| `POST` | `/` | Criar poupança (atrelada a uma conta corrente) |
| `GET` | `/{id}` | Saldo e dados da conta |
| `GET` | `/cliente/{clienteId}` | Listar contas do cliente |
| `PATCH` | `/{id}/bloquear` | Bloquear |
| `PATCH` | `/{id}/encerrar` | Encerrar |

### `/api/poupanca/movimentacoes`

| Método | Path | Descrição |
|---|---|---|
| `POST` | `/deposito` | Depositar (de conta corrente → poupança) |
| `POST` | `/saque` | Sacar (poupança → conta corrente) |
| `GET` | `/conta/{contaId}` | Extrato por conta |
| `GET` | `/conta/{contaId}/periodo` | Extrato por período |

### `/api/poupanca/aniversario`

| Método | Path | Descrição |
|---|---|---|
| `POST` | `/processar` | Processa aniversários do dia (job manual) |
| `GET` | `/proximo` | Próximo rendimento estimado |

### `/api/poupanca/extrato`

| Método | Path | Descrição |
|---|---|---|
| `GET` | `/conta/{contaId}/pdf` | Download extrato PDF (últimos 12 meses) |

## Fluxos Críticos

### Depósito (conta corrente → poupança)

```
POST /api/poupanca/movimentacoes/deposito
Body: { contaPoupancaId, valor }

1. Valida conta poupança ativa
2. @ConcurrencyLimit(1) → serializa concorrência
3. @Retryable → retry em OptimisticLock / transient
4. Chama @HttpExchange ContaCorrenteClient.debitar(valor)
5. Se débito OK: saldo += valor, salva MovimentacaoPoupanca
6. Publica Kafka: poupanca-deposito-realizado
```

### Saque (poupança → conta corrente)

```
POST /api/poupanca/movimentacoes/saque
Body: { contaPoupancaId, valor }

1. Valida conta poupança ativa
2. Se data < aniversário + 30d: calcula IOF sobre rendimento via @HttpExchange TaxClient.calcularIof()
3. @ConcurrencyLimit(1)
4. @Retryable
5. Saldo -= valor (+ IOF se aplicável)
6. Chama @HttpExchange ContaCorrenteClient.creditar(valor)
7. Salva MovimentacaoPoupanca
8. Publica Kafka: poupanca-saque-realizado
```

### Aniversário (crédito de TR)

```
Job agendado @Scheduled(cron = "0 0 2 * * *")
Processa todas as contasPoupanca com aniversarioDia == today

Para cada conta:
1. @Retryable
2. Consulta TR do dia via @HttpExchange BacenClient.buscarTrDiaria()
3. rendimento = saldo * (TR / 100)
4. saldo += rendimento
5. ultimoAniversario = today
6. Salva MovimentacaoPoupanca(tipo = RENDIMENTO_TR)
7. Publica Kafka: poupanca-rendimento-creditado
```

## Clientes HTTP (Spring 7 `@HttpExchange`)

### `ContaCorrenteClient` (aureus-core)

```java
@HttpExchange("/api/core/contas")
public interface ContaCorrenteClient {
    @PostExchange("/{id}/debitar")
    @Retryable(maxRetries = 3, delay = 100, multiplier = 2)
    void debitar(@PathVariable Long id, @RequestBody DebitoRequest request);

    @PostExchange("/{id}/creditar")
    @Retryable(maxRetries = 3, delay = 100, multiplier = 2)
    void creditar(@PathVariable Long id, @RequestBody CreditoRequest request);
}
```

### `TaxClient` (aureus-tax)

```java
@HttpExchange("/api/tax")
public interface TaxClient {
    @PostExchange("/iof/calcular")
    IofResponse calcularIof(@RequestBody IofRequest request);
}
```

### `BacenClient` (aureus-bacen)

```java
@HttpExchange("/api/bacen")
public interface BacenClient {
    @GetExchange("/indicadores/tr/{data}")
    @Retryable(maxRetries = 3, delay = 200, multiplier = 2, jitter = 50, maxDelay = 2000)
    BigDecimal buscarTrDiaria(@PathVariable String data);
}
```

Registrados via `@ImportHttpServices` na classe de configuração:

```java
@Configuration
@ImportHttpServices(classes = {ContaCorrenteClient.class, TaxClient.class, BacenClient.class})
@EnableResilientMethods
public class PoupancaHttpConfig {
}
```

## Resiliência

| Anotação | Onde | Parâmetros |
|---|---|---|
| `@EnableResilientMethods` | `PoupancaHttpConfig` | Ativa `@Retryable` e `@ConcurrencyLimit` |
| `@Retryable` | Depósito, Saque, Aniversário, Clientes HTTP | `maxRetries=3, delay=100, multiplier=2, jitter=50, maxDelay=2000` |
| `@ConcurrencyLimit(1)` | `depositar()`, `sacar()` | policy default `BLOCK` |
| `@CircuitBreaker` | (apenas se migrar para Feign futuramente) | |

## Observabilidade

```java
@Service
@Observed(name = "poupanca.service", contextualName = "poupança")
public class PoupancaService {
    @Observed(name = "poupanca.deposito")
    public void depositar(...) { ... }
}
```

## Null Safety

`package-info.java` em cada pacote:

```java
@NullMarked
package com.aureus.platform.poupanca.service;
```

## Eventos Kafka

| Tópico | Evento | Payload |
|---|---|---|
| `poupanca-conta-criada` | `ContaPoupancaEvent` | `{ id, clienteId, numeroConta, dataAbertura }` |
| `poupanca-deposito-realizado` | `MovimentacaoEvent` | `{ contaId, valor, saldoPosterior }` |
| `poupanca-saque-realizado` | `MovimentacaoEvent` | `{ contaId, valor, iof, saldoPosterior }` |
| `poupanca-rendimento-creditado` | `RendimentoEvent` | `{ contaId, valor, tr, saldoPosterior }` |

## Estrutura de Pacotes

```
com.aureus.platform.poupanca/
  config/
    PoupancaHttpConfig.java       (@EnableResilientMethods, @ImportHttpServices)
    PoupancaKafkaConfig.java
    PoupancaSecurityConfig.java
    PoupancaRedisConfig.java
  controller/
    ContaPoupancaController.java
    MovimentacaoController.java
    AniversarioController.java
    ExtratoController.java
  service/
    ContaPoupancaService.java
    MovimentacaoService.java
    AniversarioService.java
    ExtratoPdfService.java
  client/
    ContaCorrenteClient.java      (@HttpExchange)
    TaxClient.java                (@HttpExchange)
    BacenClient.java              (@HttpExchange)
  entity/
    ContaPoupanca.java
    MovimentacaoPoupanca.java
  repository/
    ContaPoupancaRepository.java
    MovimentacaoPoupancaRepository.java
  dto/
    CriarContaRequest.java
    DepositoRequest.java
    SaqueRequest.java
    ContaPoupancaResponse.java
    ExtratoResponse.java
  event/
    ContaPoupancaEvent.java
    MovimentacaoEvent.java
    RendimentoEvent.java
```

## Extrato PDF

- Gerado via `aureus-shared` (biblioteca de PDF compartilhada)
- Últimos 12 meses de movimentações
- Cabeçalho: banco, agência, conta, cliente, período
- Tabela: data, descrição, valor, saldo
- Rodapé: saldo atual, rendimento acumulado no período
- Endpoint: `GET /api/poupanca/extrato/conta/{contaId}/pdf`
- Cache Redis (TTL 1h) do PDF gerado para evitar regerar na mesma consulta

## Testes

| Tipo | Tecnologia | O que cobre |
|---|---|---|
| Unitário | JUnit 5 + Mockito | Services com mocked clients |
| Integração REST | `RestTestClient` | Endpoints completos com H2 |
| Repository | `@DataJpaTest` + H2 | Queries JPA |
| Contrato | `@HttpExchange` test | Mock `RestClient` para clients HTTP |
| Container | Testcontainers | PostgreSQL real para queries complexas |

## application.yml

```yaml
server:
  port: 8111
  servlet:
    context-path: /api/poupanca

spring:
  application:
    name: aureus-poupanca
  profiles:
    active: dev
  datasource:
    url: jdbc:postgresql://localhost:5432/aureus
    username: aureus
    password: aureus123
    driver-class-name: org.postgresql.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
        jdbc:
          time_zone: America/Sao_Paulo
    open-in-view: false
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2000ms
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: aureus-poupanca-group
      auto-offset-reset: earliest
    producer:
      acks: all
      retries: 3

aureus:
  poupanca:
    iof:
      alquota-geral: 0.0046
      aliquota-diaria: 0.000041
      dias-minimos: 30
    tr:
      url: "https://api.bcb.gov.br/dados/serie/bcdata.sgs.11/dados"
    extrato:
      pdf:
        cache-ttl: 3600
    limits:
      max-deposito: 50000.00
      max-saque: 50000.00
