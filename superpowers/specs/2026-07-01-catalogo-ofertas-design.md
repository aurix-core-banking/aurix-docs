# Catálogo de Ofertas — Motor de Ofertas Multi-Público

## Goal

Create a new `aureus-catalog` module implementing an **offer engine** (motor de ofertas) that supports product catalog navigation, product registration, modalidades, commercial offers, eligibility policies, campaigns, channels, and client segments — serving PF, PJ, institutions, and other audiences with a single unified architecture.

## Architecture Overview

The system separates five independent concerns:

```
Catálogo (Taxonomia — o que o cliente vê e como navega)
    ↓
Produto (Identidade do produto financeiro — o que o banco oferece)
    ↓
Oferta (Condições comerciais vigentes — como o produto é vendido)
    ↓
Elegibilidade (Quem pode contratar — regras reutilizáveis)
    ↓
Campanha / Canal / Segmento (Agrupadores transversais)
```

Each layer has its own lifecycle, ownership, and rate of change.

---

## 1. Catálogo — Taxonomy Layer

Pure navigation. No business rules, no state, no behavior. Entities change rarely (years).

### Segmento

```
id: Long (PK)
codigo: String (unique)
nome: String
descricao: String
icone: String (optional)
ordem: Integer
status: StatusCatalogo [ATIVO, INATIVO]
versao: Integer
vigenciaInicio: LocalDate
vigenciaFim: LocalDate (nullable)
createdAt: LocalDateTime
updatedAt: LocalDateTime
deletedAt: LocalDateTime (nullable)
```

### LinhaNegocio

```
id: Long (PK)
segmentoId: Long (FK → Segmento.id)
codigo: String (unique within segmento)
nome: String
descricao: String
icone: String (optional)
ordem: Integer
status: StatusCatalogo
versao: Integer
vigenciaInicio: LocalDate
vigenciaFim: LocalDate (nullable)
createdAt, updatedAt, deletedAt
```

### FamiliaProduto

```
id: Long (PK)
linhaNegocioId: Long (FK → LinhaNegocio.id)
codigo: String (unique within linhaNegocio)
nome: String
descricao: String
icone: String (optional)
ordem: Integer
status: StatusCatalogo
versao: Integer
vigenciaInicio, vigenciaFim
createdAt, updatedAt, deletedAt
```

### Categoria

```
id: Long (PK)
familiaId: Long (FK → FamiliaProduto.id)
codigo: String (unique within familia)
nome: String
descricao: String
icone: String (optional)
ordem: Integer
status: StatusCatalogo
versao: Integer
vigenciaInicio, vigenciaFim
createdAt, updatedAt, deletedAt
```

### CategoriaProduto (N:N association)

```
id: Long (PK)
categoriaId: Long (FK → Categoria.id)
produtoId: Long (FK → Produto.id)
ordem: Integer
destaque: boolean
ativo: boolean
createdAt, updatedAt
```

**Unique constraint:** `(categoriaId, produtoId)`

### StatusCatalogo enum

```
ATIVO, INATIVO
```

---

## 2. Produto — Product Identity Layer

The product itself. Agnostic — no commercial rules. Just identity + capabilities.

### TipoProduto enum

```
CREDITO, INVESTIMENTO, SEGURO, CARTAO, CONTA, CAMBIO, PAGAMENTO, BENEFICIO, CONSORCIO, PREVIDENCIA, SERVICO
```

### Produto

```
id: Long (PK)
codigo: String (unique)
nome: String
tipoProduto: TipoProduto
descricao: String
versao: Integer
status: StatusProduto [RASCUNHO, ATIVO, INATIVO, OBSOLETO]
vigenciaInicio: LocalDate
vigenciaFim: LocalDate (nullable)
createdAt, updatedAt, deletedAt
```

### ProdutoIntegracao (1:N)

```
id: Long (PK)
produtoId: Long (FK → Produto.id)
sistema: SistemaExterno (enum)
codigoExterno: String
versao: String (optional)
ativo: boolean
createdAt, updatedAt
```

### SistemaExterno enum

```
CORE_BANKING, CRM, OPEN_FINANCE, ERP, MOTOR_CREDITO, MOTOR_ANTIFRAUDE, SAP, SALESFORCE, LEGADO
```

### Modalidade

Functional/technical variation of a product.

```
id: Long (PK)
produtoId: Long (FK → Produto.id)
codigo: String (unique within produto)
nome: String
tipoModalidade: String (e.g., "VISA", "MASTER", "NOVO", "USADO")
descricao: String
versao: Integer
status: StatusProduto
vigenciaInicio: LocalDate
vigenciaFim: LocalDate (nullable)
createdAt, updatedAt, deletedAt
```

### VersaoModalidade (optional, for audited rule history)

```
id: Long (PK)
modalidadeId: Long (FK → Modalidade.id)
versao: Integer
dataInicio: LocalDate
dataFim: LocalDate (nullable)
alteracoes: String (description of changes, max 500 chars)
createdAt
```

### Configuração Especializada (Table Per Type)

Each modalidade may have one config row, typed by TipoProduto.

#### ConfigCredito

```
id: Long (PK)
modalidadeId: Long (FK → Modalidade.id, unique)
idadeMinima: Integer
idadeMaxima: Integer
prazoMinimoMeses: Integer
prazoMaximoMeses: Integer
valorMinimo: BigDecimal
valorMaximo: BigDecimal
exigeGarantia: boolean
taxaJurosBase: BigDecimal
createdAt, updatedAt
```

#### ConfigCartao

```
id: Long (PK)
modalidadeId: Long (FK → Modalidade.id, unique)
bandeiras: String (JSON array: ["VISA","MASTER"])
anuidadeBase: BigDecimal
programaPontos: String (optional)
limiteMinimo: BigDecimal
limiteMaximo: BigDecimal
createdAt, updatedAt
```

#### ConfigConta

```
id: Long (PK)
modalidadeId: Long (FK → Modalidade.id, unique)
tipoConta: String (CORRENTE, POUPANCA, DIGITAL, SALARIO, UNIVERSITARIA, PREMIUM, INTERNACIONAL, MEI, EMPRESARIAL, CONDOMINIO)
tarifaMensal: BigDecimal
aceitaVariosPorCliente: boolean
createdAt, updatedAt
```

#### ConfigInvestimento

```
id: Long (PK)
modalidadeId: Long (FK → Modalidade.id, unique)
tipoRenda: TipoRenda [FIXA, VARIAVEL, HIBRIDA]
aplicacaoMinima: BigDecimal
carenciaDias: Integer
vencimentoPadraoDias: Integer
createdAt, updatedAt
```

#### ConfigSeguro

```
id: Long (PK)
modalidadeId: Long (FK → Modalidade.id, unique)
coberturasBase: String (JSONB)
premioMinimo: BigDecimal
createdAt, updatedAt
```

Additional config entities (ConfigCambio, ConfigConsorcio, etc.) follow the same pattern and can be added later.

---

## 3. Oferta — Commercial Offer Layer

The most dynamic layer. An offer = a modalidade + commercial conditions + eligibility.

### Oferta

```
id: Long (PK)
modalidadeId: Long (FK → Modalidade.id)
nome: String (e.g., "Taxa 0,99%", "Black Friday Veículos")
inicio: LocalDate
fim: LocalDate (nullable)
status: StatusOferta [RASCUNHO, ATIVA, PAUSADA, ENCERRADA]
prioridade: Integer (lower = higher priority)
publicoAlvo: String (free text, optional — see SegmentoCliente for structured target)
versao: Integer
createdAt, updatedAt, deletedAt
```

### CondicaoComercial

```
id: Long (PK)
ofertaId: Long (FK → Oferta.id)
taxaJurosAnual: BigDecimal (optional)
taxaJurosMensal: BigDecimal (optional)
carenciaDias: Integer (optional)
entradaPercentual: BigDecimal (optional)
prazoMaximoMeses: Integer (optional)
valorMinimo: BigDecimal (optional)
valorMaximo: BigDecimal (optional)
parcelamentoMaximo: Integer (optional)
iof: BigDecimal (optional)
cet: BigDecimal (optional)  — Custo Efetivo Total
tac: BigDecimal (optional)  — Taxa de Abertura de Crédito
cashbackPercentual: BigDecimal (optional)
cashbackValor: BigDecimal (optional)
descontoPercentual: BigDecimal (optional)
tarifaMensal: BigDecimal (optional)
bonus: BigDecimal (optional)
createdAt, updatedAt
```

---

## 4. Elegibilidade — Eligibility Policy Layer

Reusable policies shared across offers.

### PoliticaElegibilidade

```
id: Long (PK)
codigo: String (unique)
nome: String (e.g., "PF Alta Renda", "Servidor Público")
descricao: String
versao: Integer
status: StatusPolitica [ATIVA, INATIVA]
createdAt, updatedAt, deletedAt
```

### RegraElegibilidade

```
id: Long (PK)
politicaId: Long (FK → PoliticaElegibilidade.id)
campo: String (e.g., "IDADE", "RENDA_MENSAL", "SCORE", "UF", "SEGMENTO_CLIENTE", "TEMPO_RELACIONAMENTO_MESES")
operador: String (e.g., ">=", "<=", "==", "!=", "IN", "BETWEEN", "EXISTS")
valor: String (e.g., "18", "15000", "700", "SP,PR,SC")
ordem: Integer (evaluation order)
createdAt, updatedAt
```

### OfertaPoliticaElegibilidade (N:N)

```
id: Long (PK)
ofertaId: Long (FK → Oferta.id)
politicaId: Long (FK → PoliticaElegibilidade.id)
tipo: TipoVinculoElegibilidade [OBRIGATORIA, OPCIONAL]
createdAt
```

**Unique constraint:** `(ofertaId, politicaId)`

---

## 5. Agrupadores Transversais

### Campanha

```
id: Long (PK)
codigo: String (unique)
nome: String
descricao: String
inicio: LocalDate
fim: LocalDate (nullable)
status: StatusCampanha [RASCUNHO, ATIVA, ENCERRADA]
createdAt, updatedAt, deletedAt
```

### CampanhaOferta (N:N)

```
id: Long (PK)
campanhaId: Long (FK → Campanha.id)
ofertaId: Long (FK → Oferta.id)
createdAt
```

**Unique:** `(campanhaId, ofertaId)`

### Canal

```
id: Long (PK)
codigo: String (unique)
nome: String (e.g., "Internet Banking", "Mobile", "Agência", "Correspondente", "Open Finance", "API", "Marketplace")
createdAt, updatedAt
```

### CanalOferta (N:N)

```
id: Long (PK)
canalId: Long (FK → Canal.id)
ofertaId: Long (FK → Oferta.id)
createdAt
```

**Unique:** `(canalId, ofertaId)`

### SegmentoCliente

```
id: Long (PK)
codigo: String (unique)
nome: String (e.g., "PF", "PJ", "MEI", "Alta Renda", "Private", "Agro", "Servidor Público", "Instituição Financeira")
createdAt, updatedAt
```

### OfertaSegmento (N:N)

```
id: Long (PK)
ofertaId: Long (FK → Oferta.id)
segmentoClienteId: Long (FK → SegmentoCliente.id)
createdAt
```

**Unique:** `(ofertaId, segmentoClienteId)`

---

## 6. Module Structure

```
aureus-catalog/
├── pom.xml
└── src/main/java/com/aureus/platform/catalog/
    ├── AureusCatalogApplication.java
    ├── entity/
    │   ├── Segmento.java
    │   ├── LinhaNegocio.java
    │   ├── FamiliaProduto.java
    │   ├── Categoria.java
    │   ├── CategoriaProduto.java
    │   ├── Produto.java
    │   ├── ProdutoIntegracao.java
    │   ├── Modalidade.java
    │   ├── VersaoModalidade.java
    │   ├── config/
    │   │   ├── ConfigCredito.java
    │   │   ├── ConfigCartao.java
    │   │   ├── ConfigConta.java
    │   │   ├── ConfigInvestimento.java
    │   │   └── ConfigSeguro.java
    │   ├── Oferta.java
    │   ├── CondicaoComercial.java
    │   ├── PoliticaElegibilidade.java
    │   ├── RegraElegibilidade.java
    │   ├── OfertaPoliticaElegibilidade.java
    │   ├── Campanha.java
    │   ├── CampanhaOferta.java
    │   ├── Canal.java
    │   ├── CanalOferta.java
    │   ├── SegmentoCliente.java
    │   └── OfertaSegmento.java
    ├── enums/
    │   ├── StatusCatalogo.java
    │   ├── StatusProduto.java
    │   ├── StatusOferta.java
    │   ├── StatusPolitica.java
    │   ├── StatusCampanha.java
    │   ├── TipoProduto.java
    │   ├── SistemaExterno.java
    │   └── TipoVinculoElegibilidade.java
    ├── dto/  (mirrors each entity)
    ├── repository/ (Spring Data JPA per entity)
    ├── service/
    │   ├── CatalogoService.java (taxonomy CRUD + tree queries)
    │   ├── ProdutoService.java (product + modalidade + config CRUD)
    │   ├── OfertaService.java (offer + conditions CRUD, activation)
    │   ├── ElegibilidadeService.java (policy evaluation engine)
    │   ├── CampanhaService.java
    │   └── CatalogoQueryService.java (read-optimized: search, filter, tree)
    ├── controller/
    │   ├── CatalogoController.java (tree navigation endpoints)
    │   ├── ProdutoController.java
    │   ├── OfertaController.java
    │   ├── CampanhaController.java
    │   ├── CanalController.java
    │   └── SegmentoClienteController.java
    └── exception/
```

### Dependencies

`aureus-catalog` depends only on `aureus-shared` (for `BaseEntity`, shared enums, utilities). No dependency on `aureus-core`, `aureus-credit`, or any domain-specific module.

---

## 7. API Endpoints

### Catálogo Navigation

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/catalog/segmentos` | List all segmentos |
| GET | `/api/catalog/segmentos/{id}` | Get segmento + linhas |
| GET | `/api/catalog/linhas-negocio/{id}` | Get linha + familias |
| GET | `/api/catalog/familias/{id}` | Get familia + categorias |
| GET | `/api/catalog/categorias/{id}` | Get categoria + produtos |
| GET | `/api/catalog/arvore` | Full tree (segmento→linha→familia→categoria) |
| POST | `/api/catalog/segmentos` | Create segmento |
| PUT | `/api/catalog/segmentos/{id}` | Update segmento |
| DELETE | `/api/catalog/segmentos/{id}` | Delete segmento (soft) |
| POST | `/api/catalog/linhas-negocio` | Create linha |
| ... | ... | (CRUD for Familia, Categoria, CategoriaProduto) |

### Produto

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/catalog/produtos` | List produtos (filters: tipo, status) |
| GET | `/api/catalog/produtos/{id}` | Get produto + modalidades + configs + integracoes |
| POST | `/api/catalog/produtos` | Create produto |
| PUT | `/api/catalog/produtos/{id}` | Update produto |
| DELETE | `/api/catalog/produtos/{id}` | Soft delete |
| POST | `/api/catalog/produtos/{id}/integracoes` | Add integracao |
| DELETE | `/api/catalog/produtos/{id}/integracoes/{intId}` | Remove integracao |
| POST | `/api/catalog/produtos/{id}/modalidades` | Create modalidade |
| PUT | `/api/catalog/modalidades/{id}` | Update modalidade |
| GET | `/api/catalog/modalidades/{id}/versoes` | List modalidade versions |
| POST | `/api/catalog/modalidades/{id}/config` | Create/update typed config |

### Oferta

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/catalog/ofertas` | List ofertas (filters: status, modalidade, canal, segmento) |
| GET | `/api/catalog/ofertas/{id}` | Get oferta + condicoes + politicas |
| POST | `/api/catalog/ofertas` | Create oferta |
| PUT | `/api/catalog/ofertas/{id}` | Update oferta |
| PATCH | `/api/catalog/ofertas/{id}/status` | Change status (ATIVAR, PAUSAR, ENCERRAR) |
| DELETE | `/api/catalog/ofertas/{id}` | Soft delete |
| POST | `/api/catalog/ofertas/{id}/condicoes` | Set commercial conditions |
| PUT | `/api/catalog/ofertas/{id}/condicoes` | Update conditions |
| POST | `/api/catalog/ofertas/{id}/politicas` | Link eligibility policy |
| DELETE | `/api/catalog/ofertas/{id}/politicas/{politicaId}` | Unlink policy |

### Campanha / Canal / SegmentoCliente

| Method | Path | Description |
|--------|------|-------------|
| GET/POST/PUT/DELETE | `/api/catalog/campanhas` | CRUD |
| POST | `/api/catalog/campanhas/{id}/ofertas` | Link offer to campaign |
| DELETE | `/api/catalog/campanhas/{id}/ofertas/{ofertaId}` | Unlink |
| GET/POST | `/api/catalog/canais` | List/create channels |
| GET/POST | `/api/catalog/segmentos-cliente` | List/create client segments |

### Eligibility Evaluation

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/catalog/elegibilidade/avaliar` | Evaluate all rules for a client (body: clienteId, ofertaIds[]) |
| GET | `/api/catalog/elegibilidade/politicas` | List policies |
| POST | `/api/catalog/elegibilidade/politicas` | Create policy with rules |
| PUT | `/api/catalog/elegibilidade/politicas/{id}` | Update policy |
| DELETE | `/api/catalog/elegibilidade/politicas/{id}` | Delete |

The maximum nesting depth is `segmento → linha → familia → categoria`, and the maximum list size per endpoint is 100 by default (paginated). All create/update operations validate required fields and unique constraints at the service layer.

---

## 8. Database Schema

Tables follow naming pattern: `catalog_<entity_name>`.

```
catalog_segmentos
catalog_linhas_negocio
catalog_familias_produto
catalog_categorias
catalog_categorias_produtos
catalog_produtos
catalog_produtos_integracao
catalog_modalidades
catalog_versoes_modalidade
catalog_config_credito
catalog_config_cartao
catalog_config_conta
catalog_config_investimento
catalog_config_seguro
catalog_ofertas
catalog_condicoes_comerciais
catalog_politicas_elegibilidade
catalog_regras_elegibilidade
catalog_ofertas_politicas
catalog_campanhas
catalog_campanhas_ofertas
catalog_canais
catalog_canais_ofertas
catalog_segmentos_cliente
catalog_ofertas_segmentos
```

---

## 9. Non-goals (MVP)

- **Campaign scheduling engine** (e.g., auto-activate/pause) — deferred
- **A/B testing of offers** — deferred
- **Offer recommendation/ranking** (ML-based) — deferred
- **Real-time eligibility evaluation via rules engine (Drools)** — MVP uses service-layer evaluation; Drools can replace later
- **Integration with existing domain products** (`ProdutoCredito`, `ProdutoCartao`, etc.) — deferred to when legacy products migrate to `aureus-catalog` as source of truth
- **Data migration** from existing product entities — out of scope
- **Frontend** — API-only; admin UI comes later

---

## 10. Implementation Plan (next: writing-plans)

1. Create `aureus-catalog` module (pom.xml, empty structure)
2. BaseEntity + enums
3. Catálogo entities (Segmento → CategoriaProduto) + repositories + services + controllers
4. Produto + Integracao + Modalidade + VersaoModalidade entities + repos + services + controllers
5. Config entities (5 types) + repos + services
6. Oferta + CondicaoComercial entities + repos + services + controllers
7. PoliticaElegibilidade + RegraElegibilidade + OfertaPolitica entities + repos + services + controllers
8. Campanha + Canal + SegmentoCliente + N:N associations + repos + services + controllers
9. Elegibilidade evaluation engine
10. API integration tests for each layer
11. Regression across affected modules
