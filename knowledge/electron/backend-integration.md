# Electron Backend Integration

## Overview

Electron desktop applications often need to interact with backend services, whether embedded within the app or external APIs. This guide covers common patterns for integrating backend functionality.

## Integration Patterns

| Pattern | Description | Use Case |
|---------|-------------|----------|
| Embedded Server | Express/Fastify in main process | Offline-capable apps |
| Local Database | SQLite/LevelDB for persistence | Data-heavy desktop apps |
| External API | REST/GraphQL to remote server | Cloud-connected apps |
| Hybrid | Local DB + Remote sync | Offline-first apps |

## Embedded Express Server

Run a local HTTP server in the main process.

### Setup

```typescript
// src/main/backend/server.ts
import express, { Express, Request, Response, NextFunction } from 'express';
import cors from 'cors';
import { Server } from 'http';
import { app as electronApp } from 'electron';

let server: Server | null = null;
let serverPort: number = 0;

export async function startServer(): Promise<number> {
  const app: Express = express();

  // Middleware
  app.use(cors({ origin: 'file://' }));
  app.use(express.json({ limit: '10mb' }));

  // Request logging
  app.use((req, res, next) => {
    console.log(`${req.method} ${req.path}`);
    next();
  });

  // Health check
  app.get('/api/health', (req, res) => {
    res.json({
      status: 'ok',
      version: electronApp.getVersion(),
      uptime: process.uptime(),
    });
  });

  // API routes
  app.use('/api/items', itemsRouter);
  app.use('/api/settings', settingsRouter);

  // Error handler
  app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
    console.error('Server error:', err);
    res.status(500).json({ error: err.message });
  });

  return new Promise((resolve, reject) => {
    // Listen on random available port, localhost only
    server = app.listen(0, '127.0.0.1', () => {
      const address = server!.address();
      if (typeof address === 'object' && address) {
        serverPort = address.port;
        console.log(`Backend server running on http://127.0.0.1:${serverPort}`);
        resolve(serverPort);
      } else {
        reject(new Error('Failed to get server address'));
      }
    });

    server.on('error', reject);
  });
}

export function stopServer(): void {
  if (server) {
    server.close();
    server = null;
    console.log('Backend server stopped');
  }
}

export function getServerPort(): number {
  return serverPort;
}
```

### Router Example

```typescript
// src/main/backend/routes/items.ts
import { Router } from 'express';
import { itemsRepo } from '../../database';

const router = Router();

router.get('/', async (req, res) => {
  const items = await itemsRepo.getAll();
  res.json(items);
});

router.get('/:id', async (req, res) => {
  const item = await itemsRepo.getById(parseInt(req.params.id));
  if (!item) {
    return res.status(404).json({ error: 'Item not found' });
  }
  res.json(item);
});

router.post('/', async (req, res) => {
  const { name, description } = req.body;

  if (!name || typeof name !== 'string') {
    return res.status(400).json({ error: 'Name is required' });
  }

  const item = await itemsRepo.create({ name, description });
  res.status(201).json(item);
});

router.put('/:id', async (req, res) => {
  const id = parseInt(req.params.id);
  const item = await itemsRepo.update(id, req.body);
  if (!item) {
    return res.status(404).json({ error: 'Item not found' });
  }
  res.json(item);
});

router.delete('/:id', async (req, res) => {
  const deleted = await itemsRepo.delete(parseInt(req.params.id));
  if (!deleted) {
    return res.status(404).json({ error: 'Item not found' });
  }
  res.status(204).send();
});

export default router;
```

### Integration with Main Process

```typescript
// src/main/index.ts
import { app, BrowserWindow, ipcMain } from 'electron';
import { startServer, stopServer, getServerPort } from './backend/server';
import { initDatabase } from './database';

let mainWindow: BrowserWindow | null = null;

app.whenReady().then(async () => {
  // Initialize database first
  initDatabase();

  // Start embedded server
  const port = await startServer();

  // Create window
  mainWindow = createWindow();

  // Expose server port to renderer
  ipcMain.handle('server:getPort', () => getServerPort());
});

app.on('will-quit', () => {
  stopServer();
});
```

### Renderer Usage

```typescript
// src/renderer/services/api.ts
let serverPort: number | null = null;

async function getBaseUrl(): Promise<string> {
  if (!serverPort) {
    serverPort = await window.electron.getServerPort();
  }
  return `http://127.0.0.1:${serverPort}/api`;
}

export const api = {
  async get<T>(endpoint: string): Promise<T> {
    const baseUrl = await getBaseUrl();
    const response = await fetch(`${baseUrl}${endpoint}`);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
  },

  async post<T>(endpoint: string, data: unknown): Promise<T> {
    const baseUrl = await getBaseUrl();
    const response = await fetch(`${baseUrl}${endpoint}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
  },

  // ... put, delete, etc.
};
```

## Local Database (SQLite)

### Setup with better-sqlite3

```bash
npm install better-sqlite3
npm install --save-dev @types/better-sqlite3
```

```typescript
// src/main/database/index.ts
import Database from 'better-sqlite3';
import { app } from 'electron';
import path from 'path';
import fs from 'fs';

let db: Database.Database;

export function initDatabase(): Database.Database {
  const userDataPath = app.getPath('userData');
  const dbPath = path.join(userDataPath, 'app.db');

  // Ensure directory exists
  fs.mkdirSync(path.dirname(dbPath), { recursive: true });

  db = new Database(dbPath, {
    // Enable verbose logging in development
    // verbose: console.log,
  });

  // Performance optimizations
  db.pragma('journal_mode = WAL');
  db.pragma('synchronous = NORMAL');
  db.pragma('foreign_keys = ON');

  // Run migrations
  runMigrations();

  return db;
}

export function getDatabase(): Database.Database {
  if (!db) throw new Error('Database not initialized');
  return db;
}

export function closeDatabase(): void {
  db?.close();
}
```

### Migrations

```typescript
// src/main/database/migrations.ts
import { getDatabase } from './index';
import fs from 'fs';
import path from 'path';

interface Migration {
  name: string;
  sql: string;
}

const migrationsDir = path.join(__dirname, 'migrations');

export function runMigrations(): void {
  const db = getDatabase();

  // Create migrations table
  db.exec(`
    CREATE TABLE IF NOT EXISTS _migrations (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL UNIQUE,
      applied_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )
  `);

  // Get applied migrations
  const applied = db
    .prepare('SELECT name FROM _migrations')
    .all()
    .map((row: any) => row.name);

  // Read and sort migration files
  const files = fs
    .readdirSync(migrationsDir)
    .filter(f => f.endsWith('.sql'))
    .sort();

  // Apply pending migrations
  for (const file of files) {
    if (!applied.includes(file)) {
      const sql = fs.readFileSync(path.join(migrationsDir, file), 'utf-8');

      db.transaction(() => {
        db.exec(sql);
        db.prepare('INSERT INTO _migrations (name) VALUES (?)').run(file);
      })();

      console.log(`Applied migration: ${file}`);
    }
  }
}
```

### Migration Files

```sql
-- src/main/database/migrations/001_initial.sql
CREATE TABLE items (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  description TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_items_created_at ON items(created_at);

CREATE TRIGGER tr_items_updated_at
AFTER UPDATE ON items
BEGIN
  UPDATE items SET updated_at = CURRENT_TIMESTAMP WHERE id = NEW.id;
END;

-- src/main/database/migrations/002_add_categories.sql
CREATE TABLE categories (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL UNIQUE
);

ALTER TABLE items ADD COLUMN category_id INTEGER REFERENCES categories(id);
CREATE INDEX idx_items_category ON items(category_id);
```

### Repository Pattern

```typescript
// src/main/database/repositories/items.ts
import { getDatabase } from '../index';

export interface Item {
  id: number;
  name: string;
  description: string | null;
  category_id: number | null;
  created_at: string;
  updated_at: string;
}

export interface CreateItemInput {
  name: string;
  description?: string;
  category_id?: number;
}

export const itemsRepo = {
  getAll(options?: { categoryId?: number; limit?: number; offset?: number }): Item[] {
    const db = getDatabase();
    let sql = 'SELECT * FROM items';
    const params: unknown[] = [];

    if (options?.categoryId) {
      sql += ' WHERE category_id = ?';
      params.push(options.categoryId);
    }

    sql += ' ORDER BY created_at DESC';

    if (options?.limit) {
      sql += ' LIMIT ?';
      params.push(options.limit);
      if (options.offset) {
        sql += ' OFFSET ?';
        params.push(options.offset);
      }
    }

    return db.prepare(sql).all(...params) as Item[];
  },

  getById(id: number): Item | undefined {
    const db = getDatabase();
    return db.prepare('SELECT * FROM items WHERE id = ?').get(id) as Item | undefined;
  },

  create(input: CreateItemInput): Item {
    const db = getDatabase();
    const stmt = db.prepare(`
      INSERT INTO items (name, description, category_id)
      VALUES (?, ?, ?)
    `);
    const result = stmt.run(input.name, input.description || null, input.category_id || null);
    return this.getById(result.lastInsertRowid as number)!;
  },

  update(id: number, input: Partial<CreateItemInput>): Item | undefined {
    const db = getDatabase();
    const fields: string[] = [];
    const values: unknown[] = [];

    if (input.name !== undefined) {
      fields.push('name = ?');
      values.push(input.name);
    }
    if (input.description !== undefined) {
      fields.push('description = ?');
      values.push(input.description);
    }
    if (input.category_id !== undefined) {
      fields.push('category_id = ?');
      values.push(input.category_id);
    }

    if (fields.length === 0) return this.getById(id);

    values.push(id);
    db.prepare(`UPDATE items SET ${fields.join(', ')} WHERE id = ?`).run(...values);
    return this.getById(id);
  },

  delete(id: number): boolean {
    const db = getDatabase();
    const result = db.prepare('DELETE FROM items WHERE id = ?').run(id);
    return result.changes > 0;
  },

  search(query: string): Item[] {
    const db = getDatabase();
    return db.prepare(`
      SELECT * FROM items
      WHERE name LIKE ? OR description LIKE ?
      ORDER BY created_at DESC
    `).all(`%${query}%`, `%${query}%`) as Item[];
  },
};
```

### IPC Handlers for Database

```typescript
// src/main/ipc/database.ts
import { ipcMain } from 'electron';
import { itemsRepo } from '../database/repositories/items';

export function registerDatabaseHandlers(): void {
  ipcMain.handle('db:items:getAll', async (_event, options) => {
    return itemsRepo.getAll(options);
  });

  ipcMain.handle('db:items:getById', async (_event, id: number) => {
    return itemsRepo.getById(id);
  });

  ipcMain.handle('db:items:create', async (_event, input) => {
    // Validate input
    if (!input.name || typeof input.name !== 'string') {
      throw new Error('Name is required');
    }
    return itemsRepo.create(input);
  });

  ipcMain.handle('db:items:update', async (_event, id: number, input) => {
    return itemsRepo.update(id, input);
  });

  ipcMain.handle('db:items:delete', async (_event, id: number) => {
    return itemsRepo.delete(id);
  });

  ipcMain.handle('db:items:search', async (_event, query: string) => {
    return itemsRepo.search(query);
  });
}
```

## External API Communication

### Preload API Client

```typescript
// src/preload/api.ts
import { contextBridge } from 'electron';

const API_BASE = process.env.API_URL || 'https://api.example.com';

interface RequestOptions {
  method?: string;
  headers?: Record<string, string>;
  body?: unknown;
}

async function request<T>(endpoint: string, options: RequestOptions = {}): Promise<T> {
  const { method = 'GET', headers = {}, body } = options;

  // Get auth token from secure storage (via IPC)
  const token = await window.electron.getAuthToken();

  const response = await fetch(`${API_BASE}${endpoint}`, {
    method,
    headers: {
      'Content-Type': 'application/json',
      ...(token && { Authorization: `Bearer ${token}` }),
      ...headers,
    },
    body: body ? JSON.stringify(body) : undefined,
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({}));
    throw new Error(error.message || `HTTP ${response.status}`);
  }

  // Handle 204 No Content
  if (response.status === 204) return undefined as T;

  return response.json();
}

contextBridge.exposeInMainWorld('api', {
  get: <T>(endpoint: string) => request<T>(endpoint),
  post: <T>(endpoint: string, data: unknown) =>
    request<T>(endpoint, { method: 'POST', body: data }),
  put: <T>(endpoint: string, data: unknown) =>
    request<T>(endpoint, { method: 'PUT', body: data }),
  patch: <T>(endpoint: string, data: unknown) =>
    request<T>(endpoint, { method: 'PATCH', body: data }),
  delete: <T>(endpoint: string) =>
    request<T>(endpoint, { method: 'DELETE' }),
});
```

## Offline-First with Sync

### Sync Manager

```typescript
// src/renderer/services/sync-manager.ts
interface SyncQueueItem {
  id: string;
  operation: 'create' | 'update' | 'delete';
  entity: string;
  data: unknown;
  timestamp: number;
  retries: number;
}

class SyncManager {
  private queue: SyncQueueItem[] = [];
  private isOnline = navigator.onLine;
  private isSyncing = false;
  private maxRetries = 3;

  constructor() {
    window.addEventListener('online', () => this.handleOnline());
    window.addEventListener('offline', () => this.handleOffline());
    this.loadQueue();
  }

  private handleOnline(): void {
    console.log('Online - starting sync');
    this.isOnline = true;
    this.processQueue();
  }

  private handleOffline(): void {
    console.log('Offline - queuing operations');
    this.isOnline = false;
  }

  async enqueue(item: Omit<SyncQueueItem, 'id' | 'timestamp' | 'retries'>): Promise<void> {
    const queueItem: SyncQueueItem = {
      ...item,
      id: crypto.randomUUID(),
      timestamp: Date.now(),
      retries: 0,
    };

    this.queue.push(queueItem);
    await this.saveQueue();

    if (this.isOnline) {
      this.processQueue();
    }
  }

  private async processQueue(): Promise<void> {
    if (this.isSyncing || this.queue.length === 0 || !this.isOnline) return;

    this.isSyncing = true;

    while (this.queue.length > 0 && this.isOnline) {
      const item = this.queue[0];

      try {
        await this.syncItem(item);
        this.queue.shift();
        await this.saveQueue();
        console.log(`Synced: ${item.operation} ${item.entity}`);
      } catch (error) {
        console.error(`Sync failed for ${item.id}:`, error);
        item.retries++;

        if (item.retries >= this.maxRetries) {
          console.error(`Max retries reached for ${item.id}`);
          this.queue.shift();
          // Optionally move to dead-letter queue
        }
        break;
      }
    }

    this.isSyncing = false;
  }

  private async syncItem(item: SyncQueueItem): Promise<void> {
    const { operation, entity, data } = item;

    switch (operation) {
      case 'create':
        await window.api.post(`/${entity}`, data);
        break;
      case 'update':
        await window.api.put(`/${entity}/${(data as any).id}`, data);
        break;
      case 'delete':
        await window.api.delete(`/${entity}/${(data as any).id}`);
        break;
    }
  }

  private async loadQueue(): Promise<void> {
    const stored = localStorage.getItem('sync-queue');
    if (stored) {
      this.queue = JSON.parse(stored);
    }
  }

  private async saveQueue(): Promise<void> {
    localStorage.setItem('sync-queue', JSON.stringify(this.queue));
  }

  getQueueLength(): number {
    return this.queue.length;
  }

  isProcessing(): boolean {
    return this.isSyncing;
  }
}

export const syncManager = new SyncManager();
```

### Conflict Resolution

```typescript
// src/renderer/services/conflict-resolver.ts
interface ConflictData<T> {
  local: T;
  remote: T;
  base?: T; // Original before changes
}

type Resolution<T> = 'local' | 'remote' | T;

export async function resolveConflict<T extends { updated_at: string }>(
  conflict: ConflictData<T>,
  strategy: 'last-write-wins' | 'merge' | 'ask-user' = 'last-write-wins'
): Promise<Resolution<T>> {
  switch (strategy) {
    case 'last-write-wins':
      // Compare timestamps
      const localTime = new Date(conflict.local.updated_at).getTime();
      const remoteTime = new Date(conflict.remote.updated_at).getTime();
      return localTime > remoteTime ? 'local' : 'remote';

    case 'merge':
      // Attempt automatic merge (for compatible changes)
      return mergeObjects(conflict);

    case 'ask-user':
      // Show UI for user to decide
      return await showConflictDialog(conflict);
  }
}

function mergeObjects<T>(conflict: ConflictData<T>): T {
  // Simple merge: take all fields from both, prefer local for conflicts
  return { ...conflict.remote, ...conflict.local };
}

async function showConflictDialog<T>(conflict: ConflictData<T>): Promise<Resolution<T>> {
  // Implement UI dialog
  return 'local'; // Default
}
```

## Secure Token Storage

```typescript
// src/main/auth/token-store.ts
import { safeStorage, app } from 'electron';
import Store from 'electron-store';

const store = new Store({
  name: 'auth',
  encryptionKey: process.env.STORE_ENCRYPTION_KEY,
});

export const tokenStore = {
  async setToken(token: string): Promise<void> {
    if (safeStorage.isEncryptionAvailable()) {
      const encrypted = safeStorage.encryptString(token);
      store.set('accessToken', encrypted.toString('base64'));
    } else {
      // Fallback - less secure, warn user
      console.warn('OS encryption not available');
      store.set('accessToken', token);
    }
  },

  async getToken(): Promise<string | null> {
    const stored = store.get('accessToken') as string | undefined;
    if (!stored) return null;

    if (safeStorage.isEncryptionAvailable()) {
      try {
        const buffer = Buffer.from(stored, 'base64');
        return safeStorage.decryptString(buffer);
      } catch (error) {
        console.error('Failed to decrypt token:', error);
        return null;
      }
    }

    return stored;
  },

  async clearToken(): Promise<void> {
    store.delete('accessToken');
  },

  hasToken(): boolean {
    return store.has('accessToken');
  },
};

// IPC handlers
ipcMain.handle('auth:setToken', async (_event, token: string) => {
  await tokenStore.setToken(token);
});

ipcMain.handle('auth:getToken', async () => {
  return tokenStore.getToken();
});

ipcMain.handle('auth:clearToken', async () => {
  await tokenStore.clearToken();
});

ipcMain.handle('auth:hasToken', () => {
  return tokenStore.hasToken();
});
```

## OAuth Authentication Flow

```typescript
// src/main/auth/oauth.ts
import { BrowserWindow, session } from 'electron';

interface OAuthConfig {
  authUrl: string;
  tokenUrl: string;
  clientId: string;
  redirectUri: string;
  scope: string;
}

export async function authenticate(config: OAuthConfig): Promise<string> {
  return new Promise((resolve, reject) => {
    const authWindow = new BrowserWindow({
      width: 800,
      height: 600,
      show: false,
      webPreferences: {
        nodeIntegration: false,
        contextIsolation: true,
      },
    });

    const authUrl = new URL(config.authUrl);
    authUrl.searchParams.set('client_id', config.clientId);
    authUrl.searchParams.set('redirect_uri', config.redirectUri);
    authUrl.searchParams.set('response_type', 'code');
    authUrl.searchParams.set('scope', config.scope);

    authWindow.loadURL(authUrl.toString());
    authWindow.show();

    // Handle redirect
    authWindow.webContents.on('will-redirect', async (event, url) => {
      const parsedUrl = new URL(url);

      if (url.startsWith(config.redirectUri)) {
        event.preventDefault();

        const code = parsedUrl.searchParams.get('code');
        const error = parsedUrl.searchParams.get('error');

        authWindow.close();

        if (error) {
          reject(new Error(error));
          return;
        }

        if (code) {
          try {
            const token = await exchangeCodeForToken(config, code);
            resolve(token);
          } catch (err) {
            reject(err);
          }
        }
      }
    });

    authWindow.on('closed', () => {
      reject(new Error('Authentication window closed'));
    });
  });
}

async function exchangeCodeForToken(config: OAuthConfig, code: string): Promise<string> {
  const response = await fetch(config.tokenUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      client_id: config.clientId,
      redirect_uri: config.redirectUri,
    }),
  });

  if (!response.ok) {
    throw new Error('Token exchange failed');
  }

  const data = await response.json();
  return data.access_token;
}
```

## Reference

- [Electron APIs](https://www.electronjs.org/docs/latest/api)
- [better-sqlite3](https://github.com/WiseLibs/better-sqlite3)
- [electron-store](https://github.com/sindresorhus/electron-store)
- [safeStorage](https://www.electronjs.org/docs/latest/api/safe-storage)
