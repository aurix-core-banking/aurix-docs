# Instalação – Aureus Admin

Guia para instalar e rodar o painel administrativo (frontend) em ambiente local.

---

## Pré-requisitos

- Node.js 16.x ou superior, npm 8.x ou superior, Git

---

## Instalação

1. Clone o repositório e acesse `frontend/aureus-admin`
2. `npm install`
3. Copie `config.example.js` para `config.js` e ajuste a URL da API
4. `npm start` (ou `start-dev.bat` / `./start-dev.sh`)
5. Acesse `http://localhost:3000`

---

## Variáveis de ambiente

`.env`: `REACT_APP_API_URL=http://localhost:8080`, `REACT_APP_DEBUG`, `REACT_APP_DEFAULT_THEME`, etc.

---

## Build para produção

`npm run build`. Arquivos em `build/`.

---

## Referências

- [aureus-admin-desenvolvimento.md](../../03-development/frontend/aureus-admin-desenvolvimento.md) – fluxo de desenvolvimento do admin
- [Setup da plataforma](../02-lifecycle/setup.md)

[Voltar a Guias](README.md) | [Índice da wiki](../README.md)
