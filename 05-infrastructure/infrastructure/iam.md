# AURIX - Identity and Access Management (IAM)

Configuracao do IAM com Keycloak para o AURIX Core Banking.

## Visao Geral

- Keycloak 23.0.0, porta 8080, realm `aurix`, banco PostgreSQL (keycloak)
- Clientes: aurix-web, aurix-mobile, aurix-api
- Roles: admin, gerente, operador, cliente, auditor, compliance, tesouraria, credito, pix, analytics
- Grupos: Diretores, Gerencia, Operadores, Clientes, Auditores, Compliance

## Configuracao

1. Subir servicos: `cd infrastructure && docker-compose up -d`
2. Aguardar Keycloak: `curl -f http://localhost:8080/health/ready`
3. Configurar: `iam/scripts/setup-keycloak.sh` ou `iam\scripts\setup-keycloak.bat`

Admin Console: http://localhost:8080/admin (admin / admin123). Importar realm de `iam/keycloak/import/aurix-realm.json`.

## Uso

- Autenticacao: fluxo Web (Keycloak JS), Client Credentials (API), Resource Owner Password (mobile)
- Autorizacao: hasRealmRole(), claims (cpf, matricula, departamento, cargo)
- Backend: validacao de token, Spring Security OAuth2 Resource Server

## Estrutura

- `iam/keycloak/import/`: aurix-realm.json, aurix-clients.json, aurix-roles.json, aurix-users.json, aurix-groups.json, aurix-client-scopes.json
- `iam/scripts/`: setup-keycloak.sh, setup-keycloak.bat

## Producao

SSL obrigatorio, PostgreSQL com replicacao, monitoramento Prometheus/Grafana, backup do banco keycloak.
