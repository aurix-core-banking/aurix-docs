# aurix-notification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the aurix-notification microservice — multi-channel notification system with template rendering, Kafka event consumption (cliente.criado, kyc.aprovado, kyc.rejeitado, fraude.transacao.bloqueada), preference management, and REST API.

**Architecture:** Spring Boot microservice on port 8126 with context path `/api/notification`. Consumes domain events from Kafka, renders notification templates with `{{var}}` replacement, dispatches via channel interface (log-only initially), and persists all notification history. Follows same patterns as aurix-fraud and aurix-customer modules.

**Tech Stack:** Java 25, Spring Boot 4.1.0, Spring Data JPA, Spring Kafka, H2 (test), PostgreSQL (prod), Mockito, JUnit 5.

## Global Constraints

- Java 25 (`maven.compiler.source/target = 25`)
- Spring Boot 4.1.0, Spring Cloud 2025.1.2
- Extend `com.aurix.platform.shared.entity.BaseEntity` for all entities
- Schema: `aurix` in JPA `@Table` annotations
- Checkstyle: max line length 120, suppressed rules (Javadoc, MagicNumber, HiddenField)
- SpotBugs: excludes `EI_EXPOSE_REP` / `EI_EXPOSE_REP2`
- No Redis dependency (unlike fraud module)
- Test profile excludes KafkaAutoConfiguration, uses H2 in-memory
- Constructor injection (no `@Autowired` field injection)

## File Structure

```
apps/backend/aurix-notification/
├── pom.xml                                          (replace stub)
├── src/main/java/com/aurix/platform/notification/
│   ├── AurixNotificationApplication.java
│   ├── config/
│   │   ├── KafkaConfig.java                         (topics: notificacao.enviada, notificacao.falhou)
│   │   └── SecurityConfig.java
│   ├── entity/
│   │   ├── TemplateNotificacao.java
│   │   ├── FilaNotificacao.java
│   │   ├── ConfirmacaoRecebimento.java
│   │   └── PreferenciaCliente.java
│   ├── repository/
│   │   ├── TemplateNotificacaoRepository.java
│   │   ├── FilaNotificacaoRepository.java
│   │   ├── ConfirmacaoRecebimentoRepository.java
│   │   └── PreferenciaClienteRepository.java
│   ├── service/
│   │   ├── NotificacaoService.java                  (template rendering + channel dispatch)
│   │   ├── NotificacaoConsumer.java                 (Kafka listener)
│   │   ├── NotificacaoProducer.java                 (Kafka sender)
│   │   └── channel/
│   │       ├── NotificacaoChannel.java              (interface)
│   │       └── LogChannel.java                      (log-only implementation)
│   └── controller/
│       ├── NotificacaoController.java
│       └── HealthController.java
├── src/main/resources/
│   └── application.yml
└── src/test/java/com/aurix/platform/notification/
    ├── AurixNotificationApplicationTest.java
    ├── service/
    │   ├── NotificacaoServiceTest.java
    │   ├── NotificacaoConsumerTest.java
    │   └── channel/LogChannelTest.java
    └── controller/
        └── NotificacaoControllerTest.java
```

---

### Task 1: Scaffold — pom.xml, application classes, config

**Files:**
- Modify: `apps/backend/aurix-notification/pom.xml` (replace stub)
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/AurixNotificationApplication.java`
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/config/KafkaConfig.java`
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/config/SecurityConfig.java`
- Create: `apps/backend/aurix-notification/src/main/resources/application.yml`
- Create: `apps/backend/aurix-notification/src/test/resources/application-test.yml`
- Create: `apps/backend/aurix-notification/src/test/java/com/aurix/platform/notification/AurixNotificationApplicationTest.java`

**Interfaces:**
- Produces: application entry point, Kafka topic beans, security config, test context load

- [ ] **Step 1: Create directory structure**

```bash
mkdir -p apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/{config,controller,entity,repository,service/channel}
mkdir -p apps/backend/aurix-notification/src/main/resources
mkdir -p apps/backend/aurix-notification/src/test/java/com/aurix/platform/notification/{service/channel,controller}
mkdir -p apps/backend/aurix-notification/src/test/resources
```

- [ ] **Step 2: Replace pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.aurix.platform</groupId>
        <artifactId>aurix-platform</artifactId>
        <version>1.0.0</version>
    </parent>
    <artifactId>aurix-notification</artifactId>
    <packaging>jar</packaging>
    <name>AURIX Notification</name>
    <description>Multi-channel notification service with template rendering and Kafka integration</description>
    <dependencies>
        <dependency>
            <groupId>com.aurix.platform</groupId>
            <artifactId>aurix-shared</artifactId>
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
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.3.0</version>
        </dependency>
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
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
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
</project>
```

- [ ] **Step 3: Create AurixNotificationApplication.java**

```java
package com.aurix.platform.notification;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.persistence.autoconfigure.EntityScan;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication(scanBasePackages = { "com.aurix.platform.notification", "com.aurix.platform.shared" })
@EntityScan(basePackages = { "com.aurix.platform.notification.entity" })
@EnableJpaRepositories(basePackages = { "com.aurix.platform.notification.repository" })
@EnableScheduling
public class AurixNotificationApplication {
    public static void main(String[] args) {
        SpringApplication.run(AurixNotificationApplication.class, args);
    }
}
```

- [ ] **Step 4: Create KafkaConfig.java**

```java
package com.aurix.platform.notification.config;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration("notificationKafkaConfig")
public class KafkaConfig {
    @Bean
    public NewTopic notificacaoEnviadaTopic() {
        return TopicBuilder.name("notificacao.enviada").partitions(3).replicas(1).build();
    }

    @Bean
    public NewTopic notificacaoFalhouTopic() {
        return TopicBuilder.name("notificacao.falhou").partitions(3).replicas(1).build();
    }
}
```

- [ ] **Step 5: Create SecurityConfig.java**

```java
package com.aurix.platform.notification.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
                .authorizeHttpRequests(authz -> authz.anyRequest().permitAll())
                .csrf(csrf -> csrf.disable())
                .build();
    }
}
```

- [ ] **Step 6: Create application.yml**

```yaml
server:
  port: 8126
  servlet:
    context-path: /api/notification

spring:
  application:
    name: aurix-notification
  profiles:
    active: dev
  datasource:
    url: jdbc:postgresql://localhost:5432/aurix_db
    username: aurix_user
    password: aurix_dev_password
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
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
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: aurix-notification-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer

logging:
  level:
    com.aurix.platform: DEBUG

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always

aurix:
  notification:
    version: "1.0.0"
  security:
    jwt:
      secret: "aurix-jwt-secret-key-2024"
      expiration: 86400000

---
spring:
  config:
    activate:
      on-profile: dev
  jpa:
    hibernate:
      ddl-auto: create-drop

---
spring:
  config:
    activate:
      on-profile: prod
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
```

- [ ] **Step 7: Create application-test.yml**

```yaml
spring:
  main:
    allow-bean-definition-overriding: true
  autoconfigure:
    exclude:
      - org.springframework.boot.kafka.autoconfigure.KafkaAutoConfiguration
  datasource:
    url: jdbc:h2:mem:aurix;MODE=PostgreSQL;DATABASE_TO_LOWER=TRUE;DEFAULT_NULL_ORDERING=HIGH;INIT=CREATE SCHEMA IF NOT EXISTS aurix
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop
    properties:
      hibernate:
        dialect: org.hibernate.dialect.H2Dialect
    show-sql: false
  sql:
    init:
      mode: never
  kafka:
    bootstrap-servers: localhost:9092

management:
  health:
    kafka:
      enabled: false
    db:
      enabled: false
```

- [ ] **Step 8: Create AurixNotificationApplicationTest.java**

```java
package com.aurix.platform.notification;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.bean.override.mockito.MockitoBean;

@SpringBootTest
@ActiveProfiles("test")
class AurixNotificationApplicationTest {

    @MockitoBean
    private KafkaTemplate<String, String> kafkaTemplate;

    @Test
    void contextLoads() {
    }
}
```

- [ ] **Step 9: Run test to verify context loads**

Run: `mvn clean test -pl aurix-notification -am -Dtest=AurixNotificationApplicationTest -DfailIfNoTests=false`
Expected: PASS (context loads with mocked KafkaTemplate)

---

### Task 2: Entities and Repositories

**Files:**
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/entity/TemplateNotificacao.java`
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/entity/FilaNotificacao.java`
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/entity/ConfirmacaoRecebimento.java`
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/entity/PreferenciaCliente.java`
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/repository/TemplateNotificacaoRepository.java`
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/repository/FilaNotificacaoRepository.java`
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/repository/ConfirmacaoRecebimentoRepository.java`
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/repository/PreferenciaClienteRepository.java`

**Interfaces:**
- Produces: entity classes and repository interfaces consumed by service layer

- [ ] **Step 1: Create TemplateNotificacao.java**

```java
package com.aurix.platform.notification.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

@Entity
@Table(name = "templates_notificacao", schema = "aurix")
public class TemplateNotificacao extends BaseEntity {
    @Column(name = "codigo", nullable = false, unique = true, length = 100)
    private String codigo;

    @Column(name = "nome", nullable = false, length = 200)
    private String nome;

    @Column(name = "canal", nullable = false, length = 30)
    private String canal;

    @Column(name = "assunto", length = 200)
    private String assunto;

    @Column(name = "corpo", nullable = false, length = 4000)
    private String corpo;

    @Column(name = "variaveis", length = 2000)
    private String variaveis;

    @Column(name = "ativo", nullable = false)
    private Boolean ativo = true;

    public String getCodigo() { return codigo; }
    public void setCodigo(String codigo) { this.codigo = codigo; }
    public String getNome() { return nome; }
    public void setNome(String nome) { this.nome = nome; }
    public String getCanal() { return canal; }
    public void setCanal(String canal) { this.canal = canal; }
    public String getAssunto() { return assunto; }
    public void setAssunto(String assunto) { this.assunto = assunto; }
    public String getCorpo() { return corpo; }
    public void setCorpo(String corpo) { this.corpo = corpo; }
    public String getVariaveis() { return variaveis; }
    public void setVariaveis(String variaveis) { this.variaveis = variaveis; }
    public Boolean getAtivo() { return ativo; }
    public void setAtivo(Boolean ativo) { this.ativo = ativo; }
}
```

- [ ] **Step 2: Create FilaNotificacao.java**

```java
package com.aurix.platform.notification.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import java.time.LocalDateTime;

@Entity
@Table(name = "fila_notificacoes", schema = "aurix")
public class FilaNotificacao extends BaseEntity {
    @Column(name = "cliente_id", nullable = false)
    private Long clienteId;

    @Column(name = "template_codigo", nullable = false, length = 100)
    private String templateCodigo;

    @Column(name = "canal", nullable = false, length = 30)
    private String canal;

    @Column(name = "destinatario", nullable = false, length = 200)
    private String destinatario;

    @Column(name = "assunto", length = 200)
    private String assunto;

    @Column(name = "corpo_renderizado", nullable = false, length = 4000)
    private String corpoRenderizado;

    @Column(name = "status", nullable = false, length = 20)
    private String status;

    @Column(name = "tentativas")
    private Integer tentativas = 0;

    @Column(name = "max_tentativas")
    private Integer maxTentativas = 3;

    @Column(name = "agendada_para")
    private LocalDateTime agendadaPara;

    @Column(name = "enviada_em")
    private LocalDateTime enviadaEm;

    @Column(name = "falha_motivo", length = 500)
    private String falhaMotivo;

    public Long getClienteId() { return clienteId; }
    public void setClienteId(Long clienteId) { this.clienteId = clienteId; }
    public String getTemplateCodigo() { return templateCodigo; }
    public void setTemplateCodigo(String templateCodigo) { this.templateCodigo = templateCodigo; }
    public String getCanal() { return canal; }
    public void setCanal(String canal) { this.canal = canal; }
    public String getDestinatario() { return destinatario; }
    public void setDestinatario(String destinatario) { this.destinatario = destinatario; }
    public String getAssunto() { return assunto; }
    public void setAssunto(String assunto) { this.assunto = assunto; }
    public String getCorpoRenderizado() { return corpoRenderizado; }
    public void setCorpoRenderizado(String corpoRenderizado) { this.corpoRenderizado = corpoRenderizado; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public Integer getTentativas() { return tentativas; }
    public void setTentativas(Integer tentativas) { this.tentativas = tentativas; }
    public Integer getMaxTentativas() { return maxTentativas; }
    public void setMaxTentativas(Integer maxTentativas) { this.maxTentativas = maxTentativas; }
    public LocalDateTime getAgendadaPara() { return agendadaPara; }
    public void setAgendadaPara(LocalDateTime agendadaPara) { this.agendadaPara = agendadaPara; }
    public LocalDateTime getEnviadaEm() { return enviadaEm; }
    public void setEnviadaEm(LocalDateTime enviadaEm) { this.enviadaEm = enviadaEm; }
    public String getFalhaMotivo() { return falhaMotivo; }
    public void setFalhaMotivo(String falhaMotivo) { this.falhaMotivo = falhaMotivo; }
}
```

- [ ] **Step 3: Create ConfirmacaoRecebimento.java**

```java
package com.aurix.platform.notification.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import java.time.LocalDateTime;

@Entity
@Table(name = "confirmacoes_recebimento", schema = "aurix")
public class ConfirmacaoRecebimento extends BaseEntity {
    @Column(name = "fila_notificacao_id", nullable = false)
    private Long filaNotificacaoId;

    @Column(name = "cliente_id", nullable = false)
    private Long clienteId;

    @Column(name = "canal", nullable = false, length = 30)
    private String canal;

    @Column(name = "recebida_em", nullable = false)
    private LocalDateTime recebidaEm;

    @Column(name = "lida_em")
    private LocalDateTime lidaEm;

    @Column(name = "clicou_link")
    private Boolean clicouLink = false;

    public Long getFilaNotificacaoId() { return filaNotificacaoId; }
    public void setFilaNotificacaoId(Long filaNotificacaoId) { this.filaNotificacaoId = filaNotificacaoId; }
    public Long getClienteId() { return clienteId; }
    public void setClienteId(Long clienteId) { this.clienteId = clienteId; }
    public String getCanal() { return canal; }
    public void setCanal(String canal) { this.canal = canal; }
    public LocalDateTime getRecebidaEm() { return recebidaEm; }
    public void setRecebidaEm(LocalDateTime recebidaEm) { this.recebidaEm = recebidaEm; }
    public LocalDateTime getLidaEm() { return lidaEm; }
    public void setLidaEm(LocalDateTime lidaEm) { this.lidaEm = lidaEm; }
    public Boolean getClicouLink() { return clicouLink; }
    public void setClicouLink(Boolean clicouLink) { this.clicouLink = clicouLink; }
}
```

- [ ] **Step 4: Create PreferenciaCliente.java**

```java
package com.aurix.platform.notification.entity;

import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

@Entity
@Table(name = "preferencias_cliente", schema = "aurix")
public class PreferenciaCliente extends BaseEntity {
    @Column(name = "cliente_id", nullable = false, unique = true)
    private Long clienteId;

    @Column(name = "ativo", nullable = false)
    private Boolean ativo = true;

    @Column(name = "email_ativo", nullable = false)
    private Boolean emailAtivo = true;

    @Column(name = "sms_ativo", nullable = false)
    private Boolean smsAtivo = true;

    @Column(name = "push_ativo", nullable = false)
    private Boolean pushAtivo = true;

    @Column(name = "whatsapp_ativo", nullable = false)
    private Boolean whatsappAtivo = false;

    @Column(name = "horario_inicio", length = 5)
    private String horarioInicio;

    @Column(name = "horario_fim", length = 5)
    private String horarioFim;

    public Long getClienteId() { return clienteId; }
    public void setClienteId(Long clienteId) { this.clienteId = clienteId; }
    public Boolean getAtivo() { return ativo; }
    public void setAtivo(Boolean ativo) { this.ativo = ativo; }
    public Boolean getEmailAtivo() { return emailAtivo; }
    public void setEmailAtivo(Boolean emailAtivo) { this.emailAtivo = emailAtivo; }
    public Boolean getSmsAtivo() { return smsAtivo; }
    public void setSmsAtivo(Boolean smsAtivo) { this.smsAtivo = smsAtivo; }
    public Boolean getPushAtivo() { return pushAtivo; }
    public void setPushAtivo(Boolean pushAtivo) { this.pushAtivo = pushAtivo; }
    public Boolean getWhatsappAtivo() { return whatsappAtivo; }
    public void setWhatsappAtivo(Boolean whatsappAtivo) { this.whatsappAtivo = whatsappAtivo; }
    public String getHorarioInicio() { return horarioInicio; }
    public void setHorarioInicio(String horarioInicio) { this.horarioInicio = horarioInicio; }
    public String getHorarioFim() { return horarioFim; }
    public void setHorarioFim(String horarioFim) { this.horarioFim = horarioFim; }
}
```

- [ ] **Step 5: Create TemplateNotificacaoRepository.java**

```java
package com.aurix.platform.notification.repository;

import com.aurix.platform.notification.entity.TemplateNotificacao;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface TemplateNotificacaoRepository extends JpaRepository<TemplateNotificacao, Long> {
    Optional<TemplateNotificacao> findByCodigoAndAtivoTrue(String codigo);
}
```

- [ ] **Step 6: Create FilaNotificacaoRepository.java**

```java
package com.aurix.platform.notification.repository;

import com.aurix.platform.notification.entity.FilaNotificacao;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface FilaNotificacaoRepository extends JpaRepository<FilaNotificacao, Long> {
    List<FilaNotificacao> findByClienteId(Long clienteId);
    List<FilaNotificacao> findByStatus(String status);
}
```

- [ ] **Step 7: Create ConfirmacaoRecebimentoRepository.java**

```java
package com.aurix.platform.notification.repository;

import com.aurix.platform.notification.entity.ConfirmacaoRecebimento;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface ConfirmacaoRecebimentoRepository extends JpaRepository<ConfirmacaoRecebimento, Long> {
    List<ConfirmacaoRecebimento> findByClienteId(Long clienteId);
    List<ConfirmacaoRecebimento> findByFilaNotificacaoId(Long filaNotificacaoId);
}
```

- [ ] **Step 8: Create PreferenciaClienteRepository.java**

```java
package com.aurix.platform.notification.repository;

import com.aurix.platform.notification.entity.PreferenciaCliente;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface PreferenciaClienteRepository extends JpaRepository<PreferenciaCliente, Long> {
    Optional<PreferenciaCliente> findByClienteId(Long clienteId);
}
```

- [ ] **Step 9: Run tests to verify compilation**

Run: `mvn clean compile -pl aurix-notification -am`
Expected: PASS

---

### Task 3: Channel interface, LogChannel, NotificacaoService

**Files:**
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/service/channel/NotificacaoChannel.java`
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/service/channel/LogChannel.java`
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/service/NotificacaoService.java`
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/service/NotificacaoProducer.java`
- Create: `apps/backend/aurix-notification/src/test/java/com/aurix/platform/notification/service/NotificacaoServiceTest.java`
- Create: `apps/backend/aurix-notification/src/test/java/com/aurix/platform/notification/service/channel/LogChannelTest.java`

**Interfaces:**
- Consumes: `TemplateNotificacao`, `FilaNotificacao`, `PreferenciaCliente`, `ConfirmacaoRecebimento` entities, all repositories
- Produces: `NotificacaoService.enviar()`, `NotificacaoService.processarEvento()`, `NotificacaoProducer` Kafka sends

- [ ] **Step 1: Create NotificacaoChannel.java**

```java
package com.aurix.platform.notification.service.channel;

import com.aurix.platform.notification.entity.FilaNotificacao;

public interface NotificacaoChannel {
    void send(FilaNotificacao notificacao);
    String getChannelName();
}
```

- [ ] **Step 2: Create LogChannel.java**

```java
package com.aurix.platform.notification.service.channel;

import com.aurix.platform.notification.entity.FilaNotificacao;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Component
public class LogChannel implements NotificacaoChannel {
    private static final Logger log = LoggerFactory.getLogger(LogChannel.class);

    @Override
    public void send(FilaNotificacao notificacao) {
        log.info("[NOTIFICATION][{}] Para: {} | Assunto: {} | Corpo: {}",
                notificacao.getCanal(), notificacao.getDestinatario(),
                notificacao.getAssunto(), notificacao.getCorpoRenderizado());
    }

    @Override
    public String getChannelName() {
        return "LOG";
    }
}
```

- [ ] **Step 3: Create NotificacaoProducer.java**

```java
package com.aurix.platform.notification.service;

import com.aurix.platform.notification.entity.FilaNotificacao;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class NotificacaoProducer {
    private static final Logger log = LoggerFactory.getLogger(NotificacaoProducer.class);
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    public NotificacaoProducer(KafkaTemplate<String, String> kafkaTemplate, ObjectMapper objectMapper) {
        this.kafkaTemplate = kafkaTemplate;
        this.objectMapper = objectMapper;
    }

    public void notificacaoEnviada(FilaNotificacao notificacao) {
        try {
            ObjectNode json = objectMapper.createObjectNode();
            json.put("notificacaoId", notificacao.getId());
            json.put("clienteId", notificacao.getClienteId());
            json.put("canal", notificacao.getCanal());
            json.put("templateCodigo", notificacao.getTemplateCodigo());
            json.put("status", notificacao.getStatus());
            kafkaTemplate.send("notificacao.enviada",
                    String.valueOf(notificacao.getClienteId()), objectMapper.writeValueAsString(json));
        } catch (Exception e) {
            log.error("Erro ao enviar evento notificacao.enviada", e);
        }
    }

    public void notificacaoFalhou(FilaNotificacao notificacao, String motivo) {
        try {
            ObjectNode json = objectMapper.createObjectNode();
            json.put("notificacaoId", notificacao.getId());
            json.put("clienteId", notificacao.getClienteId());
            json.put("canal", notificacao.getCanal());
            json.put("motivo", motivo);
            kafkaTemplate.send("notificacao.falhou",
                    String.valueOf(notificacao.getClienteId()), objectMapper.writeValueAsString(json));
        } catch (Exception e) {
            log.error("Erro ao enviar evento notificacao.falhou", e);
        }
    }
}
```

- [ ] **Step 4: Create NotificacaoService.java**

```java
package com.aurix.platform.notification.service;

import com.aurix.platform.notification.entity.ConfirmacaoRecebimento;
import com.aurix.platform.notification.entity.FilaNotificacao;
import com.aurix.platform.notification.entity.PreferenciaCliente;
import com.aurix.platform.notification.entity.TemplateNotificacao;
import com.aurix.platform.notification.repository.ConfirmacaoRecebimentoRepository;
import com.aurix.platform.notification.repository.FilaNotificacaoRepository;
import com.aurix.platform.notification.repository.PreferenciaClienteRepository;
import com.aurix.platform.notification.repository.TemplateNotificacaoRepository;
import com.aurix.platform.notification.service.channel.LogChannel;
import com.aurix.platform.notification.service.channel.NotificacaoChannel;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import java.time.LocalDateTime;
import java.util.Map;
import java.util.Optional;

@Service
public class NotificacaoService {
    private static final Logger log = LoggerFactory.getLogger(NotificacaoService.class);
    private final TemplateNotificacaoRepository templateRepository;
    private final FilaNotificacaoRepository filaRepository;
    private final ConfirmacaoRecebimentoRepository confirmacaoRepository;
    private final PreferenciaClienteRepository preferenciaRepository;
    private final NotificacaoProducer producer;
    private final LogChannel logChannel;

    public NotificacaoService(TemplateNotificacaoRepository templateRepository,
            FilaNotificacaoRepository filaRepository,
            ConfirmacaoRecebimentoRepository confirmacaoRepository,
            PreferenciaClienteRepository preferenciaRepository,
            NotificacaoProducer producer, LogChannel logChannel) {
        this.templateRepository = templateRepository;
        this.filaRepository = filaRepository;
        this.confirmacaoRepository = confirmacaoRepository;
        this.preferenciaRepository = preferenciaRepository;
        this.producer = producer;
        this.logChannel = logChannel;
    }

    public FilaNotificacao enviar(Long clienteId, String templateCodigo,
            String destinatario, Map<String, String> variaveis) {
        Optional<TemplateNotificacao> templateOpt =
                templateRepository.findByCodigoAndAtivoTrue(templateCodigo);
        if (templateOpt.isEmpty()) {
            throw new IllegalArgumentException("Template nao encontrado ou inativo: " + templateCodigo);
        }
        TemplateNotificacao template = templateOpt.get();

        Optional<PreferenciaCliente> prefOpt = preferenciaRepository.findByClienteId(clienteId);
        if (prefOpt.isPresent()) {
            PreferenciaCliente pref = prefOpt.get();
            if (!pref.getAtivo()) {
                log.info("Notificacoes desativadas para cliente {}", clienteId);
                return null;
            }
            if (!isCanalAtivo(pref, template.getCanal())) {
                log.info("Canal {} desativado para cliente {}", template.getCanal(), clienteId);
                return null;
            }
        }

        String corpoRenderizado = renderizar(template.getCorpo(), variaveis);
        String assuntoRenderizado = template.getAssunto() != null
                ? renderizar(template.getAssunto(), variaveis) : null;

        FilaNotificacao notificacao = new FilaNotificacao();
        notificacao.setClienteId(clienteId);
        notificacao.setTemplateCodigo(templateCodigo);
        notificacao.setCanal(template.getCanal());
        notificacao.setDestinatario(destinatario);
        notificacao.setAssunto(assuntoRenderizado);
        notificacao.setCorpoRenderizado(corpoRenderizado);
        notificacao.setStatus("ENVIANDO");
        notificacao.setTentativas(0);
        notificacao.setMaxTentativas(3);
        notificacao = filaRepository.save(notificacao);

        try {
            logChannel.send(notificacao);
            notificacao.setStatus("ENVIADA");
            notificacao.setEnviadaEm(LocalDateTime.now());
            notificacao = filaRepository.save(notificacao);

            ConfirmacaoRecebimento confirmacao = new ConfirmacaoRecebimento();
            confirmacao.setFilaNotificacaoId(notificacao.getId());
            confirmacao.setClienteId(clienteId);
            confirmacao.setCanal(template.getCanal());
            confirmacao.setRecebidaEm(LocalDateTime.now());
            confirmacaoRepository.save(confirmacao);

            producer.notificacaoEnviada(notificacao);
        } catch (Exception e) {
            notificacao.setStatus("FALHA");
            notificacao.setFalhaMotivo(e.getMessage());
            notificacao.setTentativas(notificacao.getTentativas() + 1);
            filaRepository.save(notificacao);
            producer.notificacaoFalhou(notificacao, e.getMessage());
            log.error("Erro ao enviar notificacao para cliente {}: {}", clienteId, e.getMessage());
        }

        return notificacao;
    }

    public String renderizar(String template, Map<String, String> variaveis) {
        String resultado = template;
        if (variaveis != null) {
            for (Map.Entry<String, String> entry : variaveis.entrySet()) {
                resultado = resultado.replace("{{" + entry.getKey() + "}}",
                        entry.getValue() != null ? entry.getValue() : "");
            }
        }
        return resultado;
    }

    public TemplateNotificacao criarTemplate(TemplateNotificacao template) {
        return templateRepository.save(template);
    }

    public java.util.List<TemplateNotificacao> listarTemplates() {
        return templateRepository.findAll();
    }

    public java.util.List<FilaNotificacao> listarNotificacoesPorCliente(Long clienteId) {
        return filaRepository.findByClienteId(clienteId);
    }

    public PreferenciaCliente salvarPreferencia(PreferenciaCliente preferencia) {
        Optional<PreferenciaCliente> existente =
                preferenciaRepository.findByClienteId(preferencia.getClienteId());
        if (existente.isPresent()) {
            PreferenciaCliente pref = existente.get();
            pref.setAtivo(preferencia.getAtivo());
            pref.setEmailAtivo(preferencia.getEmailAtivo());
            pref.setSmsAtivo(preferencia.getSmsAtivo());
            pref.setPushAtivo(preferencia.getPushAtivo());
            pref.setWhatsappAtivo(preferencia.getWhatsappAtivo());
            pref.setHorarioInicio(preferencia.getHorarioInicio());
            pref.setHorarioFim(preferencia.getHorarioFim());
            return preferenciaRepository.save(pref);
        }
        return preferenciaRepository.save(preferencia);
    }

    public Optional<PreferenciaCliente> buscarPreferencia(Long clienteId) {
        return preferenciaRepository.findByClienteId(clienteId);
    }

    private boolean isCanalAtivo(PreferenciaCliente pref, String canal) {
        return switch (canal.toUpperCase()) {
            case "EMAIL" -> pref.getEmailAtivo();
            case "SMS" -> pref.getSmsAtivo();
            case "PUSH" -> pref.getPushAtivo();
            case "WHATSAPP" -> pref.getWhatsappAtivo();
            default -> true;
        };
    }
}
```

- [ ] **Step 5: Create NotificacaoServiceTest.java**

```java
package com.aurix.platform.notification.service;

import com.aurix.platform.notification.entity.FilaNotificacao;
import com.aurix.platform.notification.entity.PreferenciaCliente;
import com.aurix.platform.notification.entity.TemplateNotificacao;
import com.aurix.platform.notification.repository.ConfirmacaoRecebimentoRepository;
import com.aurix.platform.notification.repository.FilaNotificacaoRepository;
import com.aurix.platform.notification.repository.PreferenciaClienteRepository;
import com.aurix.platform.notification.repository.TemplateNotificacaoRepository;
import com.aurix.platform.notification.service.channel.LogChannel;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import java.util.Map;
import java.util.Optional;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class NotificacaoServiceTest {
    @Mock private TemplateNotificacaoRepository templateRepository;
    @Mock private FilaNotificacaoRepository filaRepository;
    @Mock private ConfirmacaoRecebimentoRepository confirmacaoRepository;
    @Mock private PreferenciaClienteRepository preferenciaRepository;
    @Mock private NotificacaoProducer producer;
    @Mock private LogChannel logChannel;
    @InjectMocks private NotificacaoService service;

    @Test
    void deveRenderizarTemplateComVariaveis() {
        String template = "Ola {{nome}}, bem-vindo ao {{plataforma}}!";
        Map<String, String> vars = Map.of("nome", "Joao", "plataforma", "AURIX");
        String resultado = service.renderizar(template, vars);
        assertEquals("Ola Joao, bem-vindo ao AURIX!", resultado);
    }

    @Test
    void deveRenderizarTemplateSemVariaveis() {
        String template = "Mensagem estatica";
        String resultado = service.renderizar(template, null);
        assertEquals("Mensagem estatica", resultado);
    }

    @Test
    void deveTratarVariavelNulaNaRenderizacao() {
        String template = "Ola {{nome}}!";
        Map<String, String> vars = Map.of("nome", null);
        String resultado = service.renderizar(template, vars);
        assertEquals("Ola !", resultado);
    }

    @Test
    void deveEnviarNotificacaoComSucesso() {
        TemplateNotificacao template = new TemplateNotificacao();
        template.setCodigo("cliente.criado");
        template.setCanal("EMAIL");
        template.setCorpo("Bem-vindo, {{nome}}!");
        template.setAssunto("Bem-vindo!");
        template.setAtivo(true);

        when(templateRepository.findByCodigoAndAtivoTrue("cliente.criado")).thenReturn(Optional.of(template));
        when(preferenciaRepository.findByClienteId(1L)).thenReturn(Optional.empty());
        when(filaRepository.save(any())).thenAnswer(inv -> {
            FilaNotificacao saved = inv.getArgument(0);
            saved.setId(1L);
            return saved;
        });

        FilaNotificacao resultado = service.enviar(1L, "cliente.criado", "joao@test.com", Map.of("nome", "Joao"));

        assertNotNull(resultado);
        assertEquals("ENVIADA", resultado.getStatus());
        assertEquals("Bem-vindo, Joao!", resultado.getCorpoRenderizado());
        verify(logChannel).send(any());
        verify(producer).notificacaoEnviada(any());
    }

    @Test
    void deveLancarExcecaoQuandoTemplateNaoEncontrado() {
        when(templateRepository.findByCodigoAndAtivoTrue("inexistente")).thenReturn(Optional.empty());
        assertThrows(IllegalArgumentException.class,
                () -> service.enviar(1L, "inexistente", "joao@test.com", Map.of()));
    }

    @Test
    void devePularNotificacaoQuandoClienteDesativado() {
        TemplateNotificacao template = new TemplateNotificacao();
        template.setCodigo("cliente.criado");
        template.setCanal("EMAIL");
        template.setAtivo(true);

        PreferenciaCliente pref = new PreferenciaCliente();
        pref.setClienteId(1L);
        pref.setAtivo(false);
        pref.setEmailAtivo(true);

        when(templateRepository.findByCodigoAndAtivoTrue("cliente.criado")).thenReturn(Optional.of(template));
        when(preferenciaRepository.findByClienteId(1L)).thenReturn(Optional.of(pref));

        FilaNotificacao resultado = service.enviar(1L, "cliente.criado", "joao@test.com", Map.of());

        assertNull(resultado);
        verify(filaRepository, never()).save(any());
    }

    @Test
    void devePularNotificacaoQuandoCanalDesativado() {
        TemplateNotificacao template = new TemplateNotificacao();
        template.setCodigo("cliente.criado");
        template.setCanal("SMS");
        template.setAtivo(true);

        PreferenciaCliente pref = new PreferenciaCliente();
        pref.setClienteId(1L);
        pref.setAtivo(true);
        pref.setEmailAtivo(true);
        pref.setSmsAtivo(false);

        when(templateRepository.findByCodigoAndAtivoTrue("cliente.criado")).thenReturn(Optional.of(template));
        when(preferenciaRepository.findByClienteId(1L)).thenReturn(Optional.of(pref));

        FilaNotificacao resultado = service.enviar(1L, "cliente.criado", "11999999999", Map.of());

        assertNull(resultado);
        verify(filaRepository, never()).save(any());
    }

    @Test
    void deveAtualizarPreferenciaExistente() {
        PreferenciaCliente existente = new PreferenciaCliente();
        existente.setClienteId(1L);
        existente.setAtivo(true);
        existente.setEmailAtivo(true);
        existente.setSmsAtivo(true);
        existente.setPushAtivo(true);

        PreferenciaCliente nova = new PreferenciaCliente();
        nova.setClienteId(1L);
        nova.setAtivo(true);
        nova.setEmailAtivo(false);
        nova.setSmsAtivo(true);
        nova.setPushAtivo(false);

        when(preferenciaRepository.findByClienteId(1L)).thenReturn(Optional.of(existente));
        when(preferenciaRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        PreferenciaCliente resultado = service.salvarPreferencia(nova);

        assertFalse(resultado.getEmailAtivo());
        assertFalse(resultado.getPushAtivo());
        assertTrue(resultado.getSmsAtivo());
    }
}
```

- [ ] **Step 6: Create LogChannelTest.java**

```java
package com.aurix.platform.notification.service.channel;

import com.aurix.platform.notification.entity.FilaNotificacao;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class LogChannelTest {
    @Test
    void deveRetornarNomeCanalLog() {
        LogChannel channel = new LogChannel();
        assertEquals("LOG", channel.getChannelName());
    }

    @Test
    void deveEnviarNotificacaoSemExcecao() {
        LogChannel channel = new LogChannel();
        FilaNotificacao notificacao = new FilaNotificacao();
        notificacao.setCanal("EMAIL");
        notificacao.setDestinatario("test@test.com");
        notificacao.setAssunto("Teste");
        notificacao.setCorpoRenderizado("Corpo da mensagem");
        assertDoesNotThrow(() -> channel.send(notificacao));
    }
}
```

- [ ] **Step 7: Run all tests**

Run: `mvn clean test -pl aurix-notification -am`
Expected: All tests PASS

---

### Task 4: NotificacaoConsumer (Kafka listeners)

**Files:**
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/service/NotificacaoConsumer.java`
- Create: `apps/backend/aurix-notification/src/test/java/com/aurix/platform/notification/service/NotificacaoConsumerTest.java`

**Interfaces:**
- Consumes: `NotificacaoService.enviar()` from Task 3
- Produces: Kafka event handlers for `cliente.criado`, `kyc.aprovado`, `kyc.rejeitado`, `fraude.transacao.bloqueada`

- [ ] **Step 1: Create NotificacaoConsumer.java**

```java
package com.aurix.platform.notification.service;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;
import java.util.HashMap;
import java.util.Map;

@Service
public class NotificacaoConsumer {
    private static final Logger log = LoggerFactory.getLogger(NotificacaoConsumer.class);
    private final NotificacaoService notificacaoService;
    private final ObjectMapper objectMapper;

    public NotificacaoConsumer(NotificacaoService notificacaoService, ObjectMapper objectMapper) {
        this.notificacaoService = notificacaoService;
        this.objectMapper = objectMapper;
    }

    @KafkaListener(topics = "cliente.criado", groupId = "aurix-notification-group")
    public void onClienteCriado(String message) {
        processEvent(message, "cliente.criado", "cliente_criado");
    }

    @KafkaListener(topics = "kyc.aprovado", groupId = "aurix-notification-group")
    public void onKycAprovado(String message) {
        processEvent(message, "kyc.aprovado", "kyc_aprovado");
    }

    @KafkaListener(topics = "kyc.rejeitado", groupId = "aurix-notification-group")
    public void onKycRejeitado(String message) {
        processEvent(message, "kyc.rejeitado", "kyc_rejeitado");
    }

    @KafkaListener(topics = "fraude.transacao.bloqueada", groupId = "aurix-notification-group")
    public void onFraudeTransacaoBloqueada(String message) {
        processEvent(message, "fraude.transacao.bloqueada", "fraude_transacao_bloqueada");
    }

    private void processEvent(String message, String topic, String templateCodigo) {
        try {
            JsonNode json = objectMapper.readTree(message);
            Long clienteId = json.has("clienteId") ? json.get("clienteId").asLong() : null;
            if (clienteId == null) {
                log.warn("Evento {} sem clienteId, ignorando", topic);
                return;
            }

            Map<String, String> variaveis = new HashMap<>();
            variaveis.put("clienteId", String.valueOf(clienteId));

            if (json.has("nome")) {
                variaveis.put("nome", json.get("nome").asText());
            }
            if (json.has("documento")) {
                variaveis.put("documento", json.get("documento").asText());
            }
            if (json.has("motivo")) {
                variaveis.put("motivo", json.get("motivo").asText());
            }
            if (json.has("transacaoRef")) {
                variaveis.put("transacaoRef", json.get("transacaoRef").asText());
            }

            String destinatario = json.has("email") ? json.get("email").asText()
                    : json.has("telefone") ? json.get("telefone").asText()
                    : "cliente-" + clienteId;

            notificacaoService.enviar(clienteId, templateCodigo, destinatario, variaveis);
        } catch (JsonProcessingException e) {
            log.error("Erro ao processar evento Kafka {}: {}", topic, e.getMessage());
        }
    }
}
```

- [ ] **Step 2: Create NotificacaoConsumerTest.java**

```java
package com.aurix.platform.notification.service;

import com.aurix.platform.notification.entity.FilaNotificacao;
import com.aurix.platform.notification.entity.TemplateNotificacao;
import com.aurix.platform.notification.repository.ConfirmacaoRecebimentoRepository;
import com.aurix.platform.notification.repository.FilaNotificacaoRepository;
import com.aurix.platform.notification.repository.PreferenciaClienteRepository;
import com.aurix.platform.notification.repository.TemplateNotificacaoRepository;
import com.aurix.platform.notification.service.channel.LogChannel;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import java.util.Optional;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class NotificacaoConsumerTest {
    @Mock private TemplateNotificacaoRepository templateRepository;
    @Mock private FilaNotificacaoRepository filaRepository;
    @Mock private ConfirmacaoRecebimentoRepository confirmacaoRepository;
    @Mock private PreferenciaClienteRepository preferenciaRepository;
    @Mock private NotificacaoProducer producer;
    @Mock private LogChannel logChannel;
    @InjectMocks private NotificacaoService notificacaoService;
    private NotificacaoConsumer consumer;

    @BeforeEach
    void setUp() {
        consumer = new NotificacaoConsumer(notificacaoService, new ObjectMapper());
    }

    @Test
    void deveProcessarEventoClienteCriado() {
        when(templateRepository.findByCodigoAndAtivoTrue("cliente_criado")).thenReturn(Optional.of(
                buildTemplate("cliente_criado", "EMAIL", "Bem-vindo ao AURIX, {{nome}}!")));
        when(preferenciaRepository.findByClienteId(1L)).thenReturn(Optional.empty());
        when(filaRepository.save(any())).thenAnswer(inv -> {
            FilaNotificacao saved = inv.getArgument(0);
            saved.setId(1L);
            return saved;
        });

        consumer.onClienteCriado("{\"clienteId\":1,\"nome\":\"Joao\",\"email\":\"joao@test.com\"}");

        verify(templateRepository).findByCodigoAndAtivoTrue("cliente_criado");
        verify(filaRepository).save(any());
        verify(producer).notificacaoEnviada(any());
    }

    @Test
    void deveProcessarEventoKycAprovado() {
        when(templateRepository.findByCodigoAndAtivoTrue("kyc_aprovado")).thenReturn(Optional.of(
                buildTemplate("kyc_aprovado", "PUSH", "Seu KYC foi aprovado!")));
        when(preferenciaRepository.findByClienteId(2L)).thenReturn(Optional.empty());
        when(filaRepository.save(any())).thenAnswer(inv -> {
            FilaNotificacao saved = inv.getArgument(0);
            saved.setId(2L);
            return saved;
        });

        consumer.onKycAprovado("{\"clienteId\":2}");

        verify(filaRepository).save(any());
        verify(producer).notificacaoEnviada(any());
    }

    @Test
    void deveProcessarEventoKycRejeitadoComMotivo() {
        when(templateRepository.findByCodigoAndAtivoTrue("kyc_rejeitado")).thenReturn(Optional.of(
                buildTemplate("kyc_rejeitado", "EMAIL", "Seu KYC foi rejeitado. Motivo: {{motivo}}.")));
        when(preferenciaRepository.findByClienteId(3L)).thenReturn(Optional.empty());
        when(filaRepository.save(any())).thenAnswer(inv -> {
            FilaNotificacao saved = inv.getArgument(0);
            saved.setId(3L);
            return saved;
        });

        consumer.onKycRejeitado("{\"clienteId\":3,\"motivo\":\"Documento ilegivel\"}");

        verify(filaRepository).save(argThat(n -> n.getCorpoRenderizado().contains("Documento ilegivel")));
    }

    @Test
    void deveProcessarEventoFraudeTransacaoBloqueada() {
        when(templateRepository.findByCodigoAndAtivoTrue("fraude_transacao_bloqueada")).thenReturn(Optional.of(
                buildTemplate("fraude_transacao_bloqueada", "SMS", "Alerta: Transacao {{transacaoRef}} bloqueada.")));
        when(preferenciaRepository.findByClienteId(4L)).thenReturn(Optional.empty());
        when(filaRepository.save(any())).thenAnswer(inv -> {
            FilaNotificacao saved = inv.getArgument(0);
            saved.setId(4L);
            return saved;
        });

        consumer.onFraudeTransacaoBloqueada("{\"clienteId\":4,\"transacaoRef\":\"TXN-999\"}");

        verify(filaRepository).save(argThat(n -> n.getCorpoRenderizado().contains("TXN-999")));
    }

    @Test
    void deveIgnorarEventoSemClienteId() {
        consumer.onClienteCriado("{\"nome\":\"Teste\"}");

        verify(filaRepository, never()).save(any());
    }

    @Test
    void deveIgnorarMensagemInvalida() {
        assertDoesNotThrow(() -> consumer.onClienteCriado("not-json"));
    }

    private TemplateNotificacao buildTemplate(String codigo, String canal, String corpo) {
        TemplateNotificacao template = new TemplateNotificacao();
        template.setCodigo(codigo);
        template.setCanal(canal);
        template.setCorpo(corpo);
        template.setAssunto("Notificacao");
        template.setAtivo(true);
        return template;
    }
}
```

- [ ] **Step 3: Run tests**

Run: `mvn clean test -pl aurix-notification -am`
Expected: All tests PASS

---

### Task 5: NotificacaoController + HealthController

**Files:**
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/controller/NotificacaoController.java`
- Create: `apps/backend/aurix-notification/src/main/java/com/aurix/platform/notification/controller/HealthController.java`
- Create: `apps/backend/aurix-notification/src/test/java/com/aurix/platform/notification/controller/NotificacaoControllerTest.java`

**Interfaces:**
- Consumes: `NotificacaoService` (all public methods from Task 3)
- Produces: REST endpoints for external clients

- [ ] **Step 1: Create NotificacaoController.java**

```java
package com.aurix.platform.notification.controller;

import com.aurix.platform.notification.entity.FilaNotificacao;
import com.aurix.platform.notification.entity.PreferenciaCliente;
import com.aurix.platform.notification.entity.TemplateNotificacao;
import com.aurix.platform.notification.service.NotificacaoService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/notificacoes")
@Tag(name = "Notification", description = "Multi-channel notification management")
public class NotificacaoController {
    private final NotificacaoService service;

    public NotificacaoController(NotificacaoService service) {
        this.service = service;
    }

    @PostMapping("/enviar")
    @Operation(summary = "Enviar notificacao para um cliente")
    public ResponseEntity<FilaNotificacao> enviar(@RequestParam Long clienteId,
            @RequestParam String templateCodigo, @RequestParam String destinatario,
            @RequestBody(required = false) Map<String, String> variaveis) {
        FilaNotificacao resultado = service.enviar(clienteId, templateCodigo, destinatario,
                variaveis != null ? variaveis : Map.of());
        if (resultado == null) {
            return ResponseEntity.status(HttpStatus.OK)
                    .body(null);
        }
        return ResponseEntity.status(HttpStatus.CREATED).body(resultado);
    }

    @GetMapping("/cliente/{clienteId}")
    @Operation(summary = "Listar notificacoes de um cliente")
    public ResponseEntity<List<FilaNotificacao>> listarPorCliente(@PathVariable Long clienteId) {
        return ResponseEntity.ok(service.listarNotificacoesPorCliente(clienteId));
    }

    @PostMapping("/templates")
    @Operation(summary = "Criar template de notificacao")
    public ResponseEntity<TemplateNotificacao> criarTemplate(@RequestBody TemplateNotificacao template) {
        return ResponseEntity.status(HttpStatus.CREATED).body(service.criarTemplate(template));
    }

    @GetMapping("/templates")
    @Operation(summary = "Listar todos os templates")
    public ResponseEntity<List<TemplateNotificacao>> listarTemplates() {
        return ResponseEntity.ok(service.listarTemplates());
    }

    @PostMapping("/preferencias")
    @Operation(summary = "Salvar preferencias de notificacao do cliente")
    public ResponseEntity<PreferenciaCliente> salvarPreferencia(
            @RequestBody PreferenciaCliente preferencia) {
        return ResponseEntity.ok(service.salvarPreferencia(preferencia));
    }

    @GetMapping("/preferencias/{clienteId}")
    @Operation(summary = "Buscar preferencias de notificacao do cliente")
    public ResponseEntity<PreferenciaCliente> buscarPreferencia(@PathVariable Long clienteId) {
        return service.buscarPreferencia(clienteId)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping("/renderizar")
    @Operation(summary = "Renderizar template com variaveis (preview)")
    public ResponseEntity<Map<String, String>> renderizar(@RequestParam String template,
            @RequestBody Map<String, String> variaveis) {
        String resultado = service.renderizar(template, variaveis);
        return ResponseEntity.ok(Map.of("renderizado", resultado));
    }
}
```

- [ ] **Step 2: Create HealthController.java**

```java
package com.aurix.platform.notification.controller;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import java.time.LocalDateTime;
import java.util.Map;

@RestController
@RequestMapping("/health")
@Tag(name = "Health", description = "API para verificacao de saude do servico Notification")
public class HealthController {
    @GetMapping
    @Operation(summary = "Health check", description = "Verifica se o servico Notification esta funcionando")
    public ResponseEntity<Map<String, Object>> health() {
        return ResponseEntity.ok(Map.of(
            "status", "UP",
            "service", "aurix-notification",
            "timestamp", LocalDateTime.now().toString(),
            "version", "1.0.0"
        ));
    }
}
```

- [ ] **Step 3: Create NotificacaoControllerTest.java**

```java
package com.aurix.platform.notification.controller;

import com.aurix.platform.notification.entity.FilaNotificacao;
import com.aurix.platform.notification.entity.PreferenciaCliente;
import com.aurix.platform.notification.entity.TemplateNotificacao;
import com.aurix.platform.notification.service.NotificacaoService;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class NotificacaoControllerTest {
    @Mock private NotificacaoService service;
    @InjectMocks private NotificacaoController controller;

    @Test
    void deveEnviarNotificacaoComSucesso() {
        FilaNotificacao notificacao = new FilaNotificacao();
        notificacao.setId(1L);
        notificacao.setStatus("ENVIADA");

        when(service.enviar(eq(1L), eq("cliente.criado"), eq("joao@test.com"), any()))
                .thenReturn(notificacao);

        var response = controller.enviar(1L, "cliente.criado", "joao@test.com", Map.of("nome", "Joao"));

        assertEquals(201, response.getStatusCode().value());
        assertEquals("ENVIADA", response.getBody().getStatus());
    }

    @Test
    void deveRetornarNullQuandoNotificacaoPulada() {
        when(service.enviar(eq(1L), eq("cliente.criado"), eq("joao@test.com"), any()))
                .thenReturn(null);

        var response = controller.enviar(1L, "cliente.criado", "joao@test.com", null);

        assertEquals(200, response.getStatusCode().value());
        assertNull(response.getBody());
    }

    @Test
    void deveListarNotificacoesPorCliente() {
        FilaNotificacao n1 = new FilaNotificacao();
        n1.setId(1L);
        when(service.listarNotificacoesPorCliente(1L)).thenReturn(List.of(n1));

        var response = controller.listarPorCliente(1L);

        assertEquals(200, response.getStatusCode().value());
        assertEquals(1, response.getBody().size());
    }

    @Test
    void deveCriarTemplate() {
        TemplateNotificacao template = new TemplateNotificacao();
        template.setCodigo("teste");
        when(service.criarTemplate(any())).thenReturn(template);

        var response = controller.criarTemplate(template);

        assertEquals(201, response.getStatusCode().value());
        assertEquals("teste", response.getBody().getCodigo());
    }

    @Test
    void deveListarTemplates() {
        when(service.listarTemplates()).thenReturn(List.of());
        var response = controller.listarTemplates();
        assertEquals(200, response.getStatusCode().value());
        assertTrue(response.getBody().isEmpty());
    }

    @Test
    void deveSalvarPreferencia() {
        PreferenciaCliente pref = new PreferenciaCliente();
        pref.setClienteId(1L);
        when(service.salvarPreferencia(any())).thenReturn(pref);

        var response = controller.salvarPreferencia(pref);
        assertEquals(200, response.getStatusCode().value());
        assertEquals(1L, response.getBody().getClienteId());
    }

    @Test
    void deveBuscarPreferenciaExistente() {
        PreferenciaCliente pref = new PreferenciaCliente();
        when(service.buscarPreferencia(1L)).thenReturn(Optional.of(pref));
        var response = controller.buscarPreferencia(1L);
        assertEquals(200, response.getStatusCode().value());
    }

    @Test
    void deveRetornar404QuandoPreferenciaNaoEncontrada() {
        when(service.buscarPreferencia(99L)).thenReturn(Optional.empty());
        var response = controller.buscarPreferencia(99L);
        assertEquals(404, response.getStatusCode().value());
    }

    @Test
    void deveRenderizarTemplate() {
        when(service.renderizar(eq("Ola {{nome}}!"), any())).thenReturn("Ola Joao!");
        var response = controller.renderizar("Ola {{nome}}!", Map.of("nome", "Joao"));
        assertEquals(200, response.getStatusCode().value());
        assertEquals("Ola Joao!", response.getBody().get("renderizado"));
    }
}
```

- [ ] **Step 4: Run all tests**

Run: `mvn clean test -pl aurix-notification -am`
Expected: All tests PASS

---

### Task 6: Final verification and commit

**Files:**
- Verify: all files created in Tasks 1–5

- [ ] **Step 1: Run full test suite**

Run: `mvn clean test -pl aurix-notification -am`
Expected: All tests PASS, 0 failures

- [ ] **Step 2: Verify static analysis**

Run: `mvn pmd:check -pl aurix-notification -am`
Expected: PASS

- [ ] **Step 3: Verify checkstyle**

Run: `mvn checkstyle:check -pl aurix-notification -am`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add apps/backend/aurix-notification/
git commit -m "feat(notification): full module with template rendering, Kafka consumers, and channel dispatch"
```
