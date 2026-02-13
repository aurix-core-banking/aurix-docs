# Checklist – Go live (novo cliente)

Checklist para novo cliente (instituição) antes do go live. Corresponde ao item 13.1 do roadmap.

---

## 1. BACEN e certificados

- [ ] **Adesão SPI/STR**: processo de adesão ao Sistema de Pagamentos Instantâneos e ao Sistema de Transferência de Reservas concluído com o BACEN; ISPB da instituição definido.
- [ ] **Certificado digital**: certificado (cliente) para homologação e produção obtido e exportado para keystore (PKCS12 ou JKS).
- [ ] **Truststore**: CAs do BACEN (e raiz) configuradas no truststore para validação TLS.
- [ ] **Config no módulo bacen**: `aureus.bacen.ispb`, `certificados.keystore-path`, `certificados.truststore-path`, senhas em variáveis de ambiente ou secret manager; `spi.enabled` / `str.enabled` e URLs conforme ambiente.
- [ ] **Teste de conectividade**: em homologação, validar chamadas SPI/STR (ou modo simulado se certificado ainda não disponível).

Referência: [SPI/STR certificados](../../03-compliance/conformidade/spi-str-certificados.md).

---

## 2. SPI/STR (produção)

- [ ] **Ambiente homologação**: testes com certificado de homologação; validação de mensagens e retry/circuit breaker.
- [ ] **Ambiente produção**: troca para URLs e certificado de produção; `aureus.bacen.environment: producao`.
- [ ] **Monitoramento**: logs e métricas de sucesso/falha das chamadas SPI/STR; alertas para falhas consecutivas.

---

## 3. KYC (provedor)

- [ ] **Provedor escolhido**: ID One ou outro provedor de KYC contratado; credenciais e endpoints de homologação/produção.
- [ ] **Config no módulo onboarding**: URL da API do provedor, API Key ou OAuth; webhook de callback (se aplicável).
- [ ] **Fluxo de aprovação**: regras de aprovação automática vs. análise manual; checklist de documentos por tipo de conta.
- [ ] **Teste**: envio de documento e selfie de teste; validação do retorno (aprovado/rejeitado/pendente).

---

## 4. Domínios e URLs

- [ ] **Domínio da instituição**: domínio principal (ex.: `banco.exemplo.com`) definido e DNS configurado.
- [ ] **API / Internet Banking**: subdomínio ou path para APIs e portal (ex.: `api.banco.exemplo.com`, `app.banco.exemplo.com`).
- [ ] **Certificado TLS**: certificado SSL/TLS (ex.: Let's Encrypt ou comercial) instalado no gateway ou load balancer para o domínio.
- [ ] **CORS e redirects**: se aplicável, origens permitidas e URLs de callback (OAuth, webhook) configuradas.

---

## 5. Provisioning e tenant (AUREUS Cloud)

- [ ] **Instituição cadastrada**: registro no módulo provisioning (tenant_id, nome, CNPJ, contato, plano).
- [ ] **Perfil de implantação**: tenancy (single por padrão; multi-tenant se SaaS), cloud (AWS/Azure/GCP/etc.), topologia definidos.
- [ ] **Config do tenant**: branding (logo, cores), limites, produtos habilitados; termos de uso (URL) se aplicável.
- [ ] **Provisioning executado**: banco e config do tenant criados; acesso validado (login ou API com X-Tenant-Id).

---

## 6. RegTech e relatórios

- [ ] **Relatórios obrigatórios**: E-Financeira, SCR/CCS, SPED (ECD, ECF, EFD-Reinf), BACEN Jud conforme calendário da instituição.
- [ ] **Agendamento**: jobs de geração e (quando cabível) envio configurados; dashboard de status de reportes.
- [ ] **Regulatory pack**: versão do pacote regulatório alinhada ao ambiente (homolog/prod).

---

## 7. Segurança e compliance

- [ ] **Senhas e segredos**: nenhum segredo em repositório; uso de secret manager ou variáveis de ambiente.
- [ ] **LGPD**: política de privacidade e termos de uso publicados; fluxo de consentimento e registro de consentimentos (compliance/onboarding).
- [ ] **Auditoria**: log de acessos e alterações críticas habilitado; retenção conforme política.

---

## Ordem sugerida

1. BACEN/certificados e SPI/STR (bloqueante para PIX real).
2. KYC e domínios.
3. Provisioning e config do tenant.
4. RegTech e segurança.

Quando todos os itens relevantes estiverem marcados, o go live pode ser agendado conforme runbook e guia "Do zero ao primeiro PIX" ([guia-zero-primeiro-pix.md](../kit-implementacao/guia-zero-primeiro-pix.md)).

[Voltar aos checklists](README.md) | [Índice da wiki](../../README.md)
