# SPI/STR – Certificados e credenciais

Configuração interna para integração de produção com o Sistema de Pagamentos Instantâneos (SPI) e o Sistema de Transferência de Reservas (STR) do BACEN.

---

## Ambientes

| Ambiente   | Uso      | SPI (exemplo)                     | STR (exemplo)                     |
|------------|----------|------------------------------------|------------------------------------|
| homologação| Testes   | https://spi-homologacao.bcb.gov.br | https://str-homologacao.bcb.gov.br |
| produção   | Operação | https://spi.bcb.gov.br             | https://str.bcb.gov.br             |

As URLs exatas são definidas pelo BACEN e devem ser conferidas na documentação oficial e no contrato de adesão.

---

## Certificado digital

A conexão com SPI/STR em ambiente real exige certificado digital (cliente) emitido para o ISPB da instituição. O certificado é instalado em keystore (ex.: PKCS12). O truststore contém as CAs do BACEN (e raiz) para validar o servidor.

### Obtenção

1. Contratar/adesão ao SPI e STR junto ao BACEN (processo institucional).
2. Obter certificado digital para o ambiente (homologação e produção).
3. Exportar o certificado para PKCS12 (.p12) ou manter em formato aceito pela JVM (JKS/PKCS12).

### Configuração no AUREUS

No módulo `aureus-bacen`, em `application.yml` ou em perfis `application-homolog.yml` / `application-prod.yml`:

```yaml
aureus:
  bacen:
    environment: homologacao   # ou producao
    ispb: "12345678"
    certificados:
      keystore-path: /caminho/seguro/keystore.p12
      keystore-password: ${BACEN_KEYSTORE_PASSWORD}
      truststore-path: /caminho/seguro/truststore.jks
      truststore-password: ${BACEN_TRUSTSTORE_PASSWORD}
      keystore-type: PKCS12
      truststore-type: JKS
    spi:
      enabled: true
      url: https://spi-homologacao.bcb.gov.br
      connect-timeout-ms: 10000
      read-timeout-ms: 30000
    str:
      enabled: true
      url: https://str-homologacao.bcb.gov.br
      connect-timeout-ms: 10000
      read-timeout-ms: 30000
```

**Segurança**

- Não commitar senhas no repositório. Usar variáveis de ambiente (ex.: `BACEN_KEYSTORE_PASSWORD`, `BACEN_TRUSTSTORE_PASSWORD`) ou secret manager.
- Restringir leitura dos arquivos de keystore/truststore ao usuário do processo da aplicação.

### Uso do certificado no WebClient

Quando `certificados.keystore-path` estiver preenchido, o módulo pode configurar o WebClient (SPI/STR) com SSL mútuo. A implementação atual utiliza os beans `spiWebClient` e `strWebClient` com timeouts configurados; a injeção de SSLContext a partir do keystore/truststore pode ser adicionada em `SpiStrWebClientConfig` carregando o keystore/truststore e configurando um `HttpClient` com `SslContext`. Em ambiente de homologação sem certificado real, manter `spi.enabled: false` e `str.enabled: false` para operar em modo simulado.

---

## Retry e circuit breaker

O módulo aplica Resilience4j aos clientes SPI e STR:

- **Retry**: 3 tentativas, intervalo 1s, para falhas de rede/WebClient.
- **Circuit breaker**: janela 10 chamadas, 50% de falha abre o circuito por 30s; 3 chamadas em half-open para testar recuperação.

Métricas de retry e circuit breaker são expostas pelo Actuator (Micrometer). Logs de erro e fallback são registrados pelo cliente (`SpiStrApiClientImpl`).

---

## Perfis de configuração

- **dev**: `spi.enabled: false`, `str.enabled: false` (simulado).
- **homolog**: ativar SPI/STR com URLs de homologação e certificado de homologação.
- **prod**: ativar SPI/STR com URLs de produção e certificado de produção; trocar senhas e caminhos por secrets.

---

## Referências

- Documentação BACEN sobre SPI e STR (sites oficiais e manuais de integração).
- Normas do Banco Central para adesão e certificação.
- [Regulatory pack](../../../01-business/conformidade/regulatory-pack.md)
- [Checklist regulatório](../../01-guides-checklists/checklists/regulatorio.md)

[Conformidade na wiki](README.md) | [Referências](../../05-references/referencias/README.md) | [Índice da wiki](../../README.md)
