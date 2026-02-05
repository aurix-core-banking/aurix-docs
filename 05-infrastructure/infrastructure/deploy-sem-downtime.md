# 14.5 Deploy sem downtime

Estrategias blue/green e canary e migrations versionadas para deploy sem downtime (item 14.5 do roadmap).

## Migrations versionadas

- **Recomendacao**: usar Flyway ou Liquibase no projeto para versionar DDL e DML. Executar migrations no pipeline de deploy **antes** de trocar a versao dos servicos (ou no startup da nova versao, conforme estrategia).
- **Ordem**: aplicar migrations no banco; em seguida fazer o deploy dos pods/containers da nova versao. Assim a aplicacao nova so sobe apos o schema estar compativel.
- **Rollback**: manter compatibilidade retroativa por pelo menos uma versao (ex.: nova coluna como nullable) para nao quebrar a versao antiga ainda em execucao durante o cutover.

## Blue/Green

- Dois ambientes identicos (Blue e Green). Um atende o trafego; o outro recebe o deploy da nova versao.
- Apos validar a nova versao no ambiente “idle”, trocar o load balancer (ou ingress) para apontar para esse ambiente.
- **Vantagem**: rollback imediato revertendo o ponteiro do LB. **Custo**: duplicar recursos durante o deploy.

**Exemplo (Kubernetes):** dois deployments (aureus-core-blue, aureus-core-green) e um Service que aponta para um deles; apos deploy na “green”, alterar o selector do Service para “green”.

## Canary

- Parte do trafego (ex.: 5% ou 10%) vai para a nova versao; o restante continua na versao atual.
- Se metricas de erro ou latencia permanecerem dentro do SLA, aumentar gradualmente o trafego para a nova versao ate 100%.
- **Vantagem**: menor impacto se a nova versao tiver defeito. **Requer**: load balancer ou ingress com suporte a canary (peso por backend, ou header/cookie).

**Exemplo (Kubernetes):** NGINX Ingress com annotation de canary (canary weight) ou Istio VirtualService com subset e peso.

## Pipeline sugerido

1. Build e testes (CI).
2. Publicar imagem Docker com tag da versao.
3. Em staging: deploy da nova versao; rodar smoke tests e testes de regressao.
4. Em producao: aplicar migrations (Flyway/Liquibase) se houver.
5. Deploy da nova versao em producao (blue/green ou canary).
6. Validar health e metricas; se canary, aumentar peso ate 100%.
7. Em caso de falha: rollback (reverter LB em blue/green ou reduzir canary para 0%).

## Referencias

- Roadmap e status: [../roadmap.md](../roadmap.md)
- Runbook R1: [aureus-cloud-runbook.md](aureus-cloud-runbook.md) (servico nao sobe apos deploy)
- Deploy centralizado: [aureus-cloud-runbook.md](aureus-cloud-runbook.md) (Pipeline, Migrations)
