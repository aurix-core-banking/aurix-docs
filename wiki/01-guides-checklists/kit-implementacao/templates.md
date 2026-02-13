# Templates – Contrato, política de privacidade e termos

Templates editáveis por tenant para contrato de adesão, política de privacidade e termos de uso. Item 13.3 do roadmap.

---

## Onde configurar

- **TenantConfig (provisioning)**: o campo `termosUsoUrl` armazena a URL dos termos de uso exibidos ao cliente (ex.: no fluxo de consentimento ou no app). A instituição pode hospedar a página em seu próprio domínio e informar a URL na config do tenant.
- **Templates como arquivos**: os textos completos (contrato de adesão, política de privacidade, termos) podem ser mantidos como arquivos editáveis no repositório ou em um storage (ex.: S3) e referenciados por URL na config do tenant.

---

## Estrutura sugerida

| Documento | Uso | Onde definir |
|-----------|-----|--------------|
| **Termos de uso** | Condições de uso do produto/serviço | TenantConfig.termosUsoUrl ou arquivo template por tenant |
| **Política de privacidade** | LGPD; tratamento de dados | URL em config ou template; exibida no onboarding e no portal |
| **Contrato de adesão** | Contrato comercial com a instituição | Template em Markdown/HTML; variáveis: {{nome_instituicao}}, {{data}}, {{plano}} |

---

## Variáveis nos templates

Para permitir personalização por tenant e data, os templates podem usar placeholders:

- `{{nome_instituicao}}` – nome da instituição (tenant)
- `{{tenant_id}}` – identificador do tenant
- `{{data}}` – data de geração ou vigência
- `{{plano}}` – plano contratado (Starter, Growth, Enterprise)
- `{{url_privacidade}}` – link para política de privacidade
- `{{url_termos}}` – link para termos de uso

O serviço que gera o documento (ex.: contrato de adesão em PDF ou página HTML) substitui essas variáveis pelos valores do tenant no momento da geração.

---

## Armazenamento

- **Opção 1**: templates em pasta de templates no repositório (ex.: `termos-uso.md`, `politica-privacidade.md`, `contrato-adesao.md`) com variáveis; em produção, um job ou API gera a versão final por tenant.
- **Opção 2**: cada tenant informa URLs próprias na config (termosUsoUrl, politicaPrivacidadeUrl); a instituição edita o conteúdo no próprio site.
- **Opção 3**: armazenamento em bucket (S3/MinIO) com um arquivo por tenant (ex.: `tenants/{tenant_id}/termos.html`); a URL é configurada no TenantConfig.

A implementação atual suporta a Opção 2 (URL por tenant). A inclusão de templates padrão editáveis (Opção 1) pode ser feita em um módulo de documentos.

[Voltar ao kit](README.md) | [Índice da wiki](../../README.md)
