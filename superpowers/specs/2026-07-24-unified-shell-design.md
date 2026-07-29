# Specification: AURIX Unified Shell (Dynamics 365 / Enterprise Suite Model)

**Date:** 2026-07-24  
**Status:** Approved  
**Target:** `apps/frontend/aurix-web` (with integrated modular shell supporting all Banking and Admin capabilities)

---

## 1. Overview & Objectives

The goal of the **AURIX Unified Shell** is to unify the frontend experience into a modern, enterprise-grade application shell inspired by Microsoft Dynamics 365 and cloud financial suites.

Key objectives:
- **Suite Top Bar (Header):** Featuring a 9-dot Waffle menu (App Launcher) to seamlessly switch between apps (*Banking*, *Admin*, *Investments*, *Credit*, *Compliance & Risk*).
- **Workspace Multi-Tab Bar:** Tabbed interface allowing users to open and switch between multiple financial modules without losing UI state or triggering full page reloads.
- **Dynamic Contextual Navigation Sidebar:** Sidebar navigation that automatically adapts to the currently active application selected in the suite launcher.
- **Global Command Palette (`Ctrl+K`):** Fast command and entity search modal for accessing pages, client records, transacting, and toggling tools.
- **Fluent & Elevated MUI 5 Theme System:** Premium dark/light corporate aesthetic with glassmorphism header, curated color palettes, smooth animations, and high information density layout.

---

## 2. Shell Layout Architecture

```
+---------------------------------------------------------------------------------------------------+
| 🌐 GLOBAL TOPBAR (Suite Bar)                                                                      |
| [::: Waffle Menu]  AURIX Suite | Financial Cloud     [🔍 Buscar (Ctrl+K)] [🔔 3] [⚙️] [🌙] [👤 User] |
+---------------------------------------------------------------------------------------------------+
| 📑 WORKSPACE TABS BAR                                                                            |
| [🏠 Inicio] [💳 Pix / Transferência x] [📊 Contas Admin x] [➕ Nova Aba]                           |
+------------------------------+--------------------------------------------------------------------+
| 📐 SIDEBAR CONTEXTUAL        | 🖥️ MAIN WORKSPACE CANVAS                                           |
| (Dynamic per active app)     |                                                                    |
|  • Dashboard                 | (Active tab page component rendered within shell container)        |
|  • Contas & Saldos           |                                                                    |
|  • Pix & Pagamentos          |                                                                    |
|  • Investimentos             |                                                                    |
|  • Cartões                   |                                                                    |
+------------------------------+--------------------------------------------------------------------+
```

---

## 3. Component Hierarchy

```
src/
└── components/
    └── shell/
        ├── UnifiedShell.jsx          # Root container & Context Provider wrapper
        ├── Header/
        │   ├── SuiteHeader.jsx        # Topbar global header
        │   ├── AppLauncher.jsx       # 9-dot Waffle Menu drawer/modal
        │   ├── CommandPalette.jsx    # Quick action & search palette (Ctrl+K)
        │   ├── NotificationDrawer.jsx# Real-time notifications slide-over
        │   └── UserProfileMenu.jsx   # Profile menu & theme mode toggle
        ├── Navigation/
        │   ├── WorkspaceTabs.jsx     # Tab bar supporting open/close/switch
        │   └── ContextSidebar.jsx    # Collapsible sidebar updating per app context
        └── Canvas/
            └── WorkspaceCanvas.jsx   # Active tab viewport wrapper
```

---

## 4. State Management (`ShellContext`)

State maintained globally for the shell:

```typescript
interface TabItem {
  id: string;
  title: string;
  path: string;
  app: string;
  icon?: string;
  closable: boolean;
  state?: Record<string, any>;
}

interface ShellState {
  activeApp: 'banking' | 'admin' | 'investments' | 'credit' | 'compliance' | 'settings';
  openTabs: TabItem[];
  activeTabId: string;
  sidebarCollapsed: boolean;
  commandPaletteOpen: boolean;
  notificationDrawerOpen: boolean;
  themeMode: 'dark' | 'light';
}
```

### Context Actions:
- `switchApp(appId: string)`: Updates `activeApp` and opens/focuses default app tab.
- `openTab(tab: TabItem)`: Adds new tab if not present, then sets `activeTabId`.
- `closeTab(tabId: string)`: Removes tab and activates neighboring tab.
- `toggleSidebar()`: Toggles sidebar collapsed state.
- `toggleTheme()`: Toggles between `'dark'` and `'light'` mode.
- `setCommandPalette(open: boolean)`: Toggles command palette modal.

---

## 5. Application Suite Catalog & Routes

| App Key | App Title | Icon | Default Route | Included Views |
| :--- | :--- | :--- | :--- | :--- |
| `banking` | **AURIX Banking** | `AccountBalance` | `/dashboard` | Dashboard, Contas, Extrato, Transações, Transferência, Pix, Cartões, Pagamentos, Recarga |
| `admin` | **AURIX Admin** | `AdminPanelSettings` | `/admin/dashboard` | Painel Admin, Gestão de Clientes, Aprovações, Tarifas, Logs de Auditoria |
| `investments` | **Investimentos** | `TrendingUp` | `/investimentos` | Carteira de Investimentos, Renda Fixa, Renda Variável, Yields |
| `credit` | **Crédito & Empréstimos** | `CreditScore` | `/credito` | Simulação de Crédito, Consignado, Contratos, Margem Consignável |
| `compliance` | **Fraude & Risco** | `Security` | `/compliance/alertas` | Alertas de Risco, Monitoramento em Tempo Real, Regras de Antifraude |
| `settings` | **Configurações** | `Settings` | `/configuracoes` | Perfil, Segurança 2FA, Preferências do Shell, Onboarding |

---

## 6. Theme & Visual Aesthetics

- **Primary Colors:** Slate Dark (`#0F172A`), Midnight Navy (`#1E293B`).
- **Accent Color:** AURIX Gold (`#D4AF37` / `#C5A028`).
- **Header Glassmorphism:** `background: rgba(15, 23, 42, 0.85); backdrop-filter: blur(12px); border-bottom: 1px solid rgba(212, 175, 55, 0.2);`
- **Tab Bar Styling:** Elevated pill/card active tab indicator with golden accent bottom bar or border glow.
- **Typography:** Clean sans-serif hierarchy (Inter / Roboto) with custom scrollbars and MUI transitions.
