# AURIX Unified Shell Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement a Dynamics 365 style Enterprise Unified Shell for AURIX Platform (`frontend/aurix-web`), featuring a 9-dot Waffle App Launcher, workspace tabs, dynamic contextual sidebar, Ctrl+K command palette, and MUI 5 Fluent-inspired theme.

**Architecture:** A central `ShellContext` will manage tab history, active app context, and shell state. `UnifiedShell` will wrap the application layout, replacing the legacy static navbar/sidebar setup with a suite topbar (`SuiteHeader`), tab workspace (`WorkspaceTabs`), collapsible dynamic navigation (`ContextSidebar`), and active view viewport (`WorkspaceCanvas`).

**Tech Stack:** React 18, React Router v6, Material UI (MUI 5), `@mui/icons-material`, `framer-motion`, `date-fns`.

## Global Constraints
- Target repository directory: `frontend/aurix-web`
- MUI 5 components and icons must be used for UI widgets
- Theme must support Dark and Light modes with AURIX Gold (`#D4AF37`) accent
- Standard npm test command: `npm test -- --watchAll=false` from `frontend/aurix-web`

---

### Task 1: Shell State Management (`ShellContext`)

**Files:**
- Create: `frontend/aurix-web/src/context/ShellContext.jsx`
- Create: `frontend/aurix-web/src/context/ShellContext.test.jsx`

**Interfaces:**
- Produces: `ShellProvider`, `useShell` hook exposing `{ activeApp, openTabs, activeTabId, sidebarCollapsed, commandPaletteOpen, notificationDrawerOpen, themeMode, switchApp, openTab, closeTab, setActiveTabId, toggleSidebar, toggleTheme, setCommandPaletteOpen, setNotificationDrawerOpen }`

- [ ] **Step 1: Write the failing test**

```jsx
// frontend/aurix-web/src/context/ShellContext.test.jsx
import React from 'react';
import { render, screen, act } from '@testing-library/react';
import { ShellProvider, useShell } from './ShellContext';

const TestComponent = () => {
  const { activeApp, switchApp, openTabs, openTab, closeTab, activeTabId } = useShell();
  return (
    <div>
      <span data-testid="active-app">{activeApp}</span>
      <span data-testid="tab-count">{openTabs.length}</span>
      <span data-testid="active-tab">{activeTabId}</span>
      <button onClick={() => switchApp('admin')}>Switch App</button>
      <button onClick={() => openTab({ id: 'pix-1', title: 'Pix', path: '/pix', app: 'banking', closable: true })}>Open Pix Tab</button>
      <button onClick={() => closeTab('pix-1')}>Close Pix Tab</button>
    </div>
  );
};

describe('ShellContext', () => {
  test('provides default shell state and updates on actions', () => {
    render(
      <ShellProvider>
        <TestComponent />
      </ShellProvider>
    );

    expect(screen.getByTestId('active-app').textContent).toBe('banking');
    expect(screen.getByTestId('tab-count').textContent).toBe('1'); // Initial dashboard tab

    act(() => {
      screen.getByText('Switch App').click();
    });
    expect(screen.getByTestId('active-app').textContent).toBe('admin');

    act(() => {
      screen.getByText('Open Pix Tab').click();
    });
    expect(screen.getByTestId('tab-count').textContent).toBe('2');
    expect(screen.getByTestId('active-tab').textContent).toBe('pix-1');

    act(() => {
      screen.getByText('Close Pix Tab').click();
    });
    expect(screen.getByTestId('tab-count').textContent).toBe('1');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend/aurix-web && npm test -- --watchAll=false src/context/ShellContext.test.jsx`  
Expected: FAIL with "Cannot find module ./ShellContext"

- [ ] **Step 3: Write minimal implementation**

```jsx
// frontend/aurix-web/src/context/ShellContext.jsx
import React, { createContext, useContext, useState, useEffect } from 'react';

const ShellContext = createContext(null);

const DEFAULT_TABS = [
  { id: 'tab-dashboard', title: 'Dashboard', path: '/dashboard', app: 'banking', closable: false }
];

export const ShellProvider = ({ children }) => {
  const [activeApp, setActiveApp] = useState('banking');
  const [openTabs, setOpenTabs] = useState(DEFAULT_TABS);
  const [activeTabId, setActiveTabId] = useState('tab-dashboard');
  const [sidebarCollapsed, setSidebarCollapsed] = useState(false);
  const [commandPaletteOpen, setCommandPaletteOpen] = useState(false);
  const [notificationDrawerOpen, setNotificationDrawerOpen] = useState(false);
  const [themeMode, setThemeMode] = useState(() => localStorage.getItem('aurix_theme') || 'dark');

  useEffect(() => {
    localStorage.setItem('aurix_theme', themeMode);
  }, [themeMode]);

  const switchApp = (appId) => {
    setActiveApp(appId);
    // Find or open default tab for app
    const appTab = openTabs.find(t => t.app === appId);
    if (appTab) {
      setActiveTabId(appTab.id);
    }
  };

  const openTab = (tab) => {
    const existing = openTabs.find(t => t.id === tab.id || t.path === tab.path);
    if (existing) {
      setActiveTabId(existing.id);
    } else {
      const newTabs = [...openTabs, tab];
      setOpenTabs(newTabs);
      setActiveTabId(tab.id);
    }
  };

  const closeTab = (tabId) => {
    const tabToClose = openTabs.find(t => t.id === tabId);
    if (!tabToClose || !tabToClose.closable) return;

    const filtered = openTabs.filter(t => t.id !== tabId);
    setOpenTabs(filtered);

    if (activeTabId === tabId) {
      const lastTab = filtered[filtered.length - 1];
      if (lastTab) {
        setActiveTabId(lastTab.id);
        setActiveApp(lastTab.app);
      }
    }
  };

  const toggleSidebar = () => setSidebarCollapsed(prev => !prev);
  const toggleTheme = () => setThemeMode(prev => (prev === 'dark' ? 'light' : 'dark'));

  const value = {
    activeApp,
    openTabs,
    activeTabId,
    sidebarCollapsed,
    commandPaletteOpen,
    notificationDrawerOpen,
    themeMode,
    switchApp,
    openTab,
    closeTab,
    setActiveTabId,
    toggleSidebar,
    toggleTheme,
    setCommandPaletteOpen,
    setNotificationDrawerOpen
  };

  return <ShellContext.Provider value={value}>{children}</ShellContext.Provider>;
};

export const useShell = () => {
  const context = useContext(ShellContext);
  if (!context) {
    throw new Error('useShell must be used within a ShellProvider');
  }
  return context;
};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend/aurix-web && npm test -- --watchAll=false src/context/ShellContext.test.jsx`  
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add frontend/aurix-web/src/context/ShellContext.jsx frontend/aurix-web/src/context/ShellContext.test.jsx
git commit -m "feat: add ShellContext for Unified Shell state management"
```

---

### Task 2: Header Overlay Components (AppLauncher, CommandPalette, NotificationDrawer, UserProfileMenu)

**Files:**
- Create: `frontend/aurix-web/src/components/shell/Header/AppLauncher.jsx`
- Create: `frontend/aurix-web/src/components/shell/Header/CommandPalette.jsx`
- Create: `frontend/aurix-web/src/components/shell/Header/NotificationDrawer.jsx`
- Create: `frontend/aurix-web/src/components/shell/Header/UserProfileMenu.jsx`
- Create: `frontend/aurix-web/src/components/shell/Header/HeaderComponents.test.jsx`

- [ ] **Step 1: Write failing tests for Header Components**

```jsx
// frontend/aurix-web/src/components/shell/Header/HeaderComponents.test.jsx
import React from 'react';
import { render, screen } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { ShellProvider } from '../../../context/ShellContext';
import AppLauncher from './AppLauncher';
import CommandPalette from './CommandPalette';
import UserProfileMenu from './UserProfileMenu';

const renderWithProviders = (ui) => render(
  <BrowserRouter>
    <ShellProvider>
      {ui}
    </ShellProvider>
  </BrowserRouter>
);

describe('Header Overlay Components', () => {
  test('AppLauncher renders 9-dot app options', () => {
    renderWithProviders(<AppLauncher open={true} onClose={() => {}} />);
    expect(screen.getByText('AURIX Banking')).toBeInTheDocument();
    expect(screen.getByText('AURIX Admin')).toBeInTheDocument();
  });

  test('CommandPalette renders search input', () => {
    renderWithProviders(<CommandPalette open={true} onClose={() => {}} />);
    expect(screen.getByPlaceholderText(/Digite um comando ou busque/i)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend/aurix-web && npm test -- --watchAll=false src/components/shell/Header/HeaderComponents.test.jsx`  
Expected: FAIL

- [ ] **Step 3: Write minimal implementations**

```jsx
// frontend/aurix-web/src/components/shell/Header/AppLauncher.jsx
import React from 'react';
import { Drawer, Box, Typography, Grid, Card, CardActionArea, IconButton } from '@mui/material';
import CloseIcon from '@mui/icons-material/Close';
import AccountBalanceIcon from '@mui/icons-material/AccountBalance';
import AdminPanelSettingsIcon from '@mui/icons-material/AdminPanelSettings';
import TrendingUpIcon from '@mui/icons-material/TrendingUp';
import CreditScoreIcon from '@mui/icons-material/CreditScore';
import SecurityIcon from '@mui/icons-material/Security';
import SettingsIcon from '@mui/icons-material/Settings';
import { useShell } from '../../../context/ShellContext';

const APPS = [
  { id: 'banking', name: 'AURIX Banking', desc: 'Internet Banking & Contas', icon: <AccountBalanceIcon sx={{ fontSize: 36, color: '#D4AF37' }} />, path: '/dashboard' },
  { id: 'admin', name: 'AURIX Admin', desc: 'Gestão Corporativa', icon: <AdminPanelSettingsIcon sx={{ fontSize: 36, color: '#6366F1' }} />, path: '/admin/dashboard' },
  { id: 'investments', name: 'Investimentos', desc: 'Renda Fixa e Variável', icon: <TrendingUpIcon sx={{ fontSize: 36, color: '#10B981' }} />, path: '/investimentos' },
  { id: 'credit', name: 'Crédito & Empréstimos', desc: 'Simulações e Margem', icon: <CreditScoreIcon sx={{ fontSize: 36, color: '#F59E0B' }} />, path: '/credito' },
  { id: 'compliance', name: 'Fraude & Compliance', desc: 'Monitoramento de Risco', icon: <SecurityIcon sx={{ fontSize: 36, color: '#EF4444' }} />, path: '/compliance/alertas' },
  { id: 'settings', name: 'Configurações', desc: 'Preferências do Shell', icon: <SettingsIcon sx={{ fontSize: 36, color: '#94A3B8' }} />, path: '/configuracoes' },
];

export default function AppLauncher({ open, onClose }) {
  const { switchApp, openTab } = useShell();

  const handleSelectApp = (app) => {
    switchApp(app.id);
    openTab({ id: `tab-${app.id}`, title: app.name, path: app.path, app: app.id, closable: true });
    onClose();
  };

  return (
    <Drawer anchor="top" open={open} onClose={onClose} PaperProps={{ sx: { background: '#0F172A', color: '#FFF', p: 3, borderBottom: '1px solid rgba(212,175,55,0.3)' } }}>
      <Box display="flex" justifyContent="space-between" alignItems="center" mb={2}>
        <Typography variant="h6" fontWeight="bold" sx={{ color: '#D4AF37' }}>
          AURIX Suite — App Launcher
        </Typography>
        <IconButton onClick={onClose} sx={{ color: '#FFF' }}>
          <CloseIcon />
        </IconButton>
      </Box>
      <Grid container spacing={2}>
        {APPS.map((app) => (
          <Grid item xs={12} sm={6} md={4} key={app.id}>
            <Card sx={{ background: '#1E293B', color: '#FFF', border: '1px solid rgba(255,255,255,0.08)', '&:hover': { borderColor: '#D4AF37', transform: 'translateY(-2px)' }, transition: 'all 0.2s' }}>
              <CardActionArea onClick={() => handleSelectApp(app)} sx={{ p: 2, display: 'flex', alignItems: 'center', gap: 2 }}>
                {app.icon}
                <Box>
                  <Typography variant="subtitle1" fontWeight="bold">{app.name}</Typography>
                  <Typography variant="body2" color="gray">{app.desc}</Typography>
                </Box>
              </CardActionArea>
            </Card>
          </Grid>
        ))}
      </Grid>
    </Drawer>
  );
}
```

```jsx
// frontend/aurix-web/src/components/shell/Header/CommandPalette.jsx
import React, { useState } from 'react';
import { Dialog, DialogContent, TextField, List, ListItem, ListItemButton, ListItemText, ListItemIcon, Box, Typography } from '@mui/material';
import SearchIcon from '@mui/icons-material/Search';
import SendIcon from '@mui/icons-material/Send';
import AccountBalanceWalletIcon from '@mui/icons-material/AccountBalanceWallet';
import CreditCardIcon from '@mui/icons-material/CreditCard';
import { useNavigate } from 'react-router-dom';
import { useShell } from '../../../context/ShellContext';

const COMMANDS = [
  { title: 'Ir para Dashboard', category: 'Navegação', path: '/dashboard', icon: <SearchIcon />, app: 'banking' },
  { title: 'Realizar Transferência Pix', category: 'Ações Rápidas', path: '/pix', icon: <SendIcon />, app: 'banking' },
  { title: 'Ver Extrato Consolidado', category: 'Contas', path: '/extrato', icon: <AccountBalanceWalletIcon />, app: 'banking' },
  { title: 'Gestão de Cartões', category: 'Cartões', path: '/cartoes', icon: <CreditCardIcon />, app: 'banking' },
];

export default function CommandPalette({ open, onClose }) {
  const [query, setQuery] = useState('');
  const { openTab, switchApp } = useShell();
  const navigate = useNavigate();

  const filtered = COMMANDS.filter(cmd => cmd.title.toLowerCase().includes(query.toLowerCase()));

  const handleSelect = (cmd) => {
    switchApp(cmd.app);
    openTab({ id: `tab-${cmd.path.replace('/', '')}`, title: cmd.title, path: cmd.path, app: cmd.app, closable: true });
    navigate(cmd.path);
    onClose();
  };

  return (
    <Dialog open={open} onClose={onClose} fullWidth maxWidth="sm" PaperProps={{ sx: { background: '#1E293B', color: '#FFF' } }}>
      <DialogContent sx={{ p: 2 }}>
        <Box display="flex" alignItems="center" gap={1} mb={2}>
          <SearchIcon sx={{ color: '#D4AF37' }} />
          <TextField
            fullWidth
            placeholder="Digite um comando ou busque (Ctrl+K)..."
            variant="standard"
            value={query}
            onChange={(e) => setQuery(e.target.value)}
            InputProps={{ disableUnderline: true, sx: { color: '#FFF', fontSize: '1.1rem' } }}
            autoFocus
          />
        </Box>
        <List>
          {filtered.map((cmd, idx) => (
            <ListItem disablePadding key={idx}>
              <ListItemButton onClick={() => handleSelect(cmd)} sx={{ borderRadius: 1, '&:hover': { background: 'rgba(212,175,55,0.15)' } }}>
                <ListItemIcon sx={{ color: '#D4AF37' }}>{cmd.icon}</ListItemIcon>
                <ListItemText primary={cmd.title} secondary={cmd.category} secondaryTypographyProps={{ color: 'gray' }} />
              </ListItemButton>
            </ListItem>
          ))}
          {filtered.length === 0 && (
            <Typography variant="body2" color="gray" textAlign="center" py={2}>
              Nenhum resultado encontrado.
            </Typography>
          )}
        </List>
      </DialogContent>
    </Dialog>
  );
}
```

```jsx
// frontend/aurix-web/src/components/shell/Header/NotificationDrawer.jsx
import React from 'react';
import { Drawer, Box, Typography, IconButton, List, ListItem, ListItemText, Divider } from '@mui/material';
import CloseIcon from '@mui/icons-material/Close';
import NotificationsIcon from '@mui/icons-material/Notifications';

const MOCK_NOTIFS = [
  { id: 1, title: 'Pix Recebido', desc: 'Você recebeu R$ 1.500,00 de João Silva', time: 'Há 5 min' },
  { id: 2, title: 'Alerta de Segurança', desc: 'Novo login detectado no Chrome / Linux', time: 'Há 1 hora' },
];

export default function NotificationDrawer({ open, onClose }) {
  return (
    <Drawer anchor="right" open={open} onClose={onClose} PaperProps={{ sx: { width: 340, background: '#0F172A', color: '#FFF', p: 2 } }}>
      <Box display="flex" justifyContent="space-between" alignItems="center" mb={2}>
        <Box display="flex" alignItems="center" gap={1}>
          <NotificationsIcon sx={{ color: '#D4AF37' }} />
          <Typography variant="h6" fontWeight="bold">Notificações</Typography>
        </Box>
        <IconButton onClick={onClose} sx={{ color: '#FFF' }}><CloseIcon /></IconButton>
      </Box>
      <Divider sx={{ borderColor: 'rgba(255,255,255,0.1)' }} />
      <List>
        {MOCK_NOTIFS.map((n) => (
          <ListItem key={n.id} alignFlexStart sx={{ borderBottom: '1px solid rgba(255,255,255,0.05)' }}>
            <ListItemText primary={n.title} secondary={`${n.desc} • ${n.time}`} secondaryTypographyProps={{ color: 'gray' }} />
          </ListItem>
        ))}
      </List>
    </Drawer>
  );
}
```

```jsx
// frontend/aurix-web/src/components/shell/Header/UserProfileMenu.jsx
import React, { useState } from 'react';
import { Menu, MenuItem, ListItemIcon, ListItemText, Divider, Avatar, Box, Typography } from '@mui/material';
import DarkModeIcon from '@mui/icons-material/DarkMode';
import LightModeIcon from '@mui/icons-material/LightMode';
import LogoutIcon from '@mui/icons-material/Logout';
import PersonIcon from '@mui/icons-material/Person';
import { useShell } from '../../../context/ShellContext';

export default function UserProfileMenu({ anchorEl, open, onClose, user, onLogout }) {
  const { themeMode, toggleTheme } = useShell();

  return (
    <Menu
      anchorEl={anchorEl}
      open={open}
      onClose={onClose}
      PaperProps={{ sx: { background: '#1E293B', color: '#FFF', minWidth: 220, border: '1px solid rgba(212,175,55,0.2)' } }}
    >
      <Box px={2} py={1.5}>
        <Typography variant="subtitle1" fontWeight="bold" sx={{ color: '#D4AF37' }}>
          {user?.nome || 'Usuário AURIX'}
        </Typography>
        <Typography variant="caption" color="gray">
          {user?.email || 'usuario@aurix.com.br'}
        </Typography>
      </Box>
      <Divider sx={{ borderColor: 'rgba(255,255,255,0.1)' }} />
      <MenuItem onClick={toggleTheme}>
        <ListItemIcon sx={{ color: '#D4AF37' }}>
          {themeMode === 'dark' ? <LightModeIcon /> : <DarkModeIcon />}
        </ListItemIcon>
        <ListItemText primary={themeMode === 'dark' ? 'Modo Claro' : 'Modo Escuro'} />
      </MenuItem>
      <MenuItem onClick={onClose}>
        <ListItemIcon sx={{ color: '#D4AF37' }}><PersonIcon /></ListItemIcon>
        <ListItemText primary="Meu Perfil" />
      </MenuItem>
      <Divider sx={{ borderColor: 'rgba(255,255,255,0.1)' }} />
      <MenuItem onClick={onLogout}>
        <ListItemIcon sx={{ color: '#EF4444' }}><LogoutIcon /></ListItemIcon>
        <ListItemText primary="Sair da Conta" sx={{ color: '#EF4444' }} />
      </MenuItem>
    </Menu>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend/aurix-web && npm test -- --watchAll=false src/components/shell/Header/HeaderComponents.test.jsx`  
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add frontend/aurix-web/src/components/shell/Header/
git commit -m "feat: add Header Overlay components (AppLauncher, CommandPalette, NotificationDrawer, UserProfileMenu)"
```

---

### Task 3: Suite Header & Workspace Tabs Bar

**Files:**
- Create: `frontend/aurix-web/src/components/shell/Header/SuiteHeader.jsx`
- Create: `frontend/aurix-web/src/components/shell/Navigation/WorkspaceTabs.jsx`
- Create: `frontend/aurix-web/src/components/shell/Navigation/SuiteHeaderAndTabs.test.jsx`

- [ ] **Step 1: Write failing tests for SuiteHeader and WorkspaceTabs**

```jsx
// frontend/aurix-web/src/components/shell/Navigation/SuiteHeaderAndTabs.test.jsx
import React from 'react';
import { render, screen } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { ShellProvider } from '../../../context/ShellContext';
import SuiteHeader from '../Header/SuiteHeader';
import WorkspaceTabs from './WorkspaceTabs';

describe('SuiteHeader and WorkspaceTabs', () => {
  test('SuiteHeader renders 9-dot waffle icon and title', () => {
    render(
      <BrowserRouter>
        <ShellProvider>
          <SuiteHeader user={{ nome: 'Test' }} onLogout={() => {}} />
        </ShellProvider>
      </BrowserRouter>
    );

    expect(screen.getByText(/AURIX Suite/i)).toBeInTheDocument();
  });

  test('WorkspaceTabs renders open tabs', () => {
    render(
      <BrowserRouter>
        <ShellProvider>
          <WorkspaceTabs />
        </ShellProvider>
      </BrowserRouter>
    );

    expect(screen.getByText('Dashboard')).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend/aurix-web && npm test -- --watchAll=false src/components/shell/Navigation/SuiteHeaderAndTabs.test.jsx`  
Expected: FAIL

- [ ] **Step 3: Write minimal implementations**

```jsx
// frontend/aurix-web/src/components/shell/Header/SuiteHeader.jsx
import React, { useState } from 'react';
import { AppBar, Toolbar, IconButton, Typography, Box, Button, Avatar, Badge } from '@mui/material';
import AppsIcon from '@mui/icons-material/Apps';
import SearchIcon from '@mui/icons-material/Search';
import NotificationsIcon from '@mui/icons-material/Notifications';
import MenuIcon from '@mui/icons-material/Menu';
import { useShell } from '../../../context/ShellContext';
import AppLauncher from './AppLauncher';
import CommandPalette from './CommandPalette';
import NotificationDrawer from './NotificationDrawer';
import UserProfileMenu from './UserProfileMenu';

export default function SuiteHeader({ user, onLogout }) {
  const { toggleSidebar, activeApp } = useShell();
  const [launcherOpen, setLauncherOpen] = useState(false);
  const [paletteOpen, setPaletteOpen] = useState(false);
  const [notifOpen, setNotifOpen] = useState(false);
  const [userMenuAnchor, setUserMenuAnchor] = useState(null);

  const getAppName = () => {
    switch (activeApp) {
      case 'admin': return 'AURIX Admin Portal';
      case 'investments': return 'AURIX Investimentos';
      case 'credit': return 'AURIX Crédito';
      case 'compliance': return 'AURIX Compliance & Risco';
      default: return 'AURIX Internet Banking';
    }
  };

  return (
    <>
      <AppBar position="fixed" sx={{ zIndex: (theme) => theme.zIndex.drawer + 1, background: '#0F172A', backdropFilter: 'blur(12px)', borderBottom: '1px solid rgba(212,175,55,0.2)' }}>
        <Toolbar variant="dense" sx={{ justifyContent: 'space-between' }}>
          <Box display="flex" alignItems="center" gap={1}>
            <IconButton color="inherit" onClick={() => setLauncherOpen(true)} title="App Launcher (Waffle)">
              <AppsIcon sx={{ color: '#D4AF37', fontSize: 28 }} />
            </IconButton>
            <IconButton color="inherit" onClick={toggleSidebar}>
              <MenuIcon />
            </IconButton>
            <Typography variant="h6" fontWeight="bold" sx={{ color: '#FFF', display: 'flex', alignItems: 'center', gap: 1 }}>
              <span style={{ color: '#D4AF37' }}>AURIX Suite</span>
              <Typography variant="caption" sx={{ background: 'rgba(212,175,55,0.15)', color: '#D4AF37', px: 1, py: 0.3, borderRadius: 1, fontWeight: 'bold' }}>
                {getAppName()}
              </Typography>
            </Typography>
          </Box>

          <Box display="flex" alignItems="center" gap={1.5}>
            <Button
              variant="outlined"
              startIcon={<SearchIcon sx={{ color: '#D4AF37' }} />}
              onClick={() => setPaletteOpen(true)}
              sx={{ color: '#AAA', borderColor: 'rgba(255,255,255,0.2)', textTransform: 'none', borderRadius: 2, px: 2, display: { xs: 'none', sm: 'flex' } }}
            >
              Buscar comando (Ctrl+K)...
            </Button>
            <IconButton color="inherit" onClick={() => setNotifOpen(true)}>
              <Badge badgeContent={2} color="error">
                <NotificationsIcon />
              </Badge>
            </IconButton>
            <IconButton onClick={(e) => setUserMenuAnchor(e.currentTarget)} sx={{ p: 0.5 }}>
              <Avatar sx={{ bgcolor: '#D4AF37', color: '#0F172A', fontWeight: 'bold', width: 32, height: 32 }}>
                {user?.nome ? user.nome[0] : 'A'}
              </Avatar>
            </IconButton>
          </Box>
        </Toolbar>
      </AppBar>

      <AppLauncher open={launcherOpen} onClose={() => setLauncherOpen(false)} />
      <CommandPalette open={paletteOpen} onClose={() => setPaletteOpen(false)} />
      <NotificationDrawer open={notifOpen} onClose={() => setNotifOpen(false)} />
      <UserProfileMenu anchorEl={userMenuAnchor} open={Boolean(userMenuAnchor)} onClose={() => setUserMenuAnchor(null)} user={user} onLogout={onLogout} />
    </>
  );
}
```

```jsx
// frontend/aurix-web/src/components/shell/Navigation/WorkspaceTabs.jsx
import React from 'react';
import { Box, Tabs, Tab, IconButton } from '@mui/material';
import CloseIcon from '@mui/icons-material/Close';
import AddIcon from '@mui/icons-material/Add';
import { useShell } from '../../../context/ShellContext';
import { useNavigate } from 'react-router-dom';

export default function WorkspaceTabs() {
  const { openTabs, activeTabId, setActiveTabId, closeTab, setCommandPaletteOpen } = useShell();
  const navigate = useNavigate();

  const handleTabChange = (event, tabId) => {
    setActiveTabId(tabId);
    const tab = openTabs.find(t => t.id === tabId);
    if (tab) navigate(tab.path);
  };

  return (
    <Box sx={{ background: '#1E293B', borderBottom: '1px solid rgba(255,255,255,0.08)', display: 'flex', alignItems: 'center', px: 1 }}>
      <Tabs
        value={activeTabId}
        onChange={handleTabChange}
        variant="scrollable"
        scrollButtons="auto"
        sx={{
          minHeight: 40,
          '& .MuiTab-root': {
            minHeight: 40,
            textTransform: 'none',
            color: '#94A3B8',
            fontSize: '0.85rem',
            fontWeight: 500,
            '&.Mui-selected': { color: '#D4AF37', background: 'rgba(212,175,55,0.08)', fontWeight: 'bold' }
          },
          '& .MuiTabs-indicator': { backgroundColor: '#D4AF37' }
        }}
      >
        {openTabs.map((tab) => (
          <Tab
            key={tab.id}
            value={tab.id}
            label={
              <Box display="flex" alignItems="center" gap={1}>
                <span>{tab.title}</span>
                {tab.closable && (
                  <IconButton
                    size="small"
                    component="span"
                    onClick={(e) => { e.stopPropagation(); closeTab(tab.id); }}
                    sx={{ color: 'inherit', p: 0.2, '&:hover': { color: '#EF4444' } }}
                  >
                    <CloseIcon sx={{ fontSize: 14 }} />
                  </IconButton>
                )}
              </Box>
            }
          />
        ))}
      </Tabs>
      <IconButton size="small" onClick={() => setCommandPaletteOpen(true)} sx={{ color: '#D4AF37', ml: 1 }} title="Abrir novo comando">
        <AddIcon fontSize="small" />
      </IconButton>
    </Box>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend/aurix-web && npm test -- --watchAll=false src/components/shell/Navigation/SuiteHeaderAndTabs.test.jsx`  
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add frontend/aurix-web/src/components/shell/Header/SuiteHeader.jsx frontend/aurix-web/src/components/shell/Navigation/WorkspaceTabs.jsx frontend/aurix-web/src/components/shell/Navigation/SuiteHeaderAndTabs.test.jsx
git commit -m "feat: add SuiteHeader topbar and WorkspaceTabs bar"
```

---

### Task 4: Dynamic Contextual Sidebar & Canvas Component

**Files:**
- Create: `frontend/aurix-web/src/components/shell/Navigation/ContextSidebar.jsx`
- Create: `frontend/aurix-web/src/components/shell/Canvas/WorkspaceCanvas.jsx`
- Create: `frontend/aurix-web/src/components/shell/Navigation/ContextSidebar.test.jsx`

- [ ] **Step 1: Write failing test for ContextSidebar**

```jsx
// frontend/aurix-web/src/components/shell/Navigation/ContextSidebar.test.jsx
import React from 'react';
import { render, screen } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { ShellProvider } from '../../../context/ShellContext';
import ContextSidebar from './ContextSidebar';

describe('ContextSidebar', () => {
  test('renders menu items for banking app context', () => {
    render(
      <BrowserRouter>
        <ShellProvider>
          <ContextSidebar />
        </ShellProvider>
      </BrowserRouter>
    );

    expect(screen.getByText('Inicio')).toBeInTheDocument();
    expect(screen.getByText('Área Pix')).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend/aurix-web && npm test -- --watchAll=false src/components/shell/Navigation/ContextSidebar.test.jsx`  
Expected: FAIL

- [ ] **Step 3: Write minimal implementations**

```jsx
// frontend/aurix-web/src/components/shell/Navigation/ContextSidebar.jsx
import React from 'react';
import { Drawer, List, ListItem, ListItemButton, ListItemIcon, ListItemText, Box, Tooltip } from '@mui/material';
import DashboardIcon from '@mui/icons-material/Dashboard';
import AccountBalanceIcon from '@mui/icons-material/AccountBalance';
import ReceiptLongIcon from '@mui/icons-material/ReceiptLong';
import SendIcon from '@mui/icons-material/Send';
import CreditCardIcon from '@mui/icons-material/CreditCard';
import TrendingUpIcon from '@mui/icons-material/TrendingUp';
import SettingsIcon from '@mui/icons-material/Settings';
import PeopleIcon from '@mui/icons-material/People';
import SecurityIcon from '@mui/icons-material/Security';
import { useNavigate, useLocation } from 'react-router-dom';
import { useShell } from '../../../context/ShellContext';

const NAV_ITEMS = {
  banking: [
    { title: 'Inicio', path: '/dashboard', icon: <DashboardIcon /> },
    { title: 'Contas & Saldos', path: '/contas', icon: <AccountBalanceIcon /> },
    { title: 'Extrato Bancário', path: '/extrato', icon: <ReceiptLongIcon /> },
    { title: 'Área Pix', path: '/pix', icon: <SendIcon /> },
    { title: 'Cartões', path: '/cartoes', icon: <CreditCardIcon /> },
    { title: 'Investimentos', path: '/investimentos', icon: <TrendingUpIcon /> },
  ],
  admin: [
    { title: 'Visão Geral Admin', path: '/dashboard', icon: <DashboardIcon /> },
    { title: 'Gestão de Clientes', path: '/contas', icon: <PeopleIcon /> },
    { title: 'Auditoria de Transações', path: '/transacoes', icon: <ReceiptLongIcon /> },
  ],
  compliance: [
    { title: 'Alertas de Risco', path: '/dashboard', icon: <SecurityIcon /> },
  ],
  settings: [
    { title: 'Perfil', path: '/perfil', icon: <SettingsIcon /> },
    { title: 'Configurações', path: '/configuracoes', icon: <SettingsIcon /> },
  ]
};

export default function ContextSidebar() {
  const { activeApp, sidebarCollapsed, openTab } = useShell();
  const navigate = useNavigate();
  const location = useLocation();

  const items = NAV_ITEMS[activeApp] || NAV_ITEMS.banking;

  const handleNavigate = (item) => {
    openTab({ id: `tab-${item.path.replace('/', '')}`, title: item.title, path: item.path, app: activeApp, closable: true });
    navigate(item.path);
  };

  return (
    <Drawer
      variant="permanent"
      sx={{
        width: sidebarCollapsed ? 64 : 240,
        flexShrink: 0,
        '& .MuiDrawer-paper': {
          width: sidebarCollapsed ? 64 : 240,
          boxSizing: 'border-box',
          top: 48, // Header height offset
          height: 'calc(100% - 48px)',
          background: '#0F172A',
          color: '#FFF',
          borderRight: '1px solid rgba(255,255,255,0.08)',
          transition: 'width 0.2s'
        },
      }}
    >
      <Box sx={{ overflow: 'auto', mt: 1 }}>
        <List>
          {items.map((item, idx) => {
            const isSelected = location.pathname === item.path;
            return (
              <ListItem key={idx} disablePadding sx={{ display: 'block' }}>
                <Tooltip title={sidebarCollapsed ? item.title : ''} placement="right">
                  <ListItemButton
                    onClick={() => handleNavigate(item)}
                    selected={isSelected}
                    sx={{
                      minHeight: 44,
                      justifyContent: sidebarCollapsed ? 'center' : 'initial',
                      px: 2.5,
                      '&.Mui-selected': { background: 'rgba(212,175,55,0.15)', borderLeft: '3px solid #D4AF37' },
                      '&:hover': { background: 'rgba(255,255,255,0.05)' }
                    }}
                  >
                    <ListItemIcon sx={{ minWidth: 0, mr: sidebarCollapsed ? 'auto' : 2, justifyContent: 'center', color: isSelected ? '#D4AF37' : '#94A3B8' }}>
                      {item.icon}
                    </ListItemIcon>
                    {!sidebarCollapsed && <ListItemText primary={item.title} primaryTypographyProps={{ fontSize: '0.9rem', color: isSelected ? '#D4AF37' : '#FFF' }} />}
                  </ListItemButton>
                </Tooltip>
              </ListItem>
            );
          })}
        </List>
      </Box>
    </Drawer>
  );
}
```

```jsx
// frontend/aurix-web/src/components/shell/Canvas/WorkspaceCanvas.jsx
import React from 'react';
import { Box } from '@mui/material';

export default function WorkspaceCanvas({ children }) {
  return (
    <Box component="main" sx={{ flexGrow: 1, p: 3, background: '#090D16', minHeight: 'calc(100vh - 88px)', color: '#FFF' }}>
      {children}
    </Box>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend/aurix-web && npm test -- --watchAll=false src/components/shell/Navigation/ContextSidebar.test.jsx`  
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add frontend/aurix-web/src/components/shell/Navigation/ContextSidebar.jsx frontend/aurix-web/src/components/shell/Canvas/WorkspaceCanvas.jsx frontend/aurix-web/src/components/shell/Navigation/ContextSidebar.test.jsx
git commit -m "feat: add ContextSidebar and WorkspaceCanvas layout components"
```

---

### Task 5: Assemble `UnifiedShell` and Integrate into `App.js`

**Files:**
- Create: `frontend/aurix-web/src/components/shell/UnifiedShell.jsx`
- Create: `frontend/aurix-web/src/components/shell/UnifiedShell.test.jsx`
- Modify: `frontend/aurix-web/src/App.js`

- [ ] **Step 1: Write failing test for UnifiedShell**

```jsx
// frontend/aurix-web/src/components/shell/UnifiedShell.test.jsx
import React from 'react';
import { render, screen } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import UnifiedShell from './UnifiedShell';

describe('UnifiedShell Component', () => {
  test('renders SuiteHeader, WorkspaceTabs, and children within ShellProvider', () => {
    render(
      <BrowserRouter>
        <UnifiedShell user={{ nome: 'User' }} onLogout={() => {}}>
          <div data-testid="child-content">Child Page</div>
        </UnifiedShell>
      </BrowserRouter>
    );

    expect(screen.getByTestId('child-content')).toBeInTheDocument();
    expect(screen.getByText(/AURIX Suite/i)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend/aurix-web && npm test -- --watchAll=false src/components/shell/UnifiedShell.test.jsx`  
Expected: FAIL

- [ ] **Step 3: Write minimal implementation of `UnifiedShell.jsx` and update `App.js`**

```jsx
// frontend/aurix-web/src/components/shell/UnifiedShell.jsx
import React from 'react';
import { Box, ThemeProvider, createTheme, CssBaseline } from '@mui/material';
import { ShellProvider, useShell } from '../../context/ShellContext';
import SuiteHeader from './Header/SuiteHeader';
import WorkspaceTabs from './Navigation/WorkspaceTabs';
import ContextSidebar from './Navigation/ContextSidebar';
import WorkspaceCanvas from './Canvas/WorkspaceCanvas';

function ShellContent({ children, user, onLogout }) {
  const { themeMode } = useShell();

  const theme = createTheme({
    palette: {
      mode: themeMode,
      primary: { main: '#D4AF37' },
      secondary: { main: '#0F172A' },
      background: {
        default: themeMode === 'dark' ? '#090D16' : '#F8FAFC',
        paper: themeMode === 'dark' ? '#1E293B' : '#FFFFFF',
      },
    },
  });

  return (
    <ThemeProvider theme={theme}>
      <CssBaseline />
      <Box sx={{ display: 'flex', flexDirection: 'column', minHeight: '100vh' }}>
        <SuiteHeader user={user} onLogout={onLogout} />
        <Box sx={{ display: 'flex', flexGrow: 1, pt: '48px' }}>
          <ContextSidebar />
          <Box sx={{ display: 'flex', flexDirection: 'column', flexGrow: 1 }}>
            <WorkspaceTabs />
            <WorkspaceCanvas>{children}</WorkspaceCanvas>
          </Box>
        </Box>
      </Box>
    </ThemeProvider>
  );
}

export default function UnifiedShell({ children, user, onLogout }) {
  return (
    <ShellProvider>
      <ShellContent user={user} onLogout={onLogout}>
        {children}
      </ShellContent>
    </ShellProvider>
  );
}
```

Now update `App.js` to wrap authenticated routes inside `UnifiedShell`:

```jsx
// Replace return block in App.js when authenticated:
return (
  <LocalizationProvider dateAdapter={AdapterDateFns} adapterLocale={ptBR}>
    <UnifiedShell user={user} onLogout={handleLogout}>
      <Routes>
        <Route path="/" element={<Navigate to="/dashboard" replace />} />
        <Route path="/dashboard" element={<Dashboard user={user} />} />
        <Route path="/contas" element={<Contas user={user} />} />
        <Route path="/transacoes" element={<Transacoes user={user} />} />
        <Route path="/pix" element={<PIX user={user} />} />
        <Route path="/investimentos" element={<Investimentos user={user} />} />
        <Route path="/cartoes" element={<Cartoes user={user} />} />
        <Route path="/extrato" element={<Extrato user={user} />} />
        <Route path="/perfil" element={<Perfil user={user} />} />
        <Route path="/configuracoes" element={<Configuracoes user={user} />} />
        <Route path="/onboarding" element={<Onboarding user={user} />} />
        <Route path="/credito" element={<Credito user={user} />} />
        <Route path="/transferencia" element={<Transferencia user={user} />} />
        <Route path="/pagamento" element={<Pagamento user={user} />} />
        <Route path="/recarga" element={<Recarga user={user} />} />
      </Routes>
    </UnifiedShell>
  </LocalizationProvider>
);
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend/aurix-web && npm test -- --watchAll=false`  
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git add frontend/aurix-web/src/components/shell/UnifiedShell.jsx frontend/aurix-web/src/components/shell/UnifiedShell.test.jsx frontend/aurix-web/src/App.js
git commit -m "feat: assemble UnifiedShell and integrate into App.js main layout"
```

---

### Task 6: Full Verification and Build Validation

- [ ] **Step 1: Run frontend build**

Run: `cd frontend/aurix-web && npm run build`  
Expected: Clean production build with no missing dependencies or lint errors.

- [ ] **Step 2: Commit final shell integration**

```bash
git commit -am "chore: verify build and complete Unified Shell implementation"
```
