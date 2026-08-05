# Playbook – Incidente de segurança

Procedimento de resposta a incidentes de segurança no ecossistema Aurix (vazamento de dados, acesso indevido, token/chave comprometida).

> Requisitos regulatórios: LGPD e normativos BACEN exigem notificação em prazos definidos. Registrar **todo** o incidente para auditoria.

## Classificação

| Severidade | Exemplo | Resposta |
|-----------|---------|----------|
| Crítica | Vazamento de dados de clientes, acesso a produção, chave privada exposta | Acionar imediatamente time de resposta; comunicação formal |
| Alta | Token de um cliente comprometido, API key de integração exposta | Conter no mesmo dia; revogar e rotacionar |
| Média | Log com dado sensível, permissão excessiva | Corrigir na sprint; revisar logs |
| Baixa | Versão de dependência vulnerável (sem exploit ativo) | Patch no ciclo normal |

## Passo a passo

### 1. Conter

- **Isolar**: revogar tokens/chaves comprometidos, bloquear usuário/instituição, tirar o serviço do balanceador se necessário (`docker compose -f docker-compose.v2.yml stop <servico>`).
- **Rotacionar segredos**: senhas de banco, API keys (Bureau, ClearSale, Quod, ReceitaFederal, BACEN), JWT keys no Keycloak.
- **Congelar** deploy e acesso à área afetada até entender o alcance.

### 2. Avaliar alcance

- Quais dados foram afetados (PII, contas, transações)?
- Quais tenants/instituições e por quanto tempo?
- Como ocorreu (vetor): credencial vazada, vulnerabilidade, insiders?
- Coletar e preservar evidências (logs, métricas, snapshots) — não sobrescrever.

### 3. Comunicar

- **Interno**: acionar time de segurança/dev/infra e definir porta-voz único.
- **Regulatório**: LGPD (ANPD, se aplicável) e BACEN conforme contrato — prazos podem ser curtos; preparar notificação com dados, alcance e medidas tomadas.
- **Clientes**: comunicar de acordo com a política e contrato.

### 4. Recuperar

- Remover o vetor (corrigir vulnerabilidade, rotacionar tudo que foi exposto).
- Restaurar dados se houve alteração/indisponibilidade — ver [backup-restore](../backup-restore.md).
- Subir serviços e validar health + fluxos críticos.

### 5. Pós-incidente

- Documentar: cronologia, causa raiz, ações, lições aprendidas.
- Revisar acessos e princípio do menor privilégio; revisar logs de auditoria.
- Atualizar este playbook e as regras de detecção (SIEM/alertas).

## Boas práticas permanentes

- Secrets nunca no repositório; usar Secret Manager/Keycloak.
- Rotação periódica de chaves e revisão de acessos.
- Logs sem PII desnecessária (mascarar CPF, cartão, etc.).
- MFA obrigatório para acesso administrativo e a ferramentas de produção.
