# Testes End-to-End (E2E) - AURIX

Este projeto contem os testes end-to-end para validar a plataforma AURIX rodando completa em containers Docker (infraestrutura + servicos de backend).

## Estrutura

- `tests/e2e/config.py` - Configuracao de URLs e timeout dos testes.
- `tests/e2e/test_health_endpoints.py` - Testes que validam os endpoints de health de todos os servicos expostos no `infrastructure/docker-compose.yml`.
- `tests/e2e/requirements.txt` - Dependencias Python para execucao dos testes.
- `infrastructure/scripts/run-e2e-tests.bat` e `run-e2e-tests.sh` - Scripts para subir a infra com Docker Compose, executar os testes E2E e derrubar os containers.

## Pre-requisitos

- Docker e Docker Compose instalados.
- Python 3.10+ instalado.
- Pacotes Python: `pip install -r tests/e2e/requirements.txt`

## Como executar os testes E2E

Windows:
```bash
cd infrastructure\scripts
run-e2e-tests.bat
```

Linux/macOS:
```bash
cd infrastructure/scripts
./run-e2e-tests.sh
```

O script vai: subir a infraestrutura e servicos AURIX com `docker-compose.yml`, aguardar estabilizacao, executar `pytest` em `tests/e2e`, e derrubar os containers ao final.
