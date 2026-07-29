# 14.6 Feature flags por tenant

Habilitar ou desabilitar funcionalidades por tenant via config ou tabela (item 14.6 do roadmap).

## Implementacao

- **Tabela**: `aurix.tenant_feature_flags` (tenant_id, feature_key, enabled, descricao). Criada pelo JPA (ddl-auto: update) no modulo aurix-provisioning. Para ambientes com ddl-auto: validate, executar o DDL manualmente ou via Flyway/Liquibase.
- **API**: modulo **aurix-provisioning** expoe endpoints REST em `/api/provisioning/tenants/{tenantId}/features`.

### Endpoints

| Metodo | Path | Descricao |
|--------|------|-----------|
| GET | /api/provisioning/tenants/{tenantId}/features | Lista todas as flags do tenant |
| GET | /api/provisioning/tenants/{tenantId}/features/{featureKey} | Retorna a flag |
| GET | /api/provisioning/tenants/{tenantId}/features/{featureKey}/enabled | Retorna true/false |
| PUT | /api/provisioning/tenants/{tenantId}/features/{featureKey} | Cria ou atualiza (body: { "enabled": true, "descricao": "..." }) |
| DELETE | /api/provisioning/tenants/{tenantId}/features/{featureKey} | Remove a flag |

### Uso em outros modulos

Para verificar se uma funcionalidade esta habilitada para o tenant atual:

1. **Chamada HTTP**: GET `http://aurix-provisioning:8096/api/provisioning/tenants/{tenantId}/features/{featureKey}/enabled`. Retorno: `true` ou `false`.
2. **Cache**: recomenda-se cache de curta duracao (ex.: 1 minuto) por (tenantId, featureKey) para evitar muitas chamadas ao provisioning.
3. **Exemplo de chaves**: `pix_qr_code`, `open_finance`, `cartao_prepago`, `internet_banking`, `mobile_banking`, `regtech_pacote`, etc.

### DDL (opcional, quando nao usar ddl-auto update)

```sql
CREATE TABLE IF NOT EXISTS aurix.tenant_feature_flags (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(64) NOT NULL,
    feature_key VARCHAR(128) NOT NULL,
    enabled BOOLEAN NOT NULL DEFAULT true,
    descricao VARCHAR(512),
    data_criacao TIMESTAMP NOT NULL DEFAULT NOW(),
    data_atualizacao TIMESTAMP,
    UNIQUE (tenant_id, feature_key)
);
CREATE INDEX idx_tenant_feature_flags_tenant ON aurix.tenant_feature_flags(tenant_id);
```

## Referencias

- Roadmap e status: [../roadmap.md](../roadmap.md)
- Modulo: `backend/aurix-provisioning/`
- Runbook (config): [aurix-cloud-runbook.md](aurix-cloud-runbook.md)
