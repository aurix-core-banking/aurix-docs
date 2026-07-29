# Catálogo de Ofertas — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create new `aurix-catalog` module — a multi-audience offer engine with taxonomy navigation, product registration with typed configs, commercial offers, reusable eligibility policies, campaigns, channels, and client segments.

**Architecture:** Six-layer separation — Catálogo (taxonomy, pure navigation) → Produto (identity + typed configs per modalidade) → Oferta (commercial conditions) → Elegibilidade (reusable policies + rules) → Campanha/Canal/Segmento (transversal groupings). Module depends only on `aurix-shared`.

**Tech Stack:** Java 25, Spring Boot 4.1.0, Spring Data JPA, H2 (test), PostgreSQL (prod), Spring Security, Spring Validation.

## Global Constraints

- No Lombok — manual getters/setters/equals/hashCode/toString
- No MapStruct — manual DTO conversion
- `@java.lang.SuppressWarnings("all")` on every accessor (matching codebase convention)
- `BaseEntity` from `aurix-shared` provides: `id`, `tenantId`, `dataCriacao`, `dataAtualizacao`, `versao`
- Schema: `aurix` (PostgreSQL), table prefix: `catalog_`
- Test pattern: `@SpringBootTest(webEnvironment = RANDOM_PORT)`, `@ActiveProfiles("test")`, `RestTemplate` with `NoOpResponseErrorHandler`, H2
- PRIME=59 on hashCode (codebase convention)
- All CRUD endpoints return proper HTTP status (201 POST, 200 GET/PUT, 204 DELETE)
- Soft delete via `deletedAt` field (nullable)

---

### Task 1: Module Scaffolding

**Files:**
- Create: `apps/backend/aurix-catalog/pom.xml`
- Create: `apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/AurixCatalogApplication.java`
- Create: `apps/backend/aurix-catalog/src/main/resources/application.yml`
- Create: `apps/backend/aurix-catalog/src/main/resources/application-test.yml`
- Create: `apps/backend/aurix-catalog/src/test/java/com/aurix/platform/catalog/config/TestSecurityConfig.java`
- Modify: `apps/backend/pom.xml` (add module + dependency management entry)

**Interfaces:**
- Consumes: `aurix-shared` (BaseEntity, shared utilities)
- Produces: compilable module skeleton

- [ ] **Step 1: Create `apps/backend/aurix-catalog/pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.aurix.platform</groupId>
        <artifactId>aurix-platform</artifactId>
        <version>1.0.0</version>
    </parent>
    <artifactId>aurix-catalog</artifactId>
    <packaging>jar</packaging>
    <name>AURIX Catalog</name>
    <description>Motor de Ofertas Multi-Público — Catálogo, Produtos, Ofertas, Elegibilidade</description>
    <properties>
        <maven.compiler.source>25</maven.compiler.source>
        <maven.compiler.target>25</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
    <dependencies>
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
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.0.2</version>
        </dependency>
        <dependency>
            <groupId>com.aurix.platform</groupId>
            <artifactId>aurix-shared</artifactId>
            <version>1.0.0</version>
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

- [ ] **Step 2: Create `AurixCatalogApplication.java`**

```java
package com.aurix.platform.catalog;

import io.swagger.v3.oas.annotations.OpenAPIDefinition;
import io.swagger.v3.oas.annotations.info.Info;
import io.swagger.v3.oas.annotations.info.Contact;
import io.swagger.v3.oas.annotations.info.License;
import io.swagger.v3.oas.annotations.servers.Server;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@OpenAPIDefinition(
    info = @Info(
        title = "AURIX Catalog API",
        version = "1.0.0",
        description = "Motor de Ofertas Multi-Público — Catálogo, Produtos, Ofertas e Elegibilidade",
        contact = @Contact(
            name = "AURIX Platform Team",
            email = "dev@aurix.platform",
            url = "https://aurix.platform"
        ),
        license = @License(
            name = "MIT License",
            url = "https://opensource.org/licenses/MIT"
        )
    ),
    servers = {
        @Server(url = "http://localhost:8090/api/catalog", description = "Servidor de Desenvolvimento"),
        @Server(url = "https://api.aurix.platform/catalog", description = "Servidor de Produção")
    }
)
public class AurixCatalogApplication {
    public static void main(String[] args) {
        SpringApplication.run(AurixCatalogApplication.class, args);
    }
}
```

- [ ] **Step 3: Create `application.yml`**

```yaml
server:
  port: 8090
  servlet:
    context-path: /api/catalog

spring:
  application:
    name: aurix-catalog
  datasource:
    url: jdbc:postgresql://localhost:5432/aurix_catalog
    username: aurix
    password: aurix
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false
    properties:
      hibernate:
        default_schema: aurix
        format_sql: true
  jackson:
    serialization:
      write-dates-as-timestamps: false
    date-format: yyyy-MM-dd'T'HH:mm:ss

springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html

logging:
  level:
    com.aurix: DEBUG
```

- [ ] **Step 4: Create `application-test.yml`**

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop
    database-platform: org.hibernate.dialect.H2Dialect
    show-sql: true
  security:
    enabled: false
```

- [ ] **Step 5: Create `TestSecurityConfig.java`**

```java
package com.aurix.platform.catalog.config;

import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@TestConfiguration
public class TestSecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth.anyRequest().permitAll());
        return http.build();
    }
}
```

- [ ] **Step 6: Modify `apps/backend/pom.xml`**

Add `<module>aurix-catalog</module>` to the `<modules>` list (insert alphabetically between `aurix-baas` and `aurix-billing`) and add dependency management entry:

```xml
<!-- AURIX Catalog -->
<dependency>
    <groupId>com.aurix.platform</groupId>
    <artifactId>aurix-catalog</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 7: Verify compilation**

Run: `mvn compile -pl aurix-catalog -am` from `apps/backend/`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add apps/backend/pom.xml apps/backend/aurix-catalog/
git commit -m "feat(catalog): add aurix-catalog module scaffolding"
```

---

### Task 2: Enums + DTO Base + Catalog Layer Entities (Segmento → CategoriaProduto)

**Files:**
- Create: `apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/enums/StatusCatalogo.java`
- Create: `apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/enums/StatusProduto.java`
- Create: `apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/enums/StatusOferta.java`
- Create: `apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/enums/StatusPolitica.java`
- Create: `apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/enums/StatusCampanha.java`
- Create: `apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/enums/TipoProduto.java`
- Create: `apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/enums/SistemaExterno.java`
- Create: `apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/enums/TipoVinculoElegibilidade.java`
- Create: `apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/enums/TipoRenda.java`
- Create: 5 entity files for catalog layer (Segmento, LinhaNegocio, FamiliaProduto, Categoria, CategoriaProduto)
- Create: 5 DTO files mirroring entities
- Create: 5 repository files
- Create: `CatalogoService.java` (taxonomy CRUD + tree query)
- Create: `CatalogoController.java` (tree navigation endpoints)
- Create: `exception/RecursoNaoEncontradoException.java`

**Interfaces:**
- Produces: compilable taxonomy CRUD with `/api/catalog/segmentos`, `/api/catalog/linhas-negocio`, `/api/catalog/familias`, `/api/catalog/categorias`, `/api/catalog/arvore` endpoints

- [ ] **Step 1: Create all enum files**

`StatusCatalogo.java`:
```java
package com.aurix.platform.catalog.enums;

public enum StatusCatalogo {
    ATIVO, INATIVO
}
```

`StatusProduto.java`:
```java
package com.aurix.platform.catalog.enums;

public enum StatusProduto {
    RASCUNHO, ATIVO, INATIVO, OBSOLETO
}
```

`StatusOferta.java`:
```java
package com.aurix.platform.catalog.enums;

public enum StatusOferta {
    RASCUNHO, ATIVA, PAUSADA, ENCERRADA
}
```

`StatusPolitica.java`:
```java
package com.aurix.platform.catalog.enums;

public enum StatusPolitica {
    ATIVA, INATIVA
}
```

`StatusCampanha.java`:
```java
package com.aurix.platform.catalog.enums;

public enum StatusCampanha {
    RASCUNHO, ATIVA, ENCERRADA
}
```

`TipoProduto.java`:
```java
package com.aurix.platform.catalog.enums;

public enum TipoProduto {
    CREDITO, INVESTIMENTO, SEGURO, CARTAO, CONTA, CAMBIO,
    PAGAMENTO, BENEFICIO, CONSORCIO, PREVIDENCIA, SERVICO
}
```

`SistemaExterno.java`:
```java
package com.aurix.platform.catalog.enums;

public enum SistemaExterno {
    CORE_BANKING, CRM, OPEN_FINANCE, ERP, MOTOR_CREDITO,
    MOTOR_ANTIFRAUDE, SAP, SALESFORCE, LEGADO
}
```

`TipoVinculoElegibilidade.java`:
```java
package com.aurix.platform.catalog.enums;

public enum TipoVinculoElegibilidade {
    OBRIGATORIA, OPCIONAL
}
```

`TipoRenda.java`:
```java
package com.aurix.platform.catalog.enums;

public enum TipoRenda {
    FIXA, VARIAVEL, HIBRIDA
}
```

- [ ] **Step 2: Create `Segmento.java` entity**

```java
package com.aurix.platform.catalog.entity;

import com.aurix.platform.catalog.enums.StatusCatalogo;
import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Table;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "catalog_segmentos", schema = "aurix")
public class Segmento extends BaseEntity {

    @NotBlank
    @Column(name = "codigo", nullable = false, unique = true, length = 50)
    private String codigo;

    @NotBlank
    @Column(name = "nome", nullable = false, length = 255)
    private String nome;

    @Column(name = "descricao", length = 1000)
    private String descricao;

    @Column(name = "icone", length = 255)
    private String icone;

    @Column(name = "ordem", nullable = false)
    private Integer ordem;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private StatusCatalogo status;

    @Column(name = "vigencia_inicio", nullable = false)
    private LocalDate vigenciaInicio;

    @Column(name = "vigencia_fim")
    private LocalDate vigenciaFim;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    @java.lang.SuppressWarnings("all")
    public String getCodigo() { return this.codigo; }

    @java.lang.SuppressWarnings("all")
    public void setCodigo(final String codigo) { this.codigo = codigo; }

    @java.lang.SuppressWarnings("all")
    public String getNome() { return this.nome; }

    @java.lang.SuppressWarnings("all")
    public void setNome(final String nome) { this.nome = nome; }

    @java.lang.SuppressWarnings("all")
    public String getDescricao() { return this.descricao; }

    @java.lang.SuppressWarnings("all")
    public void setDescricao(final String descricao) { this.descricao = descricao; }

    @java.lang.SuppressWarnings("all")
    public String getIcone() { return this.icone; }

    @java.lang.SuppressWarnings("all")
    public void setIcone(final String icone) { this.icone = icone; }

    @java.lang.SuppressWarnings("all")
    public Integer getOrdem() { return this.ordem; }

    @java.lang.SuppressWarnings("all")
    public void setOrdem(final Integer ordem) { this.ordem = ordem; }

    @java.lang.SuppressWarnings("all")
    public StatusCatalogo getStatus() { return this.status; }

    @java.lang.SuppressWarnings("all")
    public void setStatus(final StatusCatalogo status) { this.status = status; }

    @java.lang.SuppressWarnings("all")
    public LocalDate getVigenciaInicio() { return this.vigenciaInicio; }

    @java.lang.SuppressWarnings("all")
    public void setVigenciaInicio(final LocalDate vigenciaInicio) { this.vigenciaInicio = vigenciaInicio; }

    @java.lang.SuppressWarnings("all")
    public LocalDate getVigenciaFim() { return this.vigenciaFim; }

    @java.lang.SuppressWarnings("all")
    public void setVigenciaFim(final LocalDate vigenciaFim) { this.vigenciaFim = vigenciaFim; }

    @java.lang.SuppressWarnings("all")
    public LocalDateTime getCreatedAt() { return this.createdAt; }

    @java.lang.SuppressWarnings("all")
    public void setCreatedAt(final LocalDateTime createdAt) { this.createdAt = createdAt; }

    @java.lang.SuppressWarnings("all")
    public LocalDateTime getUpdatedAt() { return this.updatedAt; }

    @java.lang.SuppressWarnings("all")
    public void setUpdatedAt(final LocalDateTime updatedAt) { this.updatedAt = updatedAt; }

    @java.lang.SuppressWarnings("all")
    public LocalDateTime getDeletedAt() { return this.deletedAt; }

    @java.lang.SuppressWarnings("all")
    public void setDeletedAt(final LocalDateTime deletedAt) { this.deletedAt = deletedAt; }
}
```

Create `LinhaNegocio.java`, `FamiliaProduto.java`, `Categoria.java` following the same pattern but with FK fields to parent (segmentoId, linhaNegocioId, familiaId respectively). Each has the same base fields: codigo, nome, descricao, icone, ordem, status, vigenciaInicio/Fim, createdAt/updatedAt/deletedAt. The FK must have `@Column(nullable = false)`.

- [ ] **Step 3: Create `CategoriaProduto.java` (N:N association)**

```java
package com.aurix.platform.catalog.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import jakarta.persistence.UniqueConstraint;
import java.time.LocalDateTime;

@Entity
@Table(name = "catalog_categorias_produtos", schema = "aurix",
       uniqueConstraints = @UniqueConstraint(columnNames = {"categoria_id", "produto_id"}))
public class CategoriaProduto {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "categoria_id", nullable = false)
    private Long categoriaId;

    @Column(name = "produto_id", nullable = false)
    private Long produtoId;

    @Column(name = "ordem")
    private Integer ordem;

    @Column(name = "destaque", nullable = false)
    private Boolean destaque = false;

    @Column(name = "ativo", nullable = false)
    private Boolean ativo = true;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @java.lang.SuppressWarnings("all")
    public Long getId() { return this.id; }

    @java.lang.SuppressWarnings("all")
    public void setId(final Long id) { this.id = id; }

    @java.lang.SuppressWarnings("all")
    public Long getCategoriaId() { return this.categoriaId; }

    @java.lang.SuppressWarnings("all")
    public void setCategoriaId(final Long categoriaId) { this.categoriaId = categoriaId; }

    @java.lang.SuppressWarnings("all")
    public Long getProdutoId() { return this.produtoId; }

    @java.lang.SuppressWarnings("all")
    public void setProdutoId(final Long produtoId) { this.produtoId = produtoId; }

    @java.lang.SuppressWarnings("all")
    public Integer getOrdem() { return this.ordem; }

    @java.lang.SuppressWarnings("all")
    public void setOrdem(final Integer ordem) { this.ordem = ordem; }

    @java.lang.SuppressWarnings("all")
    public Boolean getDestaque() { return this.destaque; }

    @java.lang.SuppressWarnings("all")
    public void setDestaque(final Boolean destaque) { this.destaque = destaque; }

    @java.lang.SuppressWarnings("all")
    public Boolean getAtivo() { return this.ativo; }

    @java.lang.SuppressWarnings("all")
    public void setAtivo(final Boolean ativo) { this.ativo = ativo; }

    @java.lang.SuppressWarnings("all")
    public LocalDateTime getCreatedAt() { return this.createdAt; }

    @java.lang.SuppressWarnings("all")
    public void setCreatedAt(final LocalDateTime createdAt) { this.createdAt = createdAt; }

    @java.lang.SuppressWarnings("all")
    public LocalDateTime getUpdatedAt() { return this.updatedAt; }

    @java.lang.SuppressWarnings("all")
    public void setUpdatedAt(final LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
}
```

- [ ] **Step 4: Create DTOs for all 5 catalog entities**

`SegmentoDTO.java` in `com.aurix.platform.catalog.dto`:
```java
package com.aurix.platform.catalog.dto;

import com.aurix.platform.catalog.enums.StatusCatalogo;
import java.time.LocalDate;
import java.time.LocalDateTime;

public class SegmentoDTO {
    private Long id;
    private String codigo;
    private String nome;
    private String descricao;
    private String icone;
    private Integer ordem;
    private StatusCatalogo status;
    private LocalDate vigenciaInicio;
    private LocalDate vigenciaFim;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    @java.lang.SuppressWarnings("all")
    public Long getId() { return this.id; }

    @java.lang.SuppressWarnings("all")
    public void setId(final Long id) { this.id = id; }

    @java.lang.SuppressWarnings("all")
    public String getCodigo() { return this.codigo; }

    @java.lang.SuppressWarnings("all")
    public void setCodigo(final String codigo) { this.codigo = codigo; }

    @java.lang.SuppressWarnings("all")
    public String getNome() { return this.nome; }

    @java.lang.SuppressWarnings("all")
    public void setNome(final String nome) { this.nome = nome; }

    @java.lang.SuppressWarnings("all")
    public String getDescricao() { return this.descricao; }

    @java.lang.SuppressWarnings("all")
    public void setDescricao(final String descricao) { this.descricao = descricao; }

    @java.lang.SuppressWarnings("all")
    public String getIcone() { return this.icone; }

    @java.lang.SuppressWarnings("all")
    public void setIcone(final String icone) { this.icone = icone; }

    @java.lang.SuppressWarnings("all")
    public Integer getOrdem() { return this.ordem; }

    @java.lang.SuppressWarnings("all")
    public void setOrdem(final Integer ordem) { this.ordem = ordem; }

    @java.lang.SuppressWarnings("all")
    public StatusCatalogo getStatus() { return this.status; }

    @java.lang.SuppressWarnings("all")
    public void setStatus(final StatusCatalogo status) { this.status = status; }

    @java.lang.SuppressWarnings("all")
    public LocalDate getVigenciaInicio() { return this.vigenciaInicio; }

    @java.lang.SuppressWarnings("all")
    public void setVigenciaInicio(final LocalDate vigenciaInicio) { this.vigenciaInicio = vigenciaInicio; }

    @java.lang.SuppressWarnings("all")
    public LocalDate getVigenciaFim() { return this.vigenciaFim; }

    @java.lang.SuppressWarnings("all")
    public void setVigenciaFim(final LocalDate vigenciaFim) { this.vigenciaFim = vigenciaFim; }

    @java.lang.SuppressWarnings("all")
    public LocalDateTime getCreatedAt() { return this.createdAt; }

    @java.lang.SuppressWarnings("all")
    public void setCreatedAt(final LocalDateTime createdAt) { this.createdAt = createdAt; }

    @java.lang.SuppressWarnings("all")
    public LocalDateTime getUpdatedAt() { return this.updatedAt; }

    @java.lang.SuppressWarnings("all")
    public void setUpdatedAt(final LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
}
```

Create `LinhaNegocioDTO`, `FamiliaProdutoDTO`, `CategoriaDTO`, `CategoriaProdutoDTO` following the same pattern with their respective FK fields.

- [ ] **Step 5: Create repositories**

`SegmentoRepository.java`:
```java
package com.aurix.platform.catalog.repository;

import com.aurix.platform.catalog.entity.Segmento;
import com.aurix.platform.catalog.enums.StatusCatalogo;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;
import java.util.Optional;

public interface SegmentoRepository extends JpaRepository<Segmento, Long> {
    Optional<Segmento> findByCodigo(String codigo);
    List<Segmento> findByStatusOrderByOrdem(StatusCatalogo status);
    List<Segmento> findByDeletedAtIsNullOrderByOrdem();
    boolean existsByCodigo(String codigo);
}
```

Create `LinhaNegocioRepository`, `FamiliaProdutoRepository`, `CategoriaRepository`, `CategoriaProdutoRepository` following the same pattern. Each with `findBy{ParentId}OrderByOrdem` and `existsByCodigo`.

- [ ] **Step 6: Create `RecursoNaoEncontradoException.java`**

```java
package com.aurix.platform.catalog.exception;

public class RecursoNaoEncontradoException extends RuntimeException {
    public RecursoNaoEncontradoException(String recurso, Long id) {
        super(recurso + " não encontrado: " + id);
    }
    public RecursoNaoEncontradoException(String recurso, String codigo) {
        super(recurso + " não encontrado: " + codigo);
    }
}
```

- [ ] **Step 7: Create `CatalogoService.java`**

```java
package com.aurix.platform.catalog.service;

import com.aurix.platform.catalog.dto.CategoriaDTO;
import com.aurix.platform.catalog.dto.CategoriaProdutoDTO;
import com.aurix.platform.catalog.dto.FamiliaProdutoDTO;
import com.aurix.platform.catalog.dto.LinhaNegocioDTO;
import com.aurix.platform.catalog.dto.SegmentoDTO;
import com.aurix.platform.catalog.entity.Categoria;
import com.aurix.platform.catalog.entity.CategoriaProduto;
import com.aurix.platform.catalog.entity.FamiliaProduto;
import com.aurix.platform.catalog.entity.LinhaNegocio;
import com.aurix.platform.catalog.entity.Segmento;
import com.aurix.platform.catalog.enums.StatusCatalogo;
import com.aurix.platform.catalog.exception.RecursoNaoEncontradoException;
import com.aurix.platform.catalog.repository.CategoriaProdutoRepository;
import com.aurix.platform.catalog.repository.CategoriaRepository;
import com.aurix.platform.catalog.repository.FamiliaProdutoRepository;
import com.aurix.platform.catalog.repository.LinhaNegocioRepository;
import com.aurix.platform.catalog.repository.SegmentoRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Service
@Transactional
public class CatalogoService {

    private final SegmentoRepository segmentoRepository;
    private final LinhaNegocioRepository linhaNegocioRepository;
    private final FamiliaProdutoRepository familiaProdutoRepository;
    private final CategoriaRepository categoriaRepository;
    private final CategoriaProdutoRepository categoriaProdutoRepository;

    public CatalogoService(SegmentoRepository segmentoRepository,
                           LinhaNegocioRepository linhaNegocioRepository,
                           FamiliaProdutoRepository familiaProdutoRepository,
                           CategoriaRepository categoriaRepository,
                           CategoriaProdutoRepository categoriaProdutoRepository) {
        this.segmentoRepository = segmentoRepository;
        this.linhaNegocioRepository = linhaNegocioRepository;
        this.familiaProdutoRepository = familiaProdutoRepository;
        this.categoriaRepository = categoriaRepository;
        this.categoriaProdutoRepository = categoriaProdutoRepository;
    }

    // Segmento CRUD
    public List<Segmento> listarSegmentos() {
        return segmentoRepository.findByDeletedAtIsNullOrderByOrdem();
    }

    public Segmento buscarSegmento(Long id) {
        return segmentoRepository.findById(id)
            .orElseThrow(() -> new RecursoNaoEncontradoException("Segmento", id));
    }

    public Segmento criarSegmento(Segmento segmento) {
        segmento.setCreatedAt(LocalDateTime.now());
        if (segmento.getVigenciaInicio() == null) {
            segmento.setVigenciaInicio(LocalDate.now());
        }
        if (segmento.getStatus() == null) {
            segmento.setStatus(StatusCatalogo.ATIVO);
        }
        return segmentoRepository.save(segmento);
    }

    public Segmento atualizarSegmento(Long id, Segmento dados) {
        Segmento segmento = buscarSegmento(id);
        segmento.setCodigo(dados.getCodigo());
        segmento.setNome(dados.getNome());
        segmento.setDescricao(dados.getDescricao());
        segmento.setIcone(dados.getIcone());
        segmento.setOrdem(dados.getOrdem());
        segmento.setStatus(dados.getStatus());
        segmento.setVigenciaInicio(dados.getVigenciaInicio());
        segmento.setVigenciaFim(dados.getVigenciaFim());
        segmento.setUpdatedAt(LocalDateTime.now());
        return segmentoRepository.save(segmento);
    }

    public void removerSegmento(Long id) {
        Segmento segmento = buscarSegmento(id);
        segmento.setDeletedAt(LocalDateTime.now());
        segmentoRepository.save(segmento);
    }

    // LinhaNegocio CRUD
    public List<LinhaNegocio> listarLinhasPorSegmento(Long segmentoId) {
        return linhaNegocioRepository.findBySegmentoIdAndDeletedAtIsNullOrderByOrdem(segmentoId);
    }

    public LinhaNegocio buscarLinhaNegocio(Long id) {
        return linhaNegocioRepository.findById(id)
            .orElseThrow(() -> new RecursoNaoEncontradoException("LinhaNegocio", id));
    }

    public LinhaNegocio criarLinhaNegocio(LinhaNegocio linha) {
        linha.setCreatedAt(LocalDateTime.now());
        if (linha.getVigenciaInicio() == null) linha.setVigenciaInicio(LocalDate.now());
        if (linha.getStatus() == null) linha.setStatus(StatusCatalogo.ATIVO);
        return linhaNegocioRepository.save(linha);
    }

    // Similar methods for atualizarLinhaNegocio, removerLinhaNegocio
    // Similar CRUD for FamiliaProduto, Categoria, CategoriaProduto

    public List<FamiliaProduto> listarFamiliasPorLinha(Long linhaNegocioId) {
        return familiaProdutoRepository.findByLinhaNegocioIdAndDeletedAtIsNullOrderByOrdem(linhaNegocioId);
    }

    public List<Categoria> listarCategoriasPorFamilia(Long familiaId) {
        return categoriaRepository.findByFamiliaIdAndDeletedAtIsNullOrderByOrdem(familiaId);
    }

    // Full tree query
    public List<Map<String, Object>> arvoreCompleta() {
        List<Segmento> segmentos = segmentoRepository.findByDeletedAtIsNullOrderByOrdem();
        return segmentos.stream().map(seg -> {
            List<LinhaNegocio> linhas = linhaNegocioRepository
                .findBySegmentoIdAndDeletedAtIsNullOrderByOrdem(seg.getId());
            List<Map<String, Object>> linhasMap = linhas.stream().map(lin -> {
                List<FamiliaProduto> familias = familiaProdutoRepository
                    .findByLinhaNegocioIdAndDeletedAtIsNullOrderByOrdem(lin.getId());
                List<Map<String, Object>> famMap = familias.stream().map(fam -> {
                    List<Categoria> cats = categoriaRepository
                        .findByFamiliaIdAndDeletedAtIsNullOrderByOrdem(fam.getId());
                    return Map.of(
                        "id", fam.getId(), "codigo", fam.getCodigo(), "nome", fam.getNome(),
                        "categorias", cats.stream().map(cat -> Map.of(
                            "id", cat.getId(), "codigo", cat.getCodigo(), "nome", cat.getNome()
                        )).collect(Collectors.toList())
                    );
                }).collect(Collectors.toList());
                return Map.of(
                    "id", lin.getId(), "codigo", lin.getCodigo(), "nome", lin.getNome(),
                    "familias", famMap
                );
            }).collect(Collectors.toList());
            return Map.of(
                "id", seg.getId(), "codigo", seg.getCodigo(), "nome", seg.getNome(),
                "linhasNegocio", linhasMap
            );
        }).collect(Collectors.toList());
    }
}
```

- [ ] **Step 8: Create `CatalogoController.java`**

```java
package com.aurix.platform.catalog.controller;

import com.aurix.platform.catalog.dto.CategoriaDTO;
import com.aurix.platform.catalog.dto.CategoriaProdutoDTO;
import com.aurix.platform.catalog.dto.FamiliaProdutoDTO;
import com.aurix.platform.catalog.dto.LinhaNegocioDTO;
import com.aurix.platform.catalog.dto.SegmentoDTO;
import com.aurix.platform.catalog.entity.Categoria;
import com.aurix.platform.catalog.entity.CategoriaProduto;
import com.aurix.platform.catalog.entity.FamiliaProduto;
import com.aurix.platform.catalog.entity.LinhaNegocio;
import com.aurix.platform.catalog.entity.Segmento;
import com.aurix.platform.catalog.service.CatalogoService;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/catalog")
public class CatalogoController {

    private final CatalogoService catalogoService;

    public CatalogoController(CatalogoService catalogoService) {
        this.catalogoService = catalogoService;
    }

    @GetMapping("/segmentos")
    public ResponseEntity<List<Segmento>> listarSegmentos() {
        return ResponseEntity.ok(catalogoService.listarSegmentos());
    }

    @GetMapping("/segmentos/{id}")
    public ResponseEntity<Segmento> buscarSegmento(@PathVariable Long id) {
        return ResponseEntity.ok(catalogoService.buscarSegmento(id));
    }

    @PostMapping("/segmentos")
    public ResponseEntity<Segmento> criarSegmento(@RequestBody Segmento segmento) {
        return ResponseEntity.status(HttpStatus.CREATED).body(catalogoService.criarSegmento(segmento));
    }

    @PutMapping("/segmentos/{id}")
    public ResponseEntity<Segmento> atualizarSegmento(@PathVariable Long id, @RequestBody Segmento dados) {
        return ResponseEntity.ok(catalogoService.atualizarSegmento(id, dados));
    }

    @DeleteMapping("/segmentos/{id}")
    public ResponseEntity<Void> removerSegmento(@PathVariable Long id) {
        catalogoService.removerSegmento(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping("/arvore")
    public ResponseEntity<List<Map<String, Object>>> arvoreCompleta() {
        return ResponseEntity.ok(catalogoService.arvoreCompleta());
    }

    // Endpoints for LinhaNegocio, Familia, Categoria, CategoriaProduto follow same pattern
    @GetMapping("/linhas-negocio/{id}")
    public ResponseEntity<LinhaNegocio> buscarLinhaNegocio(@PathVariable Long id) {
        return ResponseEntity.ok(catalogoService.buscarLinhaNegocio(id));
    }

    @PostMapping("/linhas-negocio")
    public ResponseEntity<LinhaNegocio> criarLinhaNegocio(@RequestBody LinhaNegocio linha) {
        return ResponseEntity.status(HttpStatus.CREATED).body(catalogoService.criarLinhaNegocio(linha));
    }

    // ... remaining CRUD endpoints follow exact same pattern
    // DELETE /linhas-negocio/{id}, PUT /linhas-negocio/{id}
    // GET/POST/PUT/DELETE /familias/{id}, /categorias/{id}
    // POST/DELETE /categorias/{categoriaId}/produtos (for CategoriaProduto association)
}
```

- [ ] **Step 9: Compile verification**

Run: `mvn compile -pl aurix-catalog -am` from `apps/backend/`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/enums/
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/entity/
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/dto/
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/repository/
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/exception/
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/service/CatalogoService.java
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/controller/CatalogoController.java
git commit -m "feat(catalog): add enums + taxonomy layer (Segmento→CategoriaProduto)"
```

---

### Task 3: Produto Layer — Produto, ProdutoIntegracao, Modalidade, VersaoModalidade

**Files:**
- Create: `Produto.java` entity
- Create: `ProdutoIntegracao.java` entity
- Create: `Modalidade.java` entity
- Create: `VersaoModalidade.java` entity
- Create: DTOs for each
- Create: Repositories for each
- Create: `ProdutoService.java`
- Create: `ProdutoController.java`

**Interfaces:**
- Produces: `/api/catalog/produtos`, `/api/catalog/produtos/{id}/integracoes`, `/api/catalog/produtos/{id}/modalidades`, `/api/catalog/modalidades/{id}/versoes` endpoints

- [ ] **Step 1: Create `Produto.java`**

```java
package com.aurix.platform.catalog.entity;

import com.aurix.platform.catalog.enums.StatusProduto;
import com.aurix.platform.catalog.enums.TipoProduto;
import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Table;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "catalog_produtos", schema = "aurix")
public class Produto extends BaseEntity {

    @NotBlank
    @Column(name = "codigo", nullable = false, unique = true, length = 50)
    private String codigo;

    @NotBlank
    @Column(name = "nome", nullable = false, length = 255)
    private String nome;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(name = "tipo_produto", nullable = false, length = 30)
    private TipoProduto tipoProduto;

    @Column(name = "descricao", length = 2000)
    private String descricao;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private StatusProduto status;

    @Column(name = "vigencia_inicio", nullable = false)
    private LocalDate vigenciaInicio;

    @Column(name = "vigencia_fim")
    private LocalDate vigenciaFim;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    // getters/setters with @SuppressWarnings("all") for each field
    // same pattern as Segmento above
}
```

- [ ] **Step 2: Create `ProdutoIntegracao.java`**

```java
package com.aurix.platform.catalog.entity;

import com.aurix.platform.catalog.enums.SistemaExterno;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import java.time.LocalDateTime;

@Entity
@Table(name = "catalog_produtos_integracao", schema = "aurix")
public class ProdutoIntegracao {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "produto_id", nullable = false)
    private Long produtoId;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(name = "sistema", nullable = false, length = 30)
    private SistemaExterno sistema;

    @NotBlank
    @Column(name = "codigo_externo", nullable = false, length = 100)
    private String codigoExterno;

    @Column(name = "versao", length = 20)
    private String versao;

    @Column(name = "ativo", nullable = false)
    private Boolean ativo = true;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // getters/setters
}
```

- [ ] **Step 3: Create `Modalidade.java`**

```java
package com.aurix.platform.catalog.entity;

import com.aurix.platform.catalog.enums.StatusProduto;
import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Table;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "catalog_modalidades", schema = "aurix")
public class Modalidade extends BaseEntity {

    @Column(name = "produto_id", nullable = false)
    private Long produtoId;

    @NotBlank
    @Column(name = "codigo", nullable = false, length = 50)
    private String codigo;

    @NotBlank
    @Column(name = "nome", nullable = false, length = 255)
    private String nome;

    @Column(name = "tipo_modalidade", length = 50)
    private String tipoModalidade;

    @Column(name = "descricao", length = 1000)
    private String descricao;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private StatusProduto status;

    @Column(name = "vigencia_inicio", nullable = false)
    private LocalDate vigenciaInicio;

    @Column(name = "vigencia_fim")
    private LocalDate vigenciaFim;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    // getters/setters
}
```

- [ ] **Step 4: Create `VersaoModalidade.java`**

```java
package com.aurix.platform.catalog.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import jakarta.validation.constraints.Size;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "catalog_versoes_modalidade", schema = "aurix")
public class VersaoModalidade {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "modalidade_id", nullable = false)
    private Long modalidadeId;

    @Column(name = "versao", nullable = false)
    private Integer versao;

    @Column(name = "data_inicio", nullable = false)
    private LocalDate dataInicio;

    @Column(name = "data_fim")
    private LocalDate dataFim;

    @Size(max = 500)
    @Column(name = "alteracoes", length = 500)
    private String alteracoes;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    // getters/setters
}
```

- [ ] **Step 5: Create DTOs, repositories (following exact same pattern as Task 2)**

Repositories:
- `ProdutoRepository`: `findByCodigo`, `findByTipoProdutoAndDeletedAtIsNull`, `findByStatusAndDeletedAtIsNull`, `existsByCodigo`
- `ProdutoIntegracaoRepository`: `findByProdutoId`, `findBySistemaAndCodigoExterno`
- `ModalidadeRepository`: `findByProdutoIdAndDeletedAtIsNullOrderByCodigo`, `existsByCodigo`
- `VersaoModalidadeRepository`: `findByModalidadeIdOrderByVersaoDesc`

- [ ] **Step 6: Create `ProdutoService.java`**

CRUD for Produto (criar, buscar, listar com filtros, atualizar, remover), integrações (adicionar/remover), modalidades (criar/atualizar), versões (listar/criar).

- [ ] **Step 7: Create `ProdutoController.java`**

Endpoints:
- `GET /produtos` — list (optional query params: `tipo`, `status`)
- `GET /produtos/{id}` — get with modalidades + integracoes
- `POST /produtos` — create
- `PUT /produtos/{id}` — update
- `DELETE /produtos/{id}` — soft delete
- `POST /produtos/{id}/integracoes` — add integracao
- `DELETE /produtos/{id}/integracoes/{intId}` — remove
- `POST /produtos/{id}/modalidades` — create modalidade
- `PUT /modalidades/{id}` — update modalidade
- `GET /modalidades/{id}/versoes` — list versions

- [ ] **Step 8: Compile and commit**

```bash
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/entity/Produto.java
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/entity/ProdutoIntegracao.java
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/entity/Modalidade.java
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/entity/VersaoModalidade.java
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/dto/
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/repository/
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/service/ProdutoService.java
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/controller/ProdutoController.java
git commit -m "feat(catalog): add Produto + Modalidade layer"
```

---

### Task 4: Config Entities (Table-Per-Type)

**Files:**
- Create: `entity/config/ConfigCredito.java`
- Create: `entity/config/ConfigCartao.java`
- Create: `entity/config/ConfigConta.java`
- Create: `entity/config/ConfigInvestimento.java`
- Create: `entity/config/ConfigSeguro.java`
- Create: DTOs for each
- Create: Repositories for each
- Create: `ConfigService.java`
- Extend: `ProdutoController.java` (add config endpoints)
- Extend: `ProdutoService.java` (add config CRUD)

**Interfaces:**
- Produces: `POST /api/catalog/modalidades/{id}/config` (creates/updates typed config), `GET /api/catalog/modalidades/{id}/config`

- [ ] **Step 1: Create `ConfigCredito.java`**

```java
package com.aurix.platform.catalog.entity.config;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "catalog_config_credito", schema = "aurix")
public class ConfigCredito {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "modalidade_id", nullable = false, unique = true)
    private Long modalidadeId;

    @Column(name = "idade_minima")
    private Integer idadeMinima;

    @Column(name = "idade_maxima")
    private Integer idadeMaxima;

    @Column(name = "prazo_minimo_meses")
    private Integer prazoMinimoMeses;

    @Column(name = "prazo_maximo_meses")
    private Integer prazoMaximoMeses;

    @Column(name = "valor_minimo", precision = 18, scale = 2)
    private BigDecimal valorMinimo;

    @Column(name = "valor_maximo", precision = 18, scale = 2)
    private BigDecimal valorMaximo;

    @Column(name = "exige_garantia")
    private Boolean exigeGarantia;

    @Column(name = "taxa_juros_base", precision = 10, scale = 4)
    private BigDecimal taxaJurosBase;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // getters/setters for all fields with @SuppressWarnings("all")
}
```

- [ ] **Step 2-5: Create remaining config entities with same pattern**

- `ConfigCartao`: modalidadeId, bandeiras (String, JSON), anuidadeBase, programaPontos, limiteMinimo, limiteMaximo
- `ConfigConta`: modalidadeId, tipoConta, tarifaMensal, aceitaVariosPorCliente
- `ConfigInvestimento`: modalidadeId, tipoRenda (TipoRenda enum), aplicacaoMinima, carenciaDias, vencimentoPadraoDias
- `ConfigSeguro`: modalidadeId, coberturasBase (String, JSONB), premioMinimo

- [ ] **Step 6: Create repositories for all 5 config entities**

Each with `findByModalidadeId` returning `Optional<Config*>`.

- [ ] **Step 7: Create `ConfigService.java`**

```java
package com.aurix.platform.catalog.service;

import com.aurix.platform.catalog.entity.config.*;
import com.aurix.platform.catalog.enums.TipoProduto;
import com.aurix.platform.catalog.repository.config.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDateTime;

@Service
@Transactional
public class ConfigService {

    private final ConfigCreditoRepository configCreditoRepository;
    private final ConfigCartaoRepository configCartaoRepository;
    private final ConfigContaRepository configContaRepository;
    private final ConfigInvestimentoRepository configInvestimentoRepository;
    private final ConfigSeguroRepository configSeguroRepository;

    // constructor injection

    public Object criarOuAtualizarConfig(Long modalidadeId, TipoProduto tipoProduto, Object configDto) {
        // dispatch by tipoProduto to correct config type
        // if exists → update, else create
        return null; // placeholder
    }
}
```

- [ ] **Step 8: Extend `ProdutoController.java`** with:

```java
@PostMapping("/modalidades/{id}/config")
public ResponseEntity<Object> criarConfigModalidade(@PathVariable Long id, @RequestBody Object configDto) {
    // resolve TipoProduto from modalidade, then delegate to ConfigService
    return ResponseEntity.status(HttpStatus.CREATED).body(...);
}

@GetMapping("/modalidades/{id}/config")
public ResponseEntity<Object> buscarConfigModalidade(@PathVariable Long id) {
    return ResponseEntity.ok(...);
}
```

- [ ] **Step 9: Compile and commit**

```bash
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/entity/config/
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/dto/
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/repository/config/
git add apps/backend/aurix-catalog/src/main/java/com/aurix/platform/catalog/service/ConfigService.java
git commit -m "feat(catalog): add typed config entities (Credito, Cartao, Conta, Investimento, Seguro)"
```

---

### Task 5: Oferta Layer — Oferta + CondicaoComercial

**Files:**
- Create: `Oferta.java` entity
- Create: `CondicaoComercial.java` entity
- Create: DTOs for each
- Create: Repositories
- Create: `OfertaService.java`
- Create: `OfertaController.java`

**Interfaces:**
- Produces: `/api/catalog/ofertas` CRUD, `POST/GET/PUT /ofertas/{id}/condicoes`, `PATCH /ofertas/{id}/status`

- [ ] **Step 1: Create `Oferta.java`**

```java
package com.aurix.platform.catalog.entity;

import com.aurix.platform.catalog.enums.StatusOferta;
import com.aurix.platform.shared.entity.BaseEntity;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Table;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "catalog_ofertas", schema = "aurix")
public class Oferta extends BaseEntity {

    @Column(name = "modalidade_id", nullable = false)
    private Long modalidadeId;

    @NotBlank
    @Column(name = "nome", nullable = false, length = 255)
    private String nome;

    @Column(name = "inicio", nullable = false)
    private LocalDate inicio;

    @Column(name = "fim")
    private LocalDate fim;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private StatusOferta status;

    @Column(name = "prioridade")
    private Integer prioridade;

    @Column(name = "publico_alvo", length = 500)
    private String publicoAlvo;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    // getters/setters
}
```

- [ ] **Step 2: Create `CondicaoComercial.java`** (from spec — all BigDecimal fields optional, FK to ofertaId)

- [ ] **Step 3: Create DTOs + Repositories**

- [ ] **Step 4: Create `OfertaService.java`** with:
  - CRUD for Oferta
  - Status transitions: `ativar`, `pausar`, `encerrar`
  - CondicaoComercial create/update

- [ ] **Step 5: Create `OfertaController.java`** with spec endpoints

- [ ] **Step 6: Compile and commit**

```bash
git add ... && git commit -m "feat(catalog): add Oferta + CondicaoComercial layer"
```

---

### Task 6: Elegibilidade Layer — PoliticaElegibilidade + RegraElegibilidade + OfertaPolitica

**Files:**
- Create: `PoliticaElegibilidade.java` entity
- Create: `RegraElegibilidade.java` entity
- Create: `OfertaPoliticaElegibilidade.java` association entity
- Create: DTOs
- Create: Repositories
- Create: `ElegibilidadeService.java` (policy CRUD + evaluation engine)
- Create: `ElegibilidadeController.java`

**Interfaces:**
- Produces: `/api/catalog/elegibilidade/politicas` CRUD, `/api/catalog/elegibilidade/avaliar`
- Produces: `POST/DELETE /api/catalog/ofertas/{id}/politicas` (via OfertaController extension)

- [ ] **Step 1: Create `PoliticaElegibilidade.java`**

```java
package com.aurix.platform.catalog.entity;

import com.aurix.platform.catalog.enums.StatusPolitica;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.time.LocalDateTime;

@Entity
@Table(name = "catalog_politicas_elegibilidade", schema = "aurix")
public class PoliticaElegibilidade {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank
    @Column(name = "codigo", nullable = false, unique = true, length = 50)
    private String codigo;

    @NotBlank
    @Column(name = "nome", nullable = false, length = 255)
    private String nome;

    @Column(name = "descricao", length = 1000)
    private String descricao;

    @Column(name = "versao", nullable = false)
    private Integer versao = 1;

    @NotNull
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private StatusPolitica status;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    // getters/setters
}
```

- [ ] **Step 2: Create `RegraElegibilidade.java`**

```java
package com.aurix.platform.catalog.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import java.time.LocalDateTime;

@Entity
@Table(name = "catalog_regras_elegibilidade", schema = "aurix")
public class RegraElegibilidade {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "politica_id", nullable = false)
    private Long politicaId;

    @Column(name = "campo", nullable = false, length = 50)
    private String campo;

    @Column(name = "operador", nullable = false, length = 20)
    private String operador;

    @Column(name = "valor", nullable = false, length = 500)
    private String valor;

    @Column(name = "ordem", nullable = false)
    private Integer ordem;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // getters/setters
}
```

- [ ] **Step 3: Create `OfertaPoliticaElegibilidade.java`**

```java
package com.aurix.platform.catalog.entity;

import com.aurix.platform.catalog.enums.TipoVinculoElegibilidade;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import jakarta.persistence.UniqueConstraint;
import java.time.LocalDateTime;

@Entity
@Table(name = "catalog_ofertas_politicas", schema = "aurix",
       uniqueConstraints = @UniqueConstraint(columnNames = {"oferta_id", "politica_id"}))
public class OfertaPoliticaElegibilidade {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "oferta_id", nullable = false)
    private Long ofertaId;

    @Column(name = "politica_id", nullable = false)
    private Long politicaId;

    @Enumerated(EnumType.STRING)
    @Column(name = "tipo", nullable = false, length = 20)
    private TipoVinculoElegibilidade tipo;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    // getters/setters
}
```

- [ ] **Step 4: Create `ElegibilidadeService.java`**

```java
package com.aurix.platform.catalog.service;

import com.aurix.platform.catalog.entity.OfertaPoliticaElegibilidade;
import com.aurix.platform.catalog.entity.PoliticaElegibilidade;
import com.aurix.platform.catalog.entity.RegraElegibilidade;
import com.aurix.platform.catalog.enums.TipoVinculoElegibilidade;
import com.aurix.platform.catalog.repository.OfertaPoliticaRepository;
import com.aurix.platform.catalog.repository.PoliticaElegibilidadeRepository;
import com.aurix.platform.catalog.repository.RegraElegibilidadeRepository;
import com.aurix.platform.shared.entity.Cliente;
import com.aurix.platform.shared.repository.ClienteRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Service
@Transactional
public class ElegibilidadeService {

    private final PoliticaElegibilidadeRepository politicaRepository;
    private final RegraElegibilidadeRepository regraRepository;
    private final OfertaPoliticaRepository ofertaPoliticaRepository;
    private final ClienteRepository clienteRepository;

    public ElegibilidadeService(PoliticaElegibilidadeRepository politicaRepository,
                                RegraElegibilidadeRepository regraRepository,
                                OfertaPoliticaRepository ofertaPoliticaRepository,
                                ClienteRepository clienteRepository) {
        this.politicaRepository = politicaRepository;
        this.regraRepository = regraRepository;
        this.ofertaPoliticaRepository = ofertaPoliticaRepository;
        this.clienteRepository = clienteRepository;
    }

    public List<PoliticaElegibilidade> listarPoliticas() {
        return politicaRepository.findByDeletedAtIsNull();
    }

    public PoliticaElegibilidade criarPolitica(PoliticaElegibilidade politica, List<RegraElegibilidade> regras) {
        politica.setCreatedAt(LocalDateTime.now());
        PoliticaElegibilidade saved = politicaRepository.save(politica);
        for (RegraElegibilidade regra : regras) {
            regra.setPoliticaId(saved.getId());
            regra.setCreatedAt(LocalDateTime.now());
            regraRepository.save(regra);
        }
        return saved;
    }

    // Evaluation: for each ofertaId, find linked policies, evaluate all rules
    public Map<Long, Boolean> avaliarOfertas(Long clienteId, List<Long> ofertaIds) {
        Cliente cliente = clienteRepository.findById(clienteId).orElse(null);
        if (cliente == null) return Map.of();

        Map<Long, Boolean> resultados = new java.util.HashMap<>();
        for (Long ofertaId : ofertaIds) {
            List<OfertaPoliticaElegibilidade> vinculos =
                ofertaPoliticaRepository.findByOfertaId(ofertaId);
            boolean elegivel = true;
            for (OfertaPoliticaElegibilidade vinculo : vinculos) {
                if (vinculo.getTipo() == TipoVinculoElegibilidade.OBRIGATORIA) {
                    List<RegraElegibilidade> regras =
                        regraRepository.findByPoliticaIdOrderByOrdem(vinculo.getPoliticaId());
                    if (!avaliarRegras(cliente, regras)) {
                        elegivel = false;
                        break;
                    }
                }
            }
            resultados.put(ofertaId, elegivel);
        }
        return resultados;
    }

    private boolean avaliarRegras(Cliente cliente, List<RegraElegibilidade> regras) {
        for (RegraElegibilidade regra : regras) {
            String valorCliente = extrairValor(cliente, regra.getCampo());
            if (valorCliente == null) return false;
            if (!aplicarOperador(valorCliente, regra.getOperador(), regra.getValor())) {
                return false;
            }
        }
        return true;
    }

    private String extrairValor(Cliente cliente, String campo) {
        return switch (campo) {
            case "IDADE" -> cliente.getDataNascimento() != null
                ? String.valueOf(java.time.Period.between(cliente.getDataNascimento(), java.time.LocalDate.now()).getYears())
                : null;
            case "SEGMENTO_CLIENTE" -> cliente.getTipoPessoa() != null
                ? cliente.getTipoPessoa().name() : null;
            case "UF" -> cliente.getEstado();
            case "RENDA_MENSAL" -> null; // would need access to financial profile
            case "SCORE" -> null; // would need access to financial profile
            default -> null;
        };
    }

    private boolean aplicarOperador(String valorCliente, String operador, String valorRegra) {
        return switch (operador) {
            case "==" -> valorCliente.equals(valorRegra);
            case "!=" -> !valorCliente.equals(valorRegra);
            case "IN" -> List.of(valorRegra.split(",")).contains(valorCliente);
            default -> {
                try {
                    double vc = Double.parseDouble(valorCliente);
                    double vr = Double.parseDouble(valorRegra);
                    yield switch (operador) {
                        case ">=" -> vc >= vr;
                        case "<=" -> vc <= vr;
                        case ">" -> vc > vr;
                        case "<" -> vc < vr;
                        default -> false;
                    };
                } catch (NumberFormatException e) {
                    yield false;
                }
            }
        };
    }
}
```

- [ ] **Step 5: Create `ElegibilidadeController.java`**

```java
@RestController
@RequestMapping("/catalog/elegibilidade")
public class ElegibilidadeController {

    private final ElegibilidadeService elegibilidadeService;

    @GetMapping("/politicas")
    public ResponseEntity<List<PoliticaElegibilidade>> listarPoliticas() { ... }

    @PostMapping("/politicas")
    public ResponseEntity<PoliticaElegibilidade> criarPolitica(
        @RequestBody CriarPoliticaRequest request) { ... }

    @PostMapping("/avaliar")
    public ResponseEntity<Map<Long, Boolean>> avaliar(
        @RequestBody AvaliarRequest request) { ... }
}
```

- [ ] **Step 6: Compile and commit**

---

### Task 7: Transversal Groupings — Campanha, Canal, SegmentoCliente

**Files:**
- Create: `Campanha.java`, `Canal.java`, `SegmentoCliente.java` entities
- Create: `CampanhaOferta.java`, `CanalOferta.java`, `OfertaSegmento.java` association entities
- Create: DTOs
- Create: Repositories
- Create: `CampanhaService.java`, `CampanhaController.java`
- Create: `CanalController.java`, `SegmentoClienteController.java`

**Interfaces:**
- Produces: `/api/catalog/campanhas`, `/api/catalog/canais`, `/api/catalog/segmentos-cliente` CRUD endpoints

- [ ] **Step 1-3: Create entities (following exact patterns from earlier tasks)**

- [ ] **Step 4-6: Create DTOs, repositories, services, controllers**

- [ ] **Step 7: Compile and commit**

---

### Task 8: Integration Tests — Taxonomy Layer

**Files:**
- Create: `test/.../integration/CatalogoIntegrationTest.java`

**Interfaces:**
- Verifies: full CRUD for Segmento→CategoriaProduto, tree endpoint

- [ ] **Step 1: Create `CatalogoIntegrationTest.java`**

```java
package com.aurix.platform.catalog.integration;

import com.aurix.platform.catalog.AurixCatalogApplication;
import com.aurix.platform.catalog.config.TestSecurityConfig;
import com.aurix.platform.catalog.entity.Segmento;
import com.aurix.platform.catalog.enums.StatusCatalogo;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.web.client.RestTemplate;

import java.time.LocalDate;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
                classes = {AurixCatalogApplication.class, TestSecurityConfig.class})
@ActiveProfiles("test")
class CatalogoIntegrationTest {

    @LocalServerPort
    private int port;

    private RestTemplate restTemplate;
    private String baseUrl;

    @BeforeEach
    void setUp() {
        restTemplate = new RestTemplate();
        restTemplate.setErrorHandler(new org.springframework.web.client.ResponseErrorHandler() {
            @Override
            public boolean hasError(org.springframework.http.client.ClientHttpResponse response) {
                return false;
            }
            @Override
            public void handleError(org.springframework.http.client.ClientHttpResponse response) {
            }
        });
        baseUrl = "http://localhost:" + port + "/api/catalog";
    }

    @Test
    void deveCriarEBuscarSegmento() {
        Segmento segmento = new Segmento();
        segmento.setCodigo("PF");
        segmento.setNome("Pessoa Física");
        segmento.setDescricao("Produtos para pessoas físicas");
        segmento.setOrdem(1);
        segmento.setStatus(StatusCatalogo.ATIVO);
        segmento.setVigenciaInicio(LocalDate.now());

        ResponseEntity<Segmento> response = restTemplate.postForEntity(
            baseUrl + "/segmentos", segmento, Segmento.class);

        assertEquals(HttpStatus.CREATED, response.getStatusCode());
        assertNotNull(response.getBody().getId());
        assertEquals("PF", response.getBody().getCodigo());

        ResponseEntity<Segmento> busca = restTemplate.getForEntity(
            baseUrl + "/segmentos/" + response.getBody().getId(), Segmento.class);
        assertEquals(HttpStatus.OK, busca.getStatusCode());
    }

    @Test
    void deveListarTodosSegmentos() {
        ResponseEntity<List> response = restTemplate.getForEntity(
            baseUrl + "/segmentos", List.class);
        assertEquals(HttpStatus.OK, response.getStatusCode());
    }

    @Test
    void deveCriarArvoreCompleta() {
        // Create segmento → linha → familia → categoria
        Segmento seg = criarSegmento("PJ", "Pessoa Jurídica");
        Long linhaId = criarLinhaNegocio(seg.getId(), "CREDITO", "Crédito");
        Long famId = criarFamilia(linhaId, "CAPITAL_GIRO", "Capital de Giro");
        Long catId = criarCategoria(famId, "GIRO_GARANTIDO", "Giro Garantido");

        ResponseEntity<List> arvore = restTemplate.getForEntity(
            baseUrl + "/arvore", List.class);
        assertEquals(HttpStatus.OK, arvore.getStatusCode());
        assertFalse(((List) arvore.getBody()).isEmpty());
    }

    // Helper methods for creating tree nodes via API
    private Segmento criarSegmento(String codigo, String nome) { ... }
    private Long criarLinhaNegocio(Long segmentoId, String codigo, String nome) { ... }
    private Long criarFamilia(Long linhaId, String codigo, String nome) { ... }
    private Long criarCategoria(Long familiaId, String codigo, String nome) { ... }
}
```

- [ ] **Step 2: Run test**

`mvn test -pl aurix-catalog -am -Dtest=CatalogoIntegrationTest -DfailIfNoTests=false`
Expected: all tests pass

- [ ] **Step 3: Commit**

---

### Task 9: Integration Tests — Produto + Oferta Layers

**Files:**
- Create: `test/.../integration/ProdutoIntegrationTest.java`
- Create: `test/.../integration/OfertaIntegrationTest.java`

**Interfaces:**
- Verifies: Produto CRUD, Modalidade CRUD, Oferta CRUD + status transitions

- [ ] **Step 1: Create `ProdutoIntegrationTest.java`**

Tests: criar produto with tipoProduto, listar com filtro por tipo, criar modalidade, criar integracao, criar config credito.

- [ ] **Step 2: Create `OfertaIntegrationTest.java`**

Tests: criar oferta, vincular condicao comercial, transicionar status (RASCUNHO→ATIVA→PAUSADA→ENCERRADA), criar politica de elegibilidade, vincular politica à oferta.

- [ ] **Step 3: Run tests**

`mvn test -pl aurix-catalog -am -Dtest=ProdutoIntegrationTest,OfertaIntegrationTest -DfailIfNoTests=false`
Expected: all pass

- [ ] **Step 4: Commit**

---

### Task 10: Integration Tests — Elegibilidade + Campanha + Canal + Segmento

**Files:**
- Create: `test/.../integration/ElegibilidadeIntegrationTest.java`
- Create: `test/.../integration/CampanhaIntegrationTest.java`

**Interfaces:**

- [ ] **Step 1: Create `ElegibilidadeIntegrationTest.java`**

Tests: criar politica com regras, avaliar oferta para cliente PF, avaliar oferta que falha elegibilidade.

- [ ] **Step 2: Create `CampanhaIntegrationTest.java`**

Tests: criar campanha, vincular ofertas a campanha, criar canal, vincular oferta a canal, criar segmentoCliente, vincular oferta a segmento.

- [ ] **Step 3: Run full test suite**

`mvn test -pl aurix-catalog -am -DfailIfNoTests=false`
Expected: all pass

- [ ] **Step 4: Commit**

---

### Task 11: Regression

**Files:** none (run only)

- [ ] **Step 1: Run full compilation**

`mvn compile -pl aurix-catalog -am`
Expected: BUILD SUCCESS

- [ ] **Step 2: Run all catalog tests**

`mvn test -pl aurix-catalog -am -DfailIfNoTests=false`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 3: Verify no impact on other modules**

`mvn compile`
Expected: BUILD SUCCESS (new module is independent, no other module depends on it)

- [ ] **Step 4: Final commit (if any fixes needed)**

```bash
git add -A && git commit -m "test(catalog): add integration tests + regression"
```
