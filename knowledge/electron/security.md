# Electron Security Best Practices

## Overview

Electron applications have access to both web and native capabilities, making security critical. A vulnerability could allow attackers to execute arbitrary code on users' machines.

## Security Defaults (Electron 20+)

| Setting | Default | Since |
|---------|---------|-------|
| `contextIsolation` | `true` | Electron 12 |
| `nodeIntegration` | `false` | Electron 5 |
| `sandbox` | `true` | Electron 20 |
| `webSecurity` | `true` | Always |

## Core Security Principles

### 1. Context Isolation

Isolates preload scripts and Electron APIs in a dedicated JavaScript context, preventing renderer code from modifying Node.js or Electron internals.

```typescript
// SECURE: contextIsolation enabled (default)
new BrowserWindow({
  webPreferences: {
    contextIsolation: true,  // Default in Electron 12+
    preload: path.join(__dirname, 'preload.js'),
  },
});

// preload.ts - Safe API exposure
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('electronAPI', {
  // Only expose what's needed
  openFile: () => ipcRenderer.invoke('dialog:openFile'),
  getVersion: () => ipcRenderer.invoke('app:getVersion'),
});
```

### 2. Disable Node Integration

Never enable Node.js in renderers that load remote content.

```typescript
// SECURE: nodeIntegration disabled (default)
new BrowserWindow({
  webPreferences: {
    nodeIntegration: false,  // Default since Electron 5
    contextIsolation: true,
  },
});
```

### 3. Enable Sandbox

Use OS-level process isolation for renderer processes.

```typescript
// SECURE: Sandbox enabled (default in Electron 20+)
new BrowserWindow({
  webPreferences: {
    sandbox: true,  // Default since Electron 20
  },
});
```

### 4. Web Security

Never disable same-origin policy.

```typescript
// SECURE: webSecurity enabled (always default)
new BrowserWindow({
  webPreferences: {
    webSecurity: true,  // NEVER disable this
    allowRunningInsecureContent: false,
  },
});
```

## Secure BrowserWindow Configuration

```typescript
import { BrowserWindow, app } from 'electron';
import path from 'path';

function createSecureWindow(): BrowserWindow {
  const win = new BrowserWindow({
    width: 1200,
    height: 800,
    webPreferences: {
      // Required security settings
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false,
      sandbox: true,
      webSecurity: true,

      // Additional hardening
      allowRunningInsecureContent: false,
      experimentalFeatures: false,
      enableWebSQL: false,
      navigateOnDragDrop: false,

      // Spellcheck is safe
      spellcheck: true,
    },
  });

  // Content Security Policy
  win.webContents.session.webRequest.onHeadersReceived((details, callback) => {
    callback({
      responseHeaders: {
        ...details.responseHeaders,
        'Content-Security-Policy': [
          [
            "default-src 'self'",
            "script-src 'self'",
            "style-src 'self' 'unsafe-inline'",
            "img-src 'self' data: https:",
            "font-src 'self' data:",
            "connect-src 'self' https://api.yourdomain.com",
            "frame-ancestors 'none'",
          ].join('; '),
        ],
      },
    });
  });

  return win;
}
```

## Navigation Security

### Restrict Navigation

```typescript
// Prevent navigation to untrusted URLs
mainWindow.webContents.on('will-navigate', (event, url) => {
  const parsedUrl = new URL(url);

  // Allow only your origins
  const allowedOrigins = ['file://', 'http://localhost:5173'];

  if (!allowedOrigins.some(origin => url.startsWith(origin))) {
    event.preventDefault();
    console.warn('Blocked navigation to:', url);
  }
});

// Handle redirects
mainWindow.webContents.on('will-redirect', (event, url) => {
  // Same validation as will-navigate
});
```

### Control New Windows

```typescript
// Prevent untrusted popup windows
mainWindow.webContents.setWindowOpenHandler(({ url }) => {
  const parsedUrl = new URL(url);

  // Open external links in default browser
  if (parsedUrl.protocol === 'https:') {
    shell.openExternal(url);
    return { action: 'deny' };
  }

  // Deny all other popups
  return { action: 'deny' };
});
```

## IPC Security

### Validate All Inputs

```typescript
import { ipcMain, app } from 'electron';
import path from 'path';
import fs from 'fs/promises';

// Path validation helper
function isPathSafe(filePath: string, basePath: string): boolean {
  const resolved = path.resolve(filePath);
  const base = path.resolve(basePath);
  return resolved.startsWith(base + path.sep);
}

ipcMain.handle('file:read', async (_event, filePath: string) => {
  // Type validation
  if (typeof filePath !== 'string') {
    throw new Error('Invalid path type');
  }

  // Prevent directory traversal
  const userDataPath = app.getPath('userData');
  if (!isPathSafe(filePath, userDataPath)) {
    throw new Error('Access denied: path outside allowed directory');
  }

  return fs.readFile(filePath, 'utf-8');
});

ipcMain.handle('db:query', async (_event, query: string, params: unknown[]) => {
  // Whitelist allowed queries
  const allowedQueries = [
    'SELECT * FROM items WHERE id = ?',
    'SELECT * FROM items ORDER BY created_at DESC',
    'INSERT INTO items (name) VALUES (?)',
  ];

  if (!allowedQueries.includes(query)) {
    throw new Error('Query not allowed');
  }

  return db.prepare(query).all(...params);
});
```

### Never Expose Raw IPC

```typescript
// BAD - Never do this!
contextBridge.exposeInMainWorld('ipc', ipcRenderer);

// BAD - Still dangerous
contextBridge.exposeInMainWorld('api', {
  invoke: (channel: string, ...args: unknown[]) =>
    ipcRenderer.invoke(channel, ...args),
});

// GOOD - Expose specific, validated functions
contextBridge.exposeInMainWorld('api', {
  getItems: () => ipcRenderer.invoke('items:getAll'),
  createItem: (name: string) => {
    if (typeof name !== 'string' || name.length > 100) {
      throw new Error('Invalid name');
    }
    return ipcRenderer.invoke('items:create', name);
  },
});
```

## External Content

### Validate URLs for shell.openExternal

```typescript
import { shell } from 'electron';

function openExternalSafe(url: string): boolean {
  try {
    const parsed = new URL(url);

    // Only allow http(s) protocols
    if (!['http:', 'https:'].includes(parsed.protocol)) {
      console.warn('Blocked non-http URL:', url);
      return false;
    }

    // Optional: whitelist domains
    const allowedDomains = ['github.com', 'example.com'];
    if (!allowedDomains.some(d => parsed.hostname.endsWith(d))) {
      console.warn('Blocked non-whitelisted domain:', url);
      return false;
    }

    shell.openExternal(url);
    return true;
  } catch {
    console.error('Invalid URL:', url);
    return false;
  }
}

// IPC handler
ipcMain.handle('openExternal', async (_event, url: string) => {
  return openExternalSafe(url);
});
```

### Use HTTPS Only

```typescript
// Enforce HTTPS for all remote content
mainWindow.webContents.session.webRequest.onBeforeRequest(
  { urls: ['http://*'] },
  (details, callback) => {
    // Upgrade to HTTPS
    const httpsUrl = details.url.replace('http://', 'https://');
    callback({ redirectURL: httpsUrl });
  }
);
```

## Sensitive Data Protection

### Use safeStorage for Credentials

```typescript
import { safeStorage } from 'electron';
import Store from 'electron-store';

const store = new Store({ name: 'secure-storage' });

export const secureStore = {
  set(key: string, value: string): void {
    if (safeStorage.isEncryptionAvailable()) {
      const encrypted = safeStorage.encryptString(value);
      store.set(key, encrypted.toString('base64'));
    } else {
      // Fallback - warn user
      console.warn('Encryption not available');
      store.set(key, value);
    }
  },

  get(key: string): string | null {
    const stored = store.get(key) as string | undefined;
    if (!stored) return null;

    if (safeStorage.isEncryptionAvailable()) {
      try {
        const buffer = Buffer.from(stored, 'base64');
        return safeStorage.decryptString(buffer);
      } catch {
        return null;
      }
    }
    return stored;
  },

  delete(key: string): void {
    store.delete(key);
  },
};
```

### Never Store Secrets in Renderer

```typescript
// BAD - Exposed to renderer
const apiKey = process.env.API_KEY;
window.apiKey = apiKey;

// GOOD - Keep in main process
// main.ts
ipcMain.handle('api:request', async (_event, endpoint: string) => {
  const response = await fetch(`https://api.example.com${endpoint}`, {
    headers: { Authorization: `Bearer ${process.env.API_KEY}` },
  });
  return response.json();
});
```

## Security Checklist

```markdown
## Pre-Release Security Audit

### Process Isolation
- [ ] contextIsolation: true
- [ ] nodeIntegration: false
- [ ] sandbox: true
- [ ] webSecurity: true

### IPC
- [ ] No raw ipcRenderer exposed
- [ ] All inputs validated
- [ ] Path traversal prevented
- [ ] SQL injection prevented
- [ ] Sender verification implemented

### Navigation
- [ ] will-navigate handler restricts URLs
- [ ] will-redirect handler restricts URLs
- [ ] setWindowOpenHandler denies untrusted origins
- [ ] shell.openExternal validates URLs

### Content
- [ ] CSP headers configured
- [ ] HTTPS enforced for remote content
- [ ] No eval() or new Function() with user input
- [ ] No innerHTML with unsanitized content

### Sensitive Data
- [ ] Credentials stored with safeStorage
- [ ] No secrets in renderer process
- [ ] Sensitive data cleared on logout
- [ ] No sensitive data in logs

### Updates & Signing
- [ ] Code signing enabled
- [ ] Auto-updater verifies signatures
- [ ] ASAR integrity enabled
- [ ] Hardened runtime (macOS)

### Dependencies
- [ ] Dependencies audited (npm audit)
- [ ] No known vulnerabilities
- [ ] electron-is-dev check for dev features
```

## Electron Fuses

Electron Fuses allow you to disable certain features at build time.

```bash
# Install @electron/fuses
npm install --save-dev @electron/fuses

# Configure in your build script
npx @electron/fuses read dist/MyApp.app

# Flip fuses
npx @electron/fuses write dist/MyApp.app \
  --runAsNode=off \
  --enableCookieEncryption=on \
  --enableNodeOptionsEnvironmentVariable=off \
  --enableNodeCliInspectArguments=off \
  --enableEmbeddedAsarIntegrityValidation=on
```

## Reference

- [Security Tutorial](https://www.electronjs.org/docs/latest/tutorial/security)
- [Context Isolation](https://www.electronjs.org/docs/latest/tutorial/context-isolation)
- [Fuses](https://www.electronjs.org/docs/latest/tutorial/fuses)
- [OWASP Electron Security](https://cheatsheetseries.owasp.org/cheatsheets/Electron_Security_Cheat_Sheet.html)
