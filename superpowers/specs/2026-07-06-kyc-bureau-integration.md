# KYC / Bureau / Fraud Integration — Design

## Goal
Replace stubs with real integrations for Serasa (bureau), Quod (bureau fallback), Unico (KYC), and ClearSale (fraud), using a Gateway + Strategy pattern with transparent fallback.

## Architecture

```
OnboardingPFService
  ├── BureauGateway (implements BureauService)
  │     ├── SerasaProvider  (tenta 1º, @Profile("!test"))
  │     ├── QuodProvider    (fallback se Serasa falhar)
  │     └── BureauStub      (@Profile("dev|test"))
  │
  ├── KycService (implements KycProviderService)
  │     ├── UnicoProvider   (@Profile("producao"))
  │     └── KycProviderStub (@Profile("!producao"))
  │
  └── FraudService (nova interface)
        ├── ClearSaleProvider (@Profile("producao"))
        └── FraudStub         (@Profile("!producao"))
```

### BureauGateway (fallback)
- Interface: `BureauService` (existente) — `ResultadoBureau consultar(String cpf)`
- Implementação injeta `List<BureauProvider>` ordenada por `@Order`
- Tenta cada provider em ordem; retorna o primeiro resultado bem-sucedido
- Se todos falham, lança exceção (circuit breaker futuro)
- Stub (`BureauStub`) com `@Profile("dev|test")` — ativo apenas em dev/test
- `BureauProvider` é interface nova:

```java
public interface BureauProvider {
    BureauService.ResultadoBureau consultar(String cpf);
}
```

### SerasaProvider
- Cliente HTTP REST (Spring `RestTemplate` / `WebClient`)
- Endpoint configurável via `application-prod.yml`:
  ```yaml
  aureus:
    onboarding:
      bureau:
        serasa:
          url: https://api.serasa.com.br/v2/score
          api-key: ${SERASA_API_KEY}
          timeout: 5000
  ```
- Mapeia resposta Serasa → `ResultadoBureau(score, situacao, mensagem)`
- Loga latência e resultado

### QuodProvider
- Mesma interface `BureauProvider`, usado como fallback
- Config separada:
  ```yaml
  aureus:
    onboarding:
      bureau:
        quod:
          url: https://api.quod.com.br/v1/consultas
          api-key: ${QUOD_API_KEY}
          timeout: 5000
  ```

### KycService
- Interface: `KycProviderService` (existente) — `ResultadoKyc validarDocumentos(cpf, documentos, selfieBase64)`
- Delega para `UnicoProvider` em produção
- Stub (`KycProviderStub`) permanece com `@Profile("!producao")`
- `UnicoProvider` chama API REST da Unico (ex-ID One):
  ```yaml
  aureus:
    onboarding:
      kyc:
        unico:
          url: https://api.unico.io/v1/biometrics
          api-key: ${UNICO_API_KEY}
  ```

### FraudService (novo)
- Interface nova para análise de fraude (ClearSale):
  ```java
  public interface FraudService {
      ResultadoFraude analisar(String cpf, String nome, String email, String telefone);
      record ResultadoFraude(boolean aprovado, String codigo, String mensagem, int risco) {}
  }
  ```
- `ClearSaleProvider` chama API REST do ClearSale
- `FraudStub` retorna `(true, "APROVADO_STUB", "Stub", 0)` ativo em `!producao`
- Chamado durante `solicitarAberturaConta()` após bureau, antes de salvar
- Se `risco > 70`, marca solicitação como `EM_ANALISE_KYC` (pendente) em vez de `RECEBIDA`

## Data Flow

### Solicitação PF
```
1. solicitarAberturaConta()
   ├── BureauGateway.consultar(cpf) → scoreBureau
   ├── FraudService.analisar(cpf, nome, email, telefone) → risco
   ├── PepRepository.existsByCpf() → pep
   ├── Cria SolicitacaoOnboarding (status = risco > 70 ? EM_ANALISE_KYC : RECEBIDA)
   └── Salva SolicitacaoPF com scoreBureau

2. enviarParaKyc()
   ├── Valida transição workflow
   ├── KycService.validarDocumentos(cpf, docs, selfie)
   └── Resultado: KYC_APROVADO ou KYC_REJEITADO

3. aprovar()
   ├── Valida transição workflow
   ├── Cria cliente+conta via CoreApiClient
   └── CONTA_CRIADA
```

### Error handling
- Timeout / falha de rede em Serasa → fallback Quod
- Timeout / falha em Quod → exceção (solicitação não criada)
- Timeout / falha em Unico → exceção (KYC não processado)
- Timeout / falha em ClearSale → assume aprovado (log warning), prossegue
- Todos os providers têm `@CircuitBreaker(name = "bureau|kyc|fraud", fallbackMethod = "fallback")`

## Config
```yaml
aureus:
  onboarding:
    bureau:
      serasa:
        url: https://api.serasa.com.br/v2/score
        api-key: ${SERASA_API_KEY}
        timeout: 5000
      quod:
        url: https://api.quod.com.br/v1/consultas
        api-key: ${QUOD_API_KEY}
        timeout: 5000
    kyc:
      unico:
        url: https://api.unico.io/v1/biometrics
        api-key: ${UNICO_API_KEY}
    fraud:
      clearsale:
        url: https://api.clearsale.com.br/v1/orders
        api-key: ${CLEARSALE_API_KEY}
```

## Changes required

### New files
| File | Purpose |
|------|---------|
| `BureauProvider.java` | Interface para providers de bureau |
| `BureauGateway.java` | Gateway com fallback, implementa BureauService |
| `SerasaProvider.java` | Cliente Serasa |
| `QuodProvider.java` | Cliente Quod |
| `UnicoProvider.java` | Cliente Unico |
| `FraudService.java` | Interface de fraude |
| `ClearSaleProvider.java` | Cliente ClearSale |
| `FraudStub.java` | Stub de fraude (`@Profile("!producao")`) |

### Modified files
| File | Change |
|------|--------|
| `BureauStub.java` | Adicionar `@Profile("dev|test")` |
| `OnboardingPFService.java` | Injetar `FraudService`, chamar `analisar()` |
| `SolicitacaoOnboarding.java` | Adicionar campo `riscoFraude` (Integer) |
| `application-prod.yml` | Adicionar config dos 4 provedores |
| `application.yml` | Adicionar config padrão (stubs) |
| `pom.xml` | Adicionar `spring-cloud-starter-circuitbreaker-resilience4j` |

### Testes
- `BureauGatewayTest` — fallback Serasa → Quod → exceção
- `FraudStubTest` — stub retorna aprovado
- `OnboardingPFFlowIntegrationTest` — atualizar mock para FraudService
- Testes de integração com WireMock para cada provider (opcional, fase 2)

## Migration
1. Criar interfaces e gateways, stubs ativos em `!producao` — sem mudança comportamental
2. Adicionar config de produção
3. Implementar providers reais um por um (Serasa → Quod → Unico → ClearSale)
4. Trocar profile de produção para usar providers reais
5. Remover stubs quando todos os providers estiverem validados
