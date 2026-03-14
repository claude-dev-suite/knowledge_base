# Electron IPC Communication

## Overview

Inter-Process Communication (IPC) in Electron enables the main process and renderer processes to exchange messages through channels. This is essential because renderer processes cannot directly access Node.js APIs for security reasons.

## IPC Modules

| Module | Process | Purpose |
|--------|---------|---------|
| `ipcMain` | Main | Receive messages from renderers |
| `ipcRenderer` | Preload/Renderer | Send messages to main |
| `contextBridge` | Preload | Safely expose APIs to renderer |

## Communication Patterns

### Pattern 1: Renderer → Main (Two-Way)

The recommended pattern using `invoke`/`handle` for request-response communication.

```typescript
// preload.ts
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('electronAPI', {
  openFile: () => ipcRenderer.invoke('dialog:openFile'),
  readFile: (path: string) => ipcRenderer.invoke('file:read', path),
  saveFile: (path: string, content: string) =>
    ipcRenderer.invoke('file:save', path, content),
});

// main.ts
import { ipcMain, dialog } from 'electron';
import fs from 'fs/promises';

ipcMain.handle('dialog:openFile', async () => {
  const result = await dialog.showOpenDialog({
    properties: ['openFile'],
    filters: [
      { name: 'Text Files', extensions: ['txt', 'md'] },
      { name: 'All Files', extensions: ['*'] },
    ],
  });

  if (result.canceled) return null;
  return result.filePaths[0];
});

ipcMain.handle('file:read', async (_event, path: string) => {
  // Validate path to prevent directory traversal
  if (!isValidPath(path)) throw new Error('Invalid path');
  return fs.readFile(path, 'utf-8');
});

ipcMain.handle('file:save', async (_event, path: string, content: string) => {
  if (!isValidPath(path)) throw new Error('Invalid path');
  await fs.writeFile(path, content, 'utf-8');
  return true;
});

// renderer (React)
async function handleOpenFile() {
  const filePath = await window.electronAPI.openFile();
  if (filePath) {
    const content = await window.electronAPI.readFile(filePath);
    setFileContent(content);
  }
}
```

### Pattern 2: Renderer → Main (One-Way)

Use `send`/`on` for fire-and-forget messages.

```typescript
// preload.ts
contextBridge.exposeInMainWorld('electronAPI', {
  minimize: () => ipcRenderer.send('window:minimize'),
  maximize: () => ipcRenderer.send('window:maximize'),
  close: () => ipcRenderer.send('window:close'),
  setTitle: (title: string) => ipcRenderer.send('window:setTitle', title),
});

// main.ts
ipcMain.on('window:minimize', (event) => {
  const win = BrowserWindow.fromWebContents(event.sender);
  win?.minimize();
});

ipcMain.on('window:maximize', (event) => {
  const win = BrowserWindow.fromWebContents(event.sender);
  if (win?.isMaximized()) {
    win.unmaximize();
  } else {
    win?.maximize();
  }
});

ipcMain.on('window:close', (event) => {
  const win = BrowserWindow.fromWebContents(event.sender);
  win?.close();
});

ipcMain.on('window:setTitle', (event, title: string) => {
  const win = BrowserWindow.fromWebContents(event.sender);
  win?.setTitle(title);
});
```

### Pattern 3: Main → Renderer (Push)

Send messages from main to specific renderers.

```typescript
// main.ts
function sendToRenderer(win: BrowserWindow, channel: string, ...args: unknown[]) {
  win.webContents.send(channel, ...args);
}

// Example: Menu action
const menu = Menu.buildFromTemplate([
  {
    label: 'File',
    submenu: [
      {
        label: 'New',
        accelerator: 'CmdOrCtrl+N',
        click: () => {
          sendToRenderer(mainWindow!, 'menu:newFile');
        },
      },
      {
        label: 'Save',
        accelerator: 'CmdOrCtrl+S',
        click: () => {
          sendToRenderer(mainWindow!, 'menu:save');
        },
      },
    ],
  },
]);

// Example: Push notifications
function notifyRenderer(message: string) {
  mainWindow?.webContents.send('notification', { message, timestamp: Date.now() });
}

// preload.ts
contextBridge.exposeInMainWorld('electronEvents', {
  onNewFile: (callback: () => void) => {
    const handler = () => callback();
    ipcRenderer.on('menu:newFile', handler);
    return () => ipcRenderer.removeListener('menu:newFile', handler);
  },
  onSave: (callback: () => void) => {
    const handler = () => callback();
    ipcRenderer.on('menu:save', handler);
    return () => ipcRenderer.removeListener('menu:save', handler);
  },
  onNotification: (callback: (data: { message: string; timestamp: number }) => void) => {
    const handler = (_event: any, data: any) => callback(data);
    ipcRenderer.on('notification', handler);
    return () => ipcRenderer.removeListener('notification', handler);
  },
});

// renderer (React)
useEffect(() => {
  const unsubscribe = window.electronEvents.onSave(() => {
    saveCurrentFile();
  });
  return unsubscribe;
}, []);
```

### Pattern 4: Renderer → Renderer

No direct communication. Use main process as a message broker.

```typescript
// main.ts
const windows = new Map<string, BrowserWindow>();

ipcMain.on('broadcast', (event, channel: string, data: unknown) => {
  const senderWindow = BrowserWindow.fromWebContents(event.sender);

  windows.forEach((win) => {
    if (win !== senderWindow) {
      win.webContents.send(channel, data);
    }
  });
});

ipcMain.handle('sendToWindow', async (_event, windowId: string, channel: string, data: unknown) => {
  const targetWindow = windows.get(windowId);
  if (targetWindow) {
    targetWindow.webContents.send(channel, data);
    return true;
  }
  return false;
});
```

## Type-Safe IPC

### Define Channel Types

```typescript
// src/shared/ipc-types.ts
export interface IpcChannels {
  // Invoke/Handle (request-response)
  'dialog:openFile': {
    args: [];
    return: string | null;
  };
  'dialog:saveFile': {
    args: [defaultPath?: string];
    return: string | null;
  };
  'file:read': {
    args: [path: string];
    return: string;
  };
  'file:write': {
    args: [path: string, content: string];
    return: boolean;
  };
  'db:query': {
    args: [sql: string, params?: unknown[]];
    return: unknown[];
  };
  'app:getVersion': {
    args: [];
    return: string;
  };
  'store:get': {
    args: [key: string];
    return: unknown;
  };
  'store:set': {
    args: [key: string, value: unknown];
    return: void;
  };
}

export interface IpcEvents {
  // Send/On (one-way to main)
  'window:minimize': [];
  'window:maximize': [];
  'window:close': [];
  'log:info': [message: string];
}

export interface MainToRendererEvents {
  // webContents.send (main to renderer)
  'menu:action': [action: string];
  'update:available': [version: string];
  'update:progress': [percent: number];
  'notification': [data: { title: string; body: string }];
}
```

### Type-Safe Preload

```typescript
// preload.ts
import { contextBridge, ipcRenderer } from 'electron';
import type { IpcChannels, IpcEvents, MainToRendererEvents } from '../shared/ipc-types';

type IpcApi = {
  [K in keyof IpcChannels]: (
    ...args: IpcChannels[K]['args']
  ) => Promise<IpcChannels[K]['return']>;
};

type IpcSend = {
  [K in keyof IpcEvents]: (...args: IpcEvents[K]) => void;
};

type IpcOn = {
  [K in keyof MainToRendererEvents as `on${Capitalize<K>}`]: (
    callback: (...args: MainToRendererEvents[K]) => void
  ) => () => void;
};

const api: IpcApi = {
  'dialog:openFile': () => ipcRenderer.invoke('dialog:openFile'),
  'dialog:saveFile': (defaultPath) => ipcRenderer.invoke('dialog:saveFile', defaultPath),
  'file:read': (path) => ipcRenderer.invoke('file:read', path),
  'file:write': (path, content) => ipcRenderer.invoke('file:write', path, content),
  'db:query': (sql, params) => ipcRenderer.invoke('db:query', sql, params),
  'app:getVersion': () => ipcRenderer.invoke('app:getVersion'),
  'store:get': (key) => ipcRenderer.invoke('store:get', key),
  'store:set': (key, value) => ipcRenderer.invoke('store:set', key, value),
};

const send: IpcSend = {
  'window:minimize': () => ipcRenderer.send('window:minimize'),
  'window:maximize': () => ipcRenderer.send('window:maximize'),
  'window:close': () => ipcRenderer.send('window:close'),
  'log:info': (message) => ipcRenderer.send('log:info', message),
};

const events: IpcOn = {
  'onMenu:action': (callback) => {
    const handler = (_e: any, action: string) => callback(action);
    ipcRenderer.on('menu:action', handler);
    return () => ipcRenderer.removeListener('menu:action', handler);
  },
  'onUpdate:available': (callback) => {
    const handler = (_e: any, version: string) => callback(version);
    ipcRenderer.on('update:available', handler);
    return () => ipcRenderer.removeListener('update:available', handler);
  },
  'onUpdate:progress': (callback) => {
    const handler = (_e: any, percent: number) => callback(percent);
    ipcRenderer.on('update:progress', handler);
    return () => ipcRenderer.removeListener('update:progress', handler);
  },
  'onNotification': (callback) => {
    const handler = (_e: any, data: { title: string; body: string }) => callback(data);
    ipcRenderer.on('notification', handler);
    return () => ipcRenderer.removeListener('notification', handler);
  },
};

contextBridge.exposeInMainWorld('electron', {
  ...api,
  ...send,
  ...events,
});
```

### Type Declarations

```typescript
// src/preload/types.d.ts
import type { IpcChannels, IpcEvents, MainToRendererEvents } from '../shared/ipc-types';

type IpcApi = {
  [K in keyof IpcChannels]: (
    ...args: IpcChannels[K]['args']
  ) => Promise<IpcChannels[K]['return']>;
};

type IpcSend = {
  [K in keyof IpcEvents]: (...args: IpcEvents[K]) => void;
};

type IpcOn = {
  [K in keyof MainToRendererEvents as `on${Capitalize<K>}`]: (
    callback: (...args: MainToRendererEvents[K]) => void
  ) => () => void;
};

declare global {
  interface Window {
    electron: IpcApi & IpcSend & IpcOn;
  }
}

export {};
```

## Security Best Practices

### DO: Expose Minimal API

```typescript
// GOOD - Expose specific functions
contextBridge.exposeInMainWorld('electronAPI', {
  getVersion: () => ipcRenderer.invoke('app:getVersion'),
  openFile: () => ipcRenderer.invoke('dialog:openFile'),
});
```

### DON'T: Expose Raw IPC

```typescript
// BAD - Never do this!
contextBridge.exposeInMainWorld('electron', {
  ipcRenderer: ipcRenderer,  // Exposes everything!
});
```

### DO: Validate All Inputs

```typescript
// main.ts
ipcMain.handle('file:read', async (_event, path: string) => {
  // Validate path
  if (typeof path !== 'string') {
    throw new Error('Path must be a string');
  }

  // Prevent directory traversal
  const resolved = path.resolve(path);
  const allowed = app.getPath('userData');
  if (!resolved.startsWith(allowed)) {
    throw new Error('Access denied');
  }

  return fs.readFile(resolved, 'utf-8');
});
```

### DO: Verify Sender

```typescript
ipcMain.handle('sensitive:operation', async (event) => {
  // Verify the sender is from your app
  const senderFrame = event.senderFrame;

  if (!senderFrame.url.startsWith('file://') &&
      !senderFrame.url.startsWith('http://localhost')) {
    throw new Error('Unauthorized');
  }

  // Proceed with operation
});
```

## Object Serialization

IPC uses the [Structured Clone Algorithm](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm). Some objects cannot be sent:

### Supported Types
- Primitives (string, number, boolean, null, undefined)
- Arrays and plain objects
- Date, RegExp, Map, Set
- ArrayBuffer, TypedArray
- Blob, File
- Error objects

### NOT Supported
- Functions
- DOM elements
- Electron objects (BrowserWindow, WebContents)
- Node.js streams
- Circular references

```typescript
// Handle non-serializable data
ipcMain.handle('getWindowInfo', (event) => {
  const win = BrowserWindow.fromWebContents(event.sender);

  // Return serializable data only
  return {
    id: win?.id,
    title: win?.getTitle(),
    bounds: win?.getBounds(),
    isMaximized: win?.isMaximized(),
  };
});
```

## Reference

- [IPC Tutorial](https://www.electronjs.org/docs/latest/tutorial/ipc)
- [ipcMain](https://www.electronjs.org/docs/latest/api/ipc-main)
- [ipcRenderer](https://www.electronjs.org/docs/latest/api/ipc-renderer)
- [contextBridge](https://www.electronjs.org/docs/latest/api/context-bridge)
