# Runbooks operacionais

Procedimentos operacionais para diagnóstico e recuperação de incidentes comuns no ecossistema Aurix.

| Runbook | Quando usar |
|---------|-------------|
| [Queda de serviço](runbook-queda-de-servico.md) | Serviço `svc-*` fora do ar, não sobe ou responde com erro |
| [Lentidão no banco](runbook-lentidao-no-banco.md) | Queries lentas, locks, conexões esgotadas, failover |
| [Falha no Kafka](runbook-falha-no-kafka.md) | Consumidor parado, lag crescente, replay de mensagens |
| [Incidente de segurança](runbook-security-incident.md) | Vazamento, acesso indevido, token/chave comprometida |
| [Rollback de deploy](runbook-rollback-deploy.md) | Deploy com defeito, migração com falha, reverter versão |

Cada alerta de monitoramento deve referenciar um item desta tabela. Ver também: [AURIX Cloud – Runbook e alertas](../aureus-cloud-runbook.md) (R1 a R6), [DR – procedimento](../dr-procedimento.md) e [Backup e restore](../backup-restore.md).
