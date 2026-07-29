# PIX → BACEN SPI/STR Integration Design

## Overview

Wire the PIX transfer flow to the BACEN SPI (Sistema de Pagamentos Instantâneos) engine via synchronous REST calls, replacing the current simulation/log-only pattern. Add a WireMock-based BACEN mock container for local development that can be swapped for the real BACEN sandbox when credentials are available.

**Status:** Design approved — ready for implementation planning.

---

## Architecture

```
POST /api/pix/transferencias → PixTransferenciaController
  │
  └─ PixTransferenciaService.processarTransferencia()
       │
       ├─ 1. Valida status PENDENTE
       ├─ 2. Débito atômico (UPDATE saldo WHERE saldo >= valor)
       ├─ 3. Chama PixBacenClient.enviarPix(transacaoSPI)
       │      └─ REST POST → aurix-bacen:8094/api/spi-str/spi/pix
       │           └─ SpiStrApiClientImpl
       │                ├─ dev/sandbox → http://bacen-mock:8095 (mock)
       │                └─ homolog/prod → https://spi-homologacao.bcb.gov.br (mTLS)
       ├─ 4. Se sucesso (LIQUIDADA):
       │      ├─ Se chave local → crédito atômico no DB
       │      └─ Se chave externa → SPI já liquidou, só atualizar status
       ├─ 5. Se falha (rejeitado/timeout):
       │      ├─ Estorna débito (creditarSaldoAtomico)
       │      └─ Marca transferência como FALHA
       └─ 6. Publica evento TransacaoEvent via Outbox
```

### Components

| Component | Module | Change |
|-----------|--------|--------|
| `PixBacenClient` | `aurix-pix` (new) | REST client → `aurix-bacen` SPI endpoint |
| `PixTransferenciaService` | `aurix-pix` | Inject `PixBacenClient`, call SPI for ALL transfers |
| `SpiStrIntegrationService` | `aurix-bacen` | No code change — just enable via config |
| `SpiStrApiClientImpl` | `aurix-bacen` | Skip mTLS guard when URL is localhost/mock |
| `bacen-mock` | `infra/bacen-mock/` (new) | WireMock container |
| `docker-compose.yml` | `infra/` | Add bacen-mock service |

---

## PixBacenClient

**File:** `apps/backend/aurix-pix/src/main/java/com/aurix/platform/pix/client/PixBacenClient.java`

```java
@Component
public class PixBacenClient {
    private final RestClient restClient;

    public PixBacenClient(RestClient.Builder builder,
                          @Value("${aurix.pix.bacen.spi-url}") String spiUrl) {
        this.restClient = builder.baseUrl(spiUrl).build();
    }

    public SpiResult enviarPix(TransacaoSPI transacao) {
        return restClient.post()
            .uri("/api/spi-str/spi/pix")
            .body(transacao)
            .retrieve()
            .body(SpiResult.class);
    }
}
```

**Config:** `aurix.pix.bacen.spi-url: ${PIX_BACEN_SPI_URL:http://localhost:8094}`

---

## PixTransferenciaService — wire changes

In `processarTransferencia()`, replace the current logic:

**Before:**
```java
if (chaveDestino.isLocal()) {
    contaRepository.creditarSaldoAtomico(tenantId, contaDestinoId, valor);
} else {
    log.info("Chave PIX destino {} não é local — crédito ocorre na instituição destino via SPI", chaveDestino.getValorChave());
}
```

**After:**
```java
// Monta transacao SPI
TransacaoSPI transacaoSPI = new TransacaoSPI();
transacaoSPI.setEndToEndId(gerarEndToEndId());
transacaoSPI.setIspbOrigem(config.getIspb());
transacaoSPI.setIspbDestino(obterIspbDestino(chaveDestino));
transacaoSPI.setValor(pixTransferencia.getValor());
transacaoSPI.setChavePixDestino(chaveDestino.getValorChave());

// Envia para SPI
SpiResult resultado;
try {
    resultado = pixBacenClient.enviarPix(transacaoSPI);
} catch (Exception e) {
    log.error("Falha ao enviar PIX para SPI: {}", e.getMessage());
    estornarDebito(pixTransferencia, contaOrigemId);
    pixTransferencia.setStatus("FALHA");
    pixTransferenciaRepository.save(pixTransferencia);
    throw new IllegalStateException("Falha na comunicação com SPI", e);
}

if (!resultado.isSucesso()) {
    estornarDebito(pixTransferencia, contaOrigemId);
    pixTransferencia.setStatus("REJEITADA");
    pixTransferencia.setMensagemErro(resultado.getMensagem());
    pixTransferenciaRepository.save(pixTransferencia);
    throw new IllegalStateException("PIX rejeitado pelo SPI: " + resultado.getMensagem());
}

// SPI liquidou — crédito local se for chave interna
if (chaveDestino.isLocal()) {
    contaRepository.creditarSaldoAtomico(tenantId, contaDestinoId, valor);
}

pixTransferencia.setStatus("PROCESSADA");
pixTransferencia.setEndToEndId(resultado.getEndToEndId());
pixTransferenciaRepository.save(pixTransferencia);
```

### Compensação (estorno)

```java
private void estornarDebito(PixTransferencia pix, Long contaOrigemId) {
    int estornado = contaRepository.creditarSaldoAtomico(
        TenantContext.getTenantId(), contaOrigemId, pix.getValor());
    if (estornado == 0) {
        log.error("CRÍTICO: estorno falhou para transferência ID {}", pix.getId());
    }
}
```

---

## SPI/STR — habilitar em dev

**File:** `apps/backend/aurix-bacen/src/main/resources/application-dev.yml` (new)

```yaml
aurix:
  bacen:
    spi:
      enabled: true
      url: ${PIX_BACEN_SPI_URL:http://bacen-mock:8095}
    str:
      enabled: true
      url: ${PIX_BACEN_STR_URL:http://bacen-mock:8095}
    certificados:
      keystore-path: ${BACEN_KEYSTORE_PATH:}
      keystore-password: ${BACEN_KEYSTORE_PASSWORD:}
```

### SpiStrApiClientImpl — skip mTLS for mock/localhost

Add method:
```java
private boolean isMockUrl() {
    String url = spiProperties.getSpi().getUrl();
    return url.contains("localhost") || url.contains("bacen-mock");
}
```

In `enviarPixSpi()` and `enviarTedStr()`/`enviarDocStr()`:
```java
if (isMockUrl()) {
    webClient.post()... // sem mTLS
} else if (!spiProperties.isCertificadoConfigurado()) {
    falharSeProducaoSemCertificado();
}
```

---

## BACEN Mock — WireMock container

**Directory:** `infra/bacen-mock/`

### Dockerfile
```dockerfile
FROM wiremock/wiremock:latest
COPY mappings/ /home/wiremock/mappings/
EXPOSE 8095
```

### mappings/spi-pix.json
```json
{
  "request": {
    "method": "POST",
    "url": "/api/spi-str/spi/pix"
  },
  "response": {
    "status": 200,
    "jsonBody": {
      "sucesso": true,
      "endToEndId": "E0000000020240710XXXXXXXXXXXX",
      "status": "LIQUIDADA",
      "dataHoraLiquidacao": "{{now}}",
      "ispbDestino": "12345678",
      "mensagem": "Transacao PIX liquidada com sucesso (simulacao)"
    },
    "headers": { "Content-Type": "application/json" }
  }
}
```

### mappings/spi-status.json
```json
{
  "request": {
    "method": "GET",
    "urlPattern": "/api/spi-str/spi/status/.*"
  },
  "response": {
    "status": 200,
    "jsonBody": {
      "endToEndId": "E0000000020240710XXXXXXXXXXXX",
      "status": "LIQUIDADA"
    }
  }
}
```

### docker-compose.yml addition
```yaml
bacen-mock:
  image: wiremock/wiremock:latest
  container_name: aurix-bacen-mock
  ports:
    - "8095:8095"
  volumes:
    - ./bacen-mock/mappings:/home/wiremock/mappings
  command: ["--port", "8095", "--verbose"]
  networks:
    - aurix-network
```

---

## Environment Variables

**infra/.env.example** additions:
```env
# BACEN / PIX Sandbox
BACEN_SANDBOX=true                                    # true = mock local, false = sandbox real
BACEN_CLIENT_ID=replace_with_bacen_client_id          # Obrigatório quando SANDBOX=false
BACEN_CLIENT_SECRET=replace_with_bacen_client_secret  # Obrigatório quando SANDBOX=false
BACEN_KEYSTORE_PATH=                                  # Path do certificado ICP-Brasil
BACEN_KEYSTORE_PASSWORD=                              # Senha do keystore
PIX_BACEN_SPI_URL=http://bacen-mock:8095              # Mock local (default dev)
PIX_BACEN_STR_URL=http://bacen-mock:8095              # Mock local (default dev)
```

### Resolution logic (aurix-bacen startup)
1. If `BACEN_SANDBOX=true` → use `PIX_BACEN_SPI_URL` (mock), skip mTLS
2. If `BACEN_SANDBOX=false` → use fixed BCB URLs, require mTLS certs

---

## Tests

| Test | File | What it covers |
|------|------|----------------|
| `PixBacenClientTest` | `aurix-pix/src/test/.../client/` | HTTP serialization, error mapping |
| `PixTransferenciaServiceTest` | `aurix-pix/src/test/.../service/` | SPI success, failure, timeout, compensation |
| `PixFlowIntegrationTest` | `aurix-pix/src/test/.../integration/` | Full flow with mock BACEN |
| `SpiStrApiClientImplTest` | `aurix-bacen/src/test/.../client/` | Mock URL skip, mTLS guard |

---

## Implementation order

1. **BACEN Mock** — WireMock container + docker-compose (independently testable)
2. **SpiStrApiClientImpl guard** — mock URL skip + dev profile
3. **PixBacenClient** — REST client in aurix-pix
4. **PixTransferenciaService wire** — call SPI, compensation on failure
5. **Environment config** — .env.example, resolution logic
6. **Tests** — all levels (unit, integration)
7. **Documentation** — BACEN sandbox setup guide