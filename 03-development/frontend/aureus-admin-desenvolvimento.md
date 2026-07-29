# Guia de Desenvolvimento – Aurix Admin

## Estrutura do Projeto

- `src/components/` – Componentes reutilizáveis (Layout, Loading, Error)
- `src/pages/` – Dashboard, Clientes, Contas, Transações, Investimentos, PIX, Compliance, Auditoria, Analytics
- `src/providers/` – authProvider, dataProvider, i18nProvider
- `src/config/`, `src/constants/`, `src/hooks/`, `src/utils/`
- `theme.js`, `App.js`, `index.js`

## Padrões

Seguir convenções do React Admin e Material-UI. Novos módulos: criar pasta em `src/pages/`, implementar List/Create/Edit/Show, registrar em `App.js` e na API.

[Voltar ao frontend](aurix-admin.md) | [Instalação (wiki)](../wiki/guias/instalacao-admin.md)
