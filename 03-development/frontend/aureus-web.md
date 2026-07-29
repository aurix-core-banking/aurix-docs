# AURIX Internet Banking - Frontend

Frontend React para o AURIX Internet Banking.

## Tecnologias

- React 18, Material-UI (MUI) 5, React Router, Axios, Recharts, React Hook Form, React Query, Framer Motion, Date-fns

## Instalacao e execucao

```bash
npm install
npm start
```

Aplicacao em `http://localhost:3000`

## Estrutura

- `src/components/` - Componentes reutilizaveis (Dashboard, Navbar, Sidebar)
- `src/pages/` - Login, Dashboard, Contas, Transacoes, PIX, Investimentos, Cartoes, Perfil, Configuracoes
- `src/services/` - authService, apiService

## Funcionalidades

Login (CPF/senha, MFA, biometria), Dashboard, Contas, Transacoes, PIX, Investimentos, Cartoes, Perfil, Configuracoes.

## Variaveis de ambiente

`.env`: `REACT_APP_API_URL=http://localhost:8080/api`
