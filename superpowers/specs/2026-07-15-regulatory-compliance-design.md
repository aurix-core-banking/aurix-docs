# Regulatory Compliance — Design Spec

> **Objetivo:** Implementar as obrigações regulatórias ausentes: COAF, Basileia III, Recolhimento Compulsório, Ouvidoria, LGPD Portal do Titular, SCR automático, e enforcement de juros regulados.
> **Prioridade:** P0 (COAF) e P1 (demais) — risco de sanção e cassação de licença bancária.
> **Módulos impactados:** `aurix-compliance`, `aurix-bacen`, `aurix-credit`, `aurix-financial`, novo `aurix-ouvidoria`.

---

## Gap 1 — COAF: Comunicação de Operações Suspeitas

### Problema

O BACEN exige que instituições financeiras comuniquem ao COAF operações suspeitas de lavagem de dinheiro (Lei 9.613/1998, Res. BACEN 4.595/2017). Nenhum módulo faz essa comunicação.

### Arquitetura

```
[aurix-financial]  ──Kafka: financial.movimentacao.criada.v1──▶ [aurix-compliance]
[aurix-pix]        ──Kafka: pix.transferencia.iniciada.v1──▶   [aurix-compliance]
[aurix-cartoes]    ──Kafka: cartoes.transacao.autorizada.v1──▶ [aurix-compliance]
                                        │
                              ComunicacaoCoafService
                                        │
                           ┌────────────┴────────────┐
                     [HTTP POST COAF API]     [Kafka: coaf.comunicacao.enviada.v1]
```

### Novas Entidades em `aurix-compliance`

**`RegrasCoaf`** (`compliance.regras_coaf`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK) | Identificador |
| codigo | String | Código único da regra |
| descricao | String | Descrição da regra |
| tipo | String | VALOR_ALTO / FREQUENCIA / FRACIONAMENTO / OPERACAO_SUSPEITA |
| parametroJson | String | Parâmetros configuráveis (JSON) |
| ativa | Boolean | Se a regra está ativa |

**`AlertaCoaf`** (`compliance.alertas_coaf`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK) | Identificador |
| clienteId | Long | FK → customer.cliente |
| regraId | Long | FK → regras_coaf |
| transacaoId | Long | ID da transação de origem |
| modulo | String | pix / financial / cartoes |
| valor | BigDecimal | Valor da operação |
| descricao | String | Descrição do alerta |
| status | String | PENDENTE / COMUNICADO / DESCARTADO |
| dataCriacao | LocalDateTime | Criação do alerta |
| dataAnalise | LocalDateTime | Data de análise |
| analistaId | Long | Analista que revisou |
| motivoDescarte | String | Justificativa caso descartado |

**`ComunicacaoCoaf`** (`compliance.comunicacoes_coaf`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK) | Identificador |
| alertaId | Long | FK → alertas_coaf |
| protocolo | String | Protocolo de comunicação COAF |
| dataEnvio | LocalDateTime | Data do envio |
| statusEnvio | String | PENDENTE / ENVIADO / REJEITADO |
| payload | String (JSON) | Payload enviado ao COAF |
| resposta | String | Resposta da API COAF |

### Serviços

**`AnaliseCoafService`**

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class AnaliseCoafService {

    // Regra 1: Operação acima de R$ 50.000 sem justificativa (Art. 11, Lei 9.613)
    public boolean regra50k(BigDecimal valor) {
        return valor.compareTo(BigDecimal.valueOf(50_000)) >= 0;
    }

    // Regra 2: Fracionamento — mesma conta, múltiplas transações < R$ 50k no mesmo dia totalizando > R$ 50k
    public boolean regraFracionamento(Long clienteId, LocalDate data, BigDecimal novoValor) {
        BigDecimal totalDia = alertaRepository.somarTransacoesDia(clienteId, data);
        return totalDia.add(novoValor).compareTo(BigDecimal.valueOf(50_000)) >= 0;
    }

    // Regra 3: Operação incompatível com perfil (renda declarada)
    public boolean regraIncompatibilidadePerfil(Long clienteId, BigDecimal valor) {
        BigDecimal rendaMensal = clienteService.getRendaMensal(clienteId);
        return rendaMensal != null && valor.compareTo(rendaMensal.multiply(BigDecimal.valueOf(12))) > 0;
    }

    // Regra 4: PEP (Pessoa Politicamente Exposta) com operação > R$ 30k
    public boolean regraClientePep(Long clienteId, BigDecimal valor) {
        boolean isPep = kycService.isClientePep(clienteId);
        return isPep && valor.compareTo(BigDecimal.valueOf(30_000)) >= 0;
    }

    public AlertaCoaf criarAlerta(Long clienteId, Long transacaoId, String modulo,
            BigDecimal valor, String regraDisparada) {
        // Persistir alerta e colocar na fila de análise
    }
}
```

**`ComunicacaoCoafService`**

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ComunicacaoCoafService {

    private final CoafApiClient coafApiClient;  // HTTP client para API COAF
    private final ComunicacaoCoafRepository repo;
    private final KafkaTemplate<String, BaseEvent> kafka;

    // Analista aprova comunicação ao COAF
    public ComunicacaoCoaf comunicar(Long alertaId) {
        AlertaCoaf alerta = alertaRepository.findById(alertaId).orElseThrow();
        CoafPayload payload = CoafPayload.from(alerta);

        try {
            String protocolo = coafApiClient.enviar(payload);
            ComunicacaoCoaf com = salvar(alertaId, protocolo, "ENVIADO", payload);
            kafka.send(Topics.COAF_COMUNICACAO_ENVIADA, String.valueOf(alertaId),
                CoafComunicadaEvent.of(com));
            alerta.setStatus("COMUNICADO");
            return com;
        } catch (Exception e) {
            log.error("Falha ao comunicar COAF: alertaId={}", alertaId, e);
            salvar(alertaId, null, "REJEITADO", payload);
            throw e;
        }
    }
}
```

### `CoafApiClient` — integração com API real

```java
@FeignClient(name = "coaf-api", url = "${coaf.api.url:https://api-coaf.bcb.gov.br}")
public interface CoafApiClient {

    @PostMapping("/comunicacoes")
    String enviar(@RequestBody CoafPayload payload,
                  @RequestHeader("Authorization") String authToken);
}
```

No perfil `dev`: usar WireMock mock stub (no mesmo padrão do `bacen-mock`).
No perfil `prod`: integração real via certificado digital ICP-Brasil.

### REST API

| Método | Path | Descrição |
|--------|------|-----------|
| GET | `/api/compliance/coaf/alertas` | Listar alertas pendentes de análise |
| GET | `/api/compliance/coaf/alertas/{id}` | Detalhe do alerta |
| POST | `/api/compliance/coaf/alertas/{id}/comunicar` | Aprovar e comunicar ao COAF |
| POST | `/api/compliance/coaf/alertas/{id}/descartar` | Descartar alerta (com motivo) |
| GET | `/api/compliance/coaf/comunicacoes` | Histórico de comunicações enviadas |
| GET | `/api/compliance/coaf/stats` | Dashboard: alertas por status, por período |

### Kafka Events

- `coaf.alerta.criado.v1` — alerta gerado por regra
- `coaf.comunicacao.enviada.v1` — comunicação enviada ao COAF
- `coaf.alerta.descartado.v1` — alerta descartado pelo analista

---

## Gap 2 — Basileia III: Capital Regulatório

### Problema

O BACEN exige que instituições financeiras calculem e mantenham o Patrimônio de Referência (PR) acima do mínimo regulatório (Res. BCB 4.955/2021). Nenhum módulo calcula isso.

### Entidades em `aurix-compliance`

**`CapitalRegulatorio`** (`compliance.capital_regulatorio`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK) | Identificador |
| dataCalculo | LocalDate | Data de referência |
| patrimonioReferencia | BigDecimal | PR (Nível I + Nível II) |
| nivelI | BigDecimal | Capital Principal + Capital Complementar |
| nivelII | BigDecimal | Capital de nível II |
| patrimonioMinimo | BigDecimal | PR mínimo exigido |
| ativosPonderadosRisco | BigDecimal | RWA total |
| indiceTCR | BigDecimal | PR / RWA × 100 (mín. 10,5%) |
| indiceCET1 | BigDecimal | Capital Principal / RWA × 100 (mín. 7%) |
| statusConformidade | String | CONFORME / NAO_CONFORME |

### Services

**`BasileiaService`**

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class BasileiaService {

    // Calcular RWA: somar exposições × fator de ponderação
    public BigDecimal calcularRWA() {
        // Crédito pessoa física: peso 100%
        // Crédito hipotecário: peso 35%
        // Títulos públicos: peso 0%
        // Renda variável: peso 300%
        return financialService.getExposicoes().stream()
            .map(e -> e.getValor().multiply(e.getFatorPonderacao()))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    // Calcular TCR: PR / RWA
    public BigDecimal calcularTCR(BigDecimal pr, BigDecimal rwa) {
        return pr.divide(rwa, 4, RoundingMode.HALF_UP)
                 .multiply(BigDecimal.valueOf(100));
    }

    // Verificar conformidade (mínimo 10,5% TCR + 2,5% colchão de conservação)
    public boolean isConforme(BigDecimal tcr) {
        return tcr.compareTo(BigDecimal.valueOf(13.0)) >= 0;
    }

    // Job diário às 06:00
    @Scheduled(cron = "0 0 6 * * *")
    public void calcularCapitalDiario() {
        BigDecimal rwa = calcularRWA();
        BigDecimal pr = calcularPR();
        BigDecimal tcr = calcularTCR(pr, rwa);
        boolean conforme = isConforme(tcr);
        salvarCapital(pr, rwa, tcr, conforme);
        if (!conforme) {
            kafka.send(Topics.COMPLIANCE_CAPITAL_NAO_CONFORME,
                CapitalNaoConformeEvent.of(tcr, pr, rwa));
        }
    }
}
```

### REST API

| Método | Path | Descrição |
|--------|------|-----------|
| GET | `/api/compliance/basileia/capital` | Capital regulatório atual |
| GET | `/api/compliance/basileia/historico` | Histórico de cálculos |
| POST | `/api/compliance/basileia/recalcular` | Forçar recálculo (admin) |
| GET | `/api/compliance/basileia/rwa` | Breakdown do RWA por tipo de exposição |

---

## Gap 3 — Recolhimento Compulsório

### Problema

Bancos comerciais devem recolher um percentual dos depósitos ao BACEN (Circ. BCB 3.916/2018). Nenhum módulo calcula nem processa o compulsório.

### Entidades em `aurix-bacen`

**`RecolhimentoCompulsorio`** (`bacen.recolhimentos`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK) | Identificador |
| dataReferencia | LocalDate | Data de referência (semana) |
| tipoDeposito | String | VISTA / PRAZO / POUPANCA |
| baseCalculo | BigDecimal | Saldo médio dos depósitos |
| aliquota | BigDecimal | Alíquota (%) definida pelo BACEN |
| valorDevido | BigDecimal | Valor a recolher |
| valorRecolhido | BigDecimal | Valor efetivamente recolhido |
| status | String | PENDENTE / RECOLHIDO / DIFERENCA |
| protocoloBacen | String | Protocolo da liquidação BACEN |

### Service

```java
@Service
@Slf4j
public class CompulsorioCService {

    private static final BigDecimal ALIQUOTA_DEPOSITO_VISTA    = BigDecimal.valueOf(0.21);  // 21%
    private static final BigDecimal ALIQUOTA_DEPOSITO_PRAZO    = BigDecimal.valueOf(0.17);  // 17%
    private static final BigDecimal ALIQUOTA_POUPANCA          = BigDecimal.valueOf(0.20);  // 20%

    @Scheduled(cron = "0 0 7 * * MON")  // toda segunda-feira
    public void calcularCompulsoriSemanal() {
        BigDecimal saldoVista   = coreService.getSaldoMedioSemanal("CORRENTE");
        BigDecimal saldoPrazo   = coreService.getSaldoMedioSemanal("CDB");
        BigDecimal saldoPoupanca = coreService.getSaldoMedioSemanal("POUPANCA");

        BigDecimal valorDevido = saldoVista.multiply(ALIQUOTA_DEPOSITO_VISTA)
            .add(saldoPrazo.multiply(ALIQUOTA_DEPOSITO_PRAZO))
            .add(saldoPoupanca.multiply(ALIQUOTA_POUPANCA));

        salvar(LocalDate.now(), valorDevido);
        kafka.send(Topics.BACEN_COMPULSORIO_CALCULADO,
            CompulsorioCalculadoEvent.of(valorDevido));
    }
}
```

---

## Gap 4 — Ouvidoria (Res. BCB 160/2021)

### Problema

O BACEN exige que toda IF tenha ouvidoria com: canal de atendimento, prazo de resposta de 10 dias úteis, relatório semestral e registro numerado de manifestações.

### Novo módulo: `aurix-ouvidoria` (Porta 8130)

#### Entidades

**`Manifestacao`** (`ouvidoria.manifestacoes`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK) | Identificador |
| numero | String (unique) | Número da manifestação (formato OUV-2026-XXXXXX) |
| clienteId | Long | FK → customer.cliente |
| canal | String | WEB / APP / TELEFONE / EMAIL / CARTA |
| tipo | String | RECLAMACAO / DENUNCIA / SUGESTAO / ELOGIO |
| assunto | String | Assunto da manifestação |
| descricao | String | Texto completo |
| status | String | RECEBIDA / EM_ANALISE / RESPONDIDA / ENCERRADA |
| dataRecebimento | LocalDateTime | Data de recebimento |
| prazoResposta | LocalDate | Prazo legal de resposta (10 dias úteis) |
| dataResposta | LocalDateTime | Data da resposta |
| resposta | String | Texto da resposta |
| ouvidorId | Long | Ouvidor responsável |
| encaminhadoBacen | Boolean | Se escalou para BACEN |

**`RelatorioOuvidoria`** (`ouvidoria.relatorios`)
| Campo | Tipo | Descrição |
|-------|------|-----------|
| id | Long (PK) | Identificador |
| periodo | String | 1S2026, 2S2026 (semestre) |
| totalManifestacoes | Integer | Total no período |
| totalReclamacoes | Integer | Reclamações |
| totalDenuncias | Integer | Denúncias |
| tempoMedioRespostaDias | BigDecimal | Tempo médio de resposta |
| percentualRespondidas | BigDecimal | % respondidas no prazo |
| dataEnvioBacen | LocalDate | Data de envio ao BACEN |
| status | String | RASCUNHO / ENVIADO |

#### REST API

| Método | Path | Descrição |
|--------|------|-----------|
| POST | `/api/ouvidoria/manifestacoes` | Registrar manifestação |
| GET | `/api/ouvidoria/manifestacoes/{numero}` | Consultar por número |
| GET | `/api/ouvidoria/manifestacoes` | Listar (filtro: status, tipo, período) |
| PATCH | `/api/ouvidoria/manifestacoes/{id}/responder` | Registrar resposta |
| PATCH | `/api/ouvidoria/manifestacoes/{id}/encaminhar-bacen` | Escalar para BACEN |
| GET | `/api/ouvidoria/relatorios` | Listar relatórios semestrais |
| POST | `/api/ouvidoria/relatorios/gerar` | Gerar relatório do período |
| POST | `/api/ouvidoria/relatorios/{id}/enviar-bacen` | Enviar ao BACEN |
| GET | `/api/ouvidoria/health` | Health check |

#### Job de alertas de prazo

```java
@Scheduled(cron = "0 0 8 * * *")  // diariamente às 08:00
public void alertarPrazoVencendo() {
    LocalDate amanha = LocalDate.now().plusDays(1);
    List<Manifestacao> vencendo = repo.findByPrazoRespostaAndStatus(amanha, "EM_ANALISE");
    vencendo.forEach(m -> {
        log.warn("Manifestação {} vence amanhã!", m.getNumero());
        kafka.send(Topics.OUVIDORIA_PRAZO_VENCENDO,
            OuvidoriaPrazoVencendoEvent.of(m.getId(), m.getNumero(), m.getOuvidorId()));
    });
}
```

#### Infra

- Porta: **8130**
- Docker Compose: adicionar `aurix-ouvidoria`
- Traefik: rota `/api/ouvidoria` → `http://aurix-ouvidoria:8130`
- Maven parent: adicionar `<module>aurix-ouvidoria</module>`

---

## Gap 5 — LGPD: Portal do Titular (Art. 18)

### Problema

`aurix-compliance` tem entidades `ConsentimentoLGPD` e `DireitoEsquecimento`, mas não há: portal para o titular exercer direitos, workflow com prazos, nem endpoints expostos.

### Direitos do Titular (Art. 18 LGPD)

| Direito | Prazo Legal |
|---------|------------|
| Confirmação e acesso | Imediato |
| Correção de dados | 15 dias |
| Anonimização ou exclusão | 15 dias |
| Portabilidade | 15 dias |
| Eliminação de dados tratados com consentimento | 15 dias |
| Revogação de consentimento | Imediato |

### Novos Endpoints em `aurix-compliance`

```java
// Portal do Titular
POST /api/compliance/lgpd/solicitacoes              // Titular abre solicitação
GET  /api/compliance/lgpd/solicitacoes/{id}         // Status da solicitação
GET  /api/compliance/lgpd/solicitacoes/titular/{cpf} // Minhas solicitações
POST /api/compliance/lgpd/solicitacoes/{id}/executar  // DPO executa ação
GET  /api/compliance/lgpd/dados/{clienteId}          // Relatório de todos os dados do titular
POST /api/compliance/lgpd/consentimentos             // Registrar consentimento
DELETE /api/compliance/lgpd/consentimentos/{id}      // Revogar consentimento
GET  /api/compliance/lgpd/consentimentos/{clienteId} // Consentimentos ativos
```

### Entidade `SolicitacaoLgpd`

```java
@Entity
@Table(name = "compliance.solicitacoes_lgpd")
public class SolicitacaoLgpd {
    @Id @GeneratedValue
    private Long id;
    private Long clienteId;
    private String tipo;            // ACESSO / CORRECAO / EXCLUSAO / PORTABILIDADE / REVOGACAO
    private String status;          // PENDENTE / EM_ANALISE / EXECUTADA / RECUSADA
    private LocalDateTime criada;
    private LocalDate prazo;        // criada + 15 dias úteis
    private String descricao;
    private String resposta;
    private Long dpoId;             // DPO responsável
    private LocalDateTime executada;
}
```

### Job de controle de prazo

```java
@Scheduled(cron = "0 0 8 * * *")
public void alertarPrazosLgpd() {
    LocalDate amanha = LocalDate.now().plusDays(1);
    repo.findByPrazoAndStatusIn(amanha, List.of("PENDENTE", "EM_ANALISE"))
        .forEach(s -> kafka.send(Topics.LGPD_PRAZO_VENCENDO,
            LgpdPrazoVencendoEvent.of(s.getId(), s.getTipo(), s.getClienteId())));
}
```

---

## Gap 6 — SCR: Sincronização Automática

### Problema

`aurix-bacen` gera relatório SCR, mas é exportação manual. O BACEN exige que as IFs alimentem o SCR mensalmente com posições de crédito.

### Service em `aurix-bacen`

**`ScrService`** — atualizar para sincronização automática:

```java
@Service
@Slf4j
public class ScrService {

    @Scheduled(cron = "0 0 6 5 * *")  // todo dia 5 de cada mês às 06:00
    public void sincronizarScrMensal() {
        log.info("Iniciando sincronização SCR mensal");
        LocalDate competencia = LocalDate.now().withDayOfMonth(1).minusMonths(1);

        List<PosicaoCredito> posicoes = creditService.getPosicoesPorCompetencia(competencia);
        String xmlScr = scrXmlBuilder.build(posicoes, competencia);

        try {
            String protocolo = bacenApiClient.enviarScr(xmlScr, competencia);
            salvarEnvio(competencia, protocolo, "ENVIADO", posicoes.size());
            log.info("SCR enviado com sucesso: protocolo={}", protocolo);
        } catch (Exception e) {
            log.error("Erro no envio SCR", e);
            salvarEnvio(competencia, null, "FALHOU", posicoes.size());
            kafka.send(Topics.BACEN_SCR_FALHOU, ScrFalhouEvent.of(competencia, e.getMessage()));
        }
    }
}
```

---

## Gap 7 — Enforcement de Juros Regulados (CMN)

### Problema

`aurix-credit`, `aurix-consignado` e `aurix-financiamento` não validam os limites de taxa de juros regulados pelo CMN. É possível criar produtos com taxas ilegais.

### Service de validação em `aurix-shared`

**`JurosReguladosValidator`** — componente compartilhado:

```java
@Component
public class JurosReguladosValidator {

    // Consignado INSS: limite por portaria MPS
    public static final BigDecimal LIMITE_CONSIGNADO_INSS_MENSAL = BigDecimal.valueOf(0.0199); // 1,99%

    // Consignado SIAPE: limite por portaria ME
    public static final BigDecimal LIMITE_CONSIGNADO_SIAPE_MENSAL = BigDecimal.valueOf(0.0210); // 2,10%

    // Cartão consignado
    public static final BigDecimal LIMITE_CARTAO_CONSIGNADO_MENSAL = BigDecimal.valueOf(0.0299); // 2,99%

    // Cheque especial: limite Res. CMN 4.765/2019
    public static final BigDecimal LIMITE_CHEQUE_ESPECIAL_MENSAL = BigDecimal.valueOf(0.0799); // 7,99%

    public void validarTaxaConsignado(BigDecimal taxaMensal, String tipoConvenio) {
        BigDecimal limite = switch (tipoConvenio) {
            case "INSS"  -> LIMITE_CONSIGNADO_INSS_MENSAL;
            case "SIAPE" -> LIMITE_CONSIGNADO_SIAPE_MENSAL;
            default -> BigDecimal.valueOf(0.0299);
        };
        if (taxaMensal.compareTo(limite) > 0) {
            throw new TaxaIlegalException(
                String.format("Taxa %.2f%% acima do limite legal de %.2f%% para %s",
                    taxaMensal.multiply(BigDecimal.valueOf(100)),
                    limite.multiply(BigDecimal.valueOf(100)),
                    tipoConvenio));
        }
    }

    public void validarTaxaFinanciamento(BigDecimal taxaMensal, String tipoFinanciamento) {
        // Financiamento imobiliário (SFH): limitado à TR + 12% a.a. (Res. CMN 3.347/2006)
        if ("IMOBILIARIO_SFH".equals(tipoFinanciamento)) {
            BigDecimal limiteAnual = BigDecimal.valueOf(0.12);
            BigDecimal taxaAnual = taxaMensal.multiply(BigDecimal.valueOf(12));
            if (taxaAnual.compareTo(limiteAnual) > 0) {
                throw new TaxaIlegalException("Taxa SFH excede limite legal de 12% a.a.");
            }
        }
    }
}
```

Injetar `JurosReguladosValidator` em:
- `ConsignadoService.criarProposta()`
- `FinanciamentoService.criarProposta()`
- `CreditService.criarProduto()`

---

## Infra — Novos Tópicos Kafka

Adicionar em `Topics.java` (`aurix-shared`):

```java
// ===== COAF =====
String COAF_ALERTA_CRIADO           = "coaf.alerta.criado.v1";
String COAF_COMUNICACAO_ENVIADA     = "coaf.comunicacao.enviada.v1";
String COAF_ALERTA_DESCARTADO       = "coaf.alerta.descartado.v1";

// ===== Compliance =====
String COMPLIANCE_CAPITAL_NAO_CONFORME = "compliance.capital.nao-conforme.v1";
String COMPLIANCE_COMPULSORIO_CALCULADO = "bacen.compulsorio.calculado.v1";
String BACEN_SCR_FALHOU             = "bacen.scr.falhou.v1";

// ===== Ouvidoria =====
String OUVIDORIA_MANIFESTACAO_CRIADA = "ouvidoria.manifestacao.criada.v1";
String OUVIDORIA_PRAZO_VENCENDO     = "ouvidoria.prazo.vencendo.v1";
String OUVIDORIA_RESPONDIDA         = "ouvidoria.manifestacao.respondida.v1";

// ===== LGPD =====
String LGPD_SOLICITACAO_CRIADA      = "lgpd.solicitacao.criada.v1";
String LGPD_PRAZO_VENCENDO          = "lgpd.prazo.vencendo.v1";
String LGPD_EXECUTADA               = "lgpd.solicitacao.executada.v1";
```

## Resumo de Arquivos a Criar/Modificar

| Arquivo | Ação |
|---------|------|
| `aurix-compliance/...ComunicacaoCoafService.java` | Novo |
| `aurix-compliance/...AnaliseCoafService.java` | Novo |
| `aurix-compliance/...CoafApiClient.java` | Novo |
| `aurix-compliance/...BasileiaService.java` | Novo |
| `aurix-compliance/...SolicitacaoLgpdController.java` | Novo |
| `aurix-compliance/...SolicitacaoLgpdService.java` | Novo |
| `aurix-bacen/...CompulsorioService.java` | Novo |
| `aurix-bacen/...ScrService.java` | Modificar (adicionar scheduled) |
| `aurix-shared/...JurosReguladosValidator.java` | Novo |
| `apps/backend/aurix-ouvidoria/` | Novo módulo |
| `apps/backend/pom.xml` | Adicionar `aurix-ouvidoria` |
| `infra/docker-compose.yml` | Adicionar `aurix-ouvidoria` |
| `infra/traefik/dynamic.yml` | Rota para `/api/ouvidoria` |

*Referência: business-gap-analysis.md — Gaps 3 (Regulatório), 10 (Juros).*
