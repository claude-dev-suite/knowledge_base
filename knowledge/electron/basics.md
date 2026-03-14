# Electron Basics

## Overview

Electron is a framework for building cross-platform desktop applications using JavaScript, HTML, and CSS. It embeds Chromium (for rendering) and Node.js (for backend operations) into a single runtime.

## Architecture

### Process Model

Electron applications run in two types of processes:

```
┌─────────────────────────────────────────────────────────┐
│                    MAIN PROCESS                         │
│                                                         │
│  • Single instance per application                      │
│  • Full Node.js environment                             │
│  • Manages application lifecycle                        │
│  • Creates and manages BrowserWindow instances          │
│  • Access to native OS APIs                             │
│  • Handles IPC from renderers                           │
│                                                         │
│  Modules: app, BrowserWindow, ipcMain, dialog,          │
│           Menu, Tray, nativeTheme, autoUpdater          │
└─────────────────────────────────────────────────────────┘
                           │
                           │ Creates
                           ▼
┌─────────────────────────────────────────────────────────┐
│                  RENDERER PROCESS                       │
│                                                         │
│  • One per BrowserWindow                                │
│  • Chromium web page environment                        │
│  • Standard Web APIs                                    │
│  • No direct Node.js access (by default)                │
│  • Communicates via IPC through preload                 │
│                                                         │
│  Can use: React, Vue, Angular, Svelte, vanilla JS       │
└─────────────────────────────────────────────────────────┘
```

### Preload Scripts

Preload scripts run in a separate context before the renderer loads. They can access Node.js APIs and expose safe functionality to the renderer via `contextBridge`.

```typescript
// preload.ts
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('electronAPI', {
  // Expose only what's needed
  getVersion: () => ipcRenderer.invoke('app:getVersion'),
  openFile: () => ipcRenderer.invoke('dialog:openFile'),
});
```

## Basic Application Setup

### Main Process Entry

```typescript
// main.ts
import { app, BrowserWindow } from 'electron';
import path from 'path';

let mainWindow: BrowserWindow | null = null;

function createWindow(): void {
  mainWindow = new BrowserWindow({
    width: 1200,
    height: 800,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false,
    },
  });

  // Load app
  if (process.env.NODE_ENV === 'development') {
    mainWindow.loadURL('http://localhost:5173');
    mainWindow.webContents.openDevTools();
  } else {
    mainWindow.loadFile(path.join(__dirname, '../renderer/index.html'));
  }

  mainWindow.on('closed', () => {
    mainWindow = null;
  });
}

// Lifecycle events
app.whenReady().then(() => {
  createWindow();

  app.on('activate', () => {
    // macOS: re-create window when dock icon clicked
    if (BrowserWindow.getAllWindows().length === 0) {
      createWindow();
    }
  });
});

app.on('window-all-closed', () => {
  // Quit on all platforms except macOS
  if (process.platform !== 'darwin') {
    app.quit();
  }
});
```

### Package.json Configuration

```json
{
  "name": "my-electron-app",
  "version": "1.0.0",
  "main": "dist/main/index.js",
  "scripts": {
    "dev": "vite",
    "build": "vite build && electron-builder",
    "start": "electron ."
  },
  "devDependencies": {
    "electron": "^30.0.0",
    "electron-builder": "^24.0.0",
    "vite": "^5.0.0",
    "vite-plugin-electron": "^0.28.0"
  }
}
```

## BrowserWindow Options

### Common Configuration

```typescript
const mainWindow = new BrowserWindow({
  // Size
  width: 1200,
  height: 800,
  minWidth: 800,
  minHeight: 600,

  // Position
  center: true,
  // x: 100, y: 100,

  // Frame
  frame: true,                    // Show native frame
  titleBarStyle: 'hiddenInset',   // macOS: hide title, show traffic lights
  trafficLightPosition: { x: 15, y: 10 },

  // Behavior
  show: false,                    // Show when ready
  resizable: true,
  movable: true,
  minimizable: true,
  maximizable: true,
  closable: true,
  fullscreenable: true,

  // WebPreferences (SECURITY)
  webPreferences: {
    preload: path.join(__dirname, 'preload.js'),
    contextIsolation: true,       // Required for security
    nodeIntegration: false,       // Required for security
    sandbox: true,                // Additional isolation
    webSecurity: true,            // Same-origin policy
  },

  // Appearance
  backgroundColor: '#ffffff',
  icon: path.join(__dirname, 'icon.png'),
  transparent: false,
});

// Show when ready to prevent visual flash
mainWindow.once('ready-to-show', () => {
  mainWindow.show();
});
```

## App Module

### Lifecycle Events

```typescript
import { app } from 'electron';

// App is ready to create windows
app.whenReady().then(() => {
  createWindow();
});

// All windows closed
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});

// App activated (macOS dock click)
app.on('activate', () => {
  if (BrowserWindow.getAllWindows().length === 0) {
    createWindow();
  }
});

// Before quit
app.on('before-quit', () => {
  // Cleanup
});

// Will quit
app.on('will-quit', () => {
  // Final cleanup
});

// Second instance launched
app.on('second-instance', (event, argv, workingDirectory) => {
  // Focus existing window
  if (mainWindow) {
    if (mainWindow.isMinimized()) mainWindow.restore();
    mainWindow.focus();
  }
});
```

### Useful App Methods

```typescript
// Paths
app.getPath('userData');      // User data directory
app.getPath('documents');     // Documents folder
app.getPath('downloads');     // Downloads folder
app.getPath('temp');          // Temp directory

// App info
app.getName();
app.getVersion();
app.getLocale();

// Single instance lock
const gotLock = app.requestSingleInstanceLock();
if (!gotLock) {
  app.quit();
}

// Quit
app.quit();
app.exit(0);
```

## Project Structure

```
electron-app/
├── src/
│   ├── main/                    # Main process code
│   │   ├── index.ts             # Entry point
│   │   ├── window.ts            # Window management
│   │   ├── menu.ts              # Application menu
│   │   └── ipc/                 # IPC handlers
│   ├── preload/                 # Preload scripts
│   │   ├── index.ts
│   │   └── types.d.ts           # Type declarations
│   └── renderer/                # Frontend (React/Vue/etc)
│       ├── index.html
│       ├── main.tsx
│       └── App.tsx
├── resources/                   # Static assets
│   ├── icon.icns                # macOS icon
│   ├── icon.ico                 # Windows icon
│   └── icon.png                 # Linux icon
├── electron-builder.yml         # Build configuration
├── vite.config.ts               # Vite configuration
├── tsconfig.json
└── package.json
```

## Development Workflow

### With Vite

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import electron from 'vite-plugin-electron';
import renderer from 'vite-plugin-electron-renderer';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react(),
    electron([
      {
        entry: 'src/main/index.ts',
      },
      {
        entry: 'src/preload/index.ts',
        onstart(options) {
          options.reload();
        },
      },
    ]),
    renderer(),
  ],
});
```

### Hot Reload

```bash
# Development with hot reload
npm run dev

# Production build
npm run build

# Start production build
npm run start
```

## Debugging

### Main Process

```json
// launch.json (VS Code)
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Main Process",
      "type": "node",
      "request": "launch",
      "cwd": "${workspaceFolder}",
      "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron",
      "args": ["."],
      "outputCapture": "std"
    }
  ]
}
```

### Renderer Process

```typescript
// Open DevTools
mainWindow.webContents.openDevTools();

// Or from menu
Menu.setApplicationMenu(Menu.buildFromTemplate([
  {
    label: 'View',
    submenu: [
      { role: 'toggleDevTools' },
      { role: 'reload' },
    ],
  },
]));
```

## Reference

- [Electron Documentation](https://www.electronjs.org/docs/latest/)
- [Process Model](https://www.electronjs.org/docs/latest/tutorial/process-model)
- [Quick Start](https://www.electronjs.org/docs/latest/tutorial/quick-start)
