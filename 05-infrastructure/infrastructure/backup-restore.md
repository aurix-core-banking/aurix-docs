# Backup e restore (item 14.2)

Procedimentos para backups automatizados do PostgreSQL, retencao e teste de restore, conforme item 14.2 do roadmap.

## Scripts

- **Backup**: `infra/scripts/backup-postgres.sh` (Linux/Mac) ou `backup-postgres.bat` (Windows).
- **Restore**: `infra/scripts/restore-postgres.sh` ou `restore-postgres.bat`.

Variaveis de ambiente (opcional):

| Variavel | Padrao | Descricao |
|----------|--------|-----------|
| POSTGRES_CONTAINER | aurix-postgres | Nome do container PostgreSQL |
| POSTGRES_USER | aurix_user | Usuario do banco |
| POSTGRES_PASSWORD | aurix_dev_password | Senha |
| POSTGRES_DB | aurix_db | Banco principal |
| BACKUP_DIR | ./backups | Diretorio onde salvar os dumps |
| RETENTION_DAYS | 30 | Remover backups mais antigos que N dias (apenas no .sh) |

## Backup

**Execucao manual (exemplo):**
```bash
cd infra/scripts
./backup-postgres.sh
```

**Automatizacao (cron, diario 02:00):**
```cron
0 2 * * * cd /caminho/para/aurix-platform/infra/scripts && BACKUP_DIR=/var/backups/aurix RETENTION_DAYS=30 ./backup-postgres.sh
```

O script gera:
- `aurix_YYYYMMDD_HHMMSS.sql` (banco principal)
- `keycloak_YYYYMMDD_HHMMSS.sql` (se existir)

Retencao: no script `.sh`, backups com mais de `RETENTION_DAYS` dias sao removidos. No Windows, limpar manualmente ou agendar tarefa equivalente.

## Restore

**Uso:**
```bash
./restore-postgres.sh <arquivo.sql> [database]
# database: aurix_db (padrao) ou keycloak
```

**Exemplo:**
```bash
./restore-postgres.sh /var/backups/aurix/aurix_20260203_020001.sql aurix_db
```

Recomendacao: parar os servicos que usam o banco antes do restore (ou usar banco de staging). Apos o restore, validar dados e reiniciar os servicos.

## Teste de restore (periodico)

Conforme item 14.2 e R6 do runbook, testar restore com periodicidade definida (ex.: trimestral):

1. Escolher um backup recente (ex.: ultimo dia util).
2. Em ambiente de teste/staging: subir um PostgreSQL limpo ou usar banco `aurix_restore_test`.
3. Executar restore do arquivo escolhido no banco de teste.
4. Validar: contagem de tabelas, amostra de dados (contas, transacoes), health dos servicos apontando ao banco de teste.
5. Registrar data e resultado do teste (OK / falha) para auditoria.

Exemplo de validacao rapida apos restore:
```bash
docker exec aurix-postgres psql -U aurix_user -d aurix_db -c "SELECT COUNT(*) FROM contas; SELECT COUNT(*) FROM transacoes;"
```

> O script de backup gera também o dump do banco `keycloak` (se existir) no mesmo diretório.

## Referencias

- Roadmap e status: [../roadmap.md](../roadmap.md)
- Runbook R6: [aurix-cloud-runbook.md](aurix-cloud-runbook.md) (Backup e restore)
- Runbooks operacionais: [runbooks/index.md](runbooks/index.md)
