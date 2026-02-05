# 14.3 Disaster Recovery (DR) – Procedimento e teste periodico

Procedimento de disaster recovery e teste periodico para o item 14.3 do roadmap.

## Objetivo

Garantir que, em caso de perda do datacenter ou do ambiente principal, exista procedimento documentado e validado por teste para restabelecer o servico dentro do RTO (Recovery Time Objective) e RPO (Recovery Point Objective) acordados.

## Pre-requisitos

- Backups automatizados (item 14.2) com retencao adequada e teste de restore trimestral.
- Configuracoes e secrets versionados ou replicados (repositorio, secret manager em regiao secundaria).
- Documentacao de topologia (URLs, DNS, banco, filas) para o ambiente de DR.

## Procedimento de DR (resumo)

1. **Declarar incidente DR**: decisao (ex.: CTO/operacoes) com base em indisponibilidade prolongada do ambiente principal.
2. **Ativar ambiente secundario**: subir ou apontar infra na regiao/DC secundario (Terraform, K8s, ou script de ativacao). Usar backups e configs replicados.
3. **Restaurar dados**: restaurar o ultimo backup consistente do PostgreSQL (e demais stores) no ambiente DR. Ver [backup-restore.md](backup-restore.md).
4. **Atualizar DNS e configs**: apontar DNS (ou CNAME) do produto para o ambiente DR; atualizar variaveis de ambiente e secrets que referenciem o ambiente antigo.
5. **Validar servicos**: health de todos os servicos criticos; smoke tests (login, uma transacao, health do gateway).
6. **Comunicar**: notificar clientes e times internos sobre ativacao do DR e possivel degradacao temporaria.
7. **Pos-incidente**: apos recuperacao do ambiente principal, planejar failback (sincronizar dados se houver escrita no DR, apontar DNS de volta, validar).

## Teste periodico

Recomendacao: **teste de DR pelo menos trimestral** (ou conforme politica interna).

### Passos do teste

1. **Agendar janela** (ex.: fim de semana ou horario de baixo uso).
2. **Documentar estado do ambiente principal** (ultimo backup usado, versao do codigo, config).
3. **Simular ativacao de DR**: em ambiente isolado (nao desligar o producao), executar os passos 2 a 5 do procedimento acima usando backup e config de producao (copia).
4. **Validar**: health checks, login, uma transacao critica (ex.: consulta de saldo, um PIX de teste).
5. **Registrar resultado**: duracao do "restore" no DR, pontos de falha, melhorias no procedimento ou nos backups.
6. **Atualizar este documento** e o runbook com qualquer mudanca no procedimento.

### Metricas do teste

- Tempo desde o "go" ate o health do gateway/core (objetivo: dentro do RTO).
- Idade do backup utilizado (objetivo: dentro do RPO).
- Checklist de validacao (todos os itens OK / falhas listadas).

## RTO e RPO

- **RTO** (tempo maximo para voltar ao ar): definir conforme contrato (ex.: 4 h ou 8 h). O teste de DR mede o tempo real e permite ajustar capacidade ou procedimento.
- **RPO** (perda maxima de dados aceitavel): depende da frequencia do backup (ex.: backup diario implica RPO de ate 24 h). Backup contínuo ou WAL shipping reduz o RPO.

## Referencias

- Roadmap e status: [../roadmap.md](../roadmap.md)
- Backup e restore: [backup-restore.md](backup-restore.md)
- Runbook: [aureus-cloud-runbook.md](aureus-cloud-runbook.md)
