# Electron Auto-Updates

## Overview

Electron provides built-in support for auto-updates through the `autoUpdater` module and the Squirrel framework. For most applications, the `electron-updater` package (part of electron-builder) is recommended.

## Update Options

| Option | Use Case | Requirements |
|--------|----------|--------------|
| `update-electron-app` | Open-source apps on GitHub | Public repo, GitHub releases |
| `electron-updater` | Any app | Custom/GitHub server |
| Built-in `autoUpdater` | macOS only | Update server |

## update-electron-app (Simplest)

For open-source apps with public GitHub repos.

### Installation

```bash
npm install update-electron-app
```

### Setup

```typescript
// main.ts
import { updateElectronApp, UpdateSourceType } from 'update-electron-app';

updateElectronApp({
  // Default: GitHub releases from package.json repository
  updateSource: {
    type: UpdateSourceType.ElectronPublicUpdateService,
    repo: 'owner/repo',
  },

  // Check interval (default: 10 minutes)
  updateInterval: '1 hour',

  // Show native dialog when update ready
  notifyUser: true,

  // Log to console
  logger: console,
});
```

### Requirements

- Public GitHub repository
- Releases published to GitHub Releases
- macOS builds must be code-signed
- Windows builds work without signing

## electron-updater (Recommended)

Full-featured auto-update solution from electron-builder.

### Installation

```bash
npm install electron-updater
```

### Basic Setup

```typescript
// src/main/updater.ts
import { autoUpdater } from 'electron-updater';
import { app, BrowserWindow, dialog } from 'electron';
import log from 'electron-log';

export function setupAutoUpdater(mainWindow: BrowserWindow): void {
  // Configure logging
  autoUpdater.logger = log;
  log.transports.file.level = 'info';

  // Don't auto-download, ask user first
  autoUpdater.autoDownload = false;

  // Install on quit
  autoUpdater.autoInstallOnAppQuit = true;

  // Event handlers
  autoUpdater.on('checking-for-update', () => {
    log.info('Checking for update...');
    sendToRenderer(mainWindow, 'update:checking');
  });

  autoUpdater.on('update-available', (info) => {
    log.info('Update available:', info.version);
    sendToRenderer(mainWindow, 'update:available', info);

    // Show dialog
    dialog.showMessageBox(mainWindow, {
      type: 'info',
      title: 'Update Available',
      message: `Version ${info.version} is available.`,
      detail: `Current version: ${app.getVersion()}\n\nWould you like to download it now?`,
      buttons: ['Download', 'Later'],
      defaultId: 0,
    }).then(({ response }) => {
      if (response === 0) {
        autoUpdater.downloadUpdate();
      }
    });
  });

  autoUpdater.on('update-not-available', (info) => {
    log.info('Update not available');
    sendToRenderer(mainWindow, 'update:not-available', info);
  });

  autoUpdater.on('download-progress', (progress) => {
    log.info(`Download progress: ${progress.percent.toFixed(1)}%`);
    mainWindow.setProgressBar(progress.percent / 100);
    sendToRenderer(mainWindow, 'update:progress', progress);
  });

  autoUpdater.on('update-downloaded', (info) => {
    log.info('Update downloaded:', info.version);
    mainWindow.setProgressBar(-1); // Remove progress bar
    sendToRenderer(mainWindow, 'update:downloaded', info);

    // Show restart dialog
    dialog.showMessageBox(mainWindow, {
      type: 'info',
      title: 'Update Ready',
      message: 'Update has been downloaded.',
      detail: 'The application will restart to install the update.',
      buttons: ['Restart Now', 'Later'],
      defaultId: 0,
    }).then(({ response }) => {
      if (response === 0) {
        autoUpdater.quitAndInstall(false, true);
      }
    });
  });

  autoUpdater.on('error', (error) => {
    log.error('Update error:', error);
    mainWindow.setProgressBar(-1);
    sendToRenderer(mainWindow, 'update:error', error.message);

    // Don't show error dialogs in development
    if (!app.isPackaged) return;

    dialog.showErrorBox(
      'Update Error',
      'Failed to check for updates. Please try again later.'
    );
  });
}

function sendToRenderer(win: BrowserWindow, channel: string, data?: unknown): void {
  win.webContents.send(channel, data);
}

// Start checking
export function checkForUpdates(): void {
  // Don't check in development
  if (!app.isPackaged) {
    log.info('Skipping update check in development');
    return;
  }

  autoUpdater.checkForUpdates();
}
```

### Usage in Main Process

```typescript
// src/main/index.ts
import { app, BrowserWindow } from 'electron';
import { setupAutoUpdater, checkForUpdates } from './updater';

let mainWindow: BrowserWindow | null = null;

app.whenReady().then(() => {
  mainWindow = createWindow();

  // Setup auto-updater
  setupAutoUpdater(mainWindow);

  // Check for updates after window is ready
  mainWindow.once('ready-to-show', () => {
    // Delay check to not slow down startup
    setTimeout(checkForUpdates, 3000);
  });
});

// Periodic update check (every 4 hours)
setInterval(checkForUpdates, 4 * 60 * 60 * 1000);
```

### Renderer Integration

```typescript
// src/preload/index.ts
contextBridge.exposeInMainWorld('updates', {
  onChecking: (callback: () => void) => {
    ipcRenderer.on('update:checking', callback);
    return () => ipcRenderer.removeListener('update:checking', callback);
  },
  onAvailable: (callback: (info: UpdateInfo) => void) => {
    ipcRenderer.on('update:available', (_e, info) => callback(info));
    return () => ipcRenderer.removeListener('update:available', callback);
  },
  onNotAvailable: (callback: () => void) => {
    ipcRenderer.on('update:not-available', callback);
    return () => ipcRenderer.removeListener('update:not-available', callback);
  },
  onProgress: (callback: (progress: ProgressInfo) => void) => {
    ipcRenderer.on('update:progress', (_e, progress) => callback(progress));
    return () => ipcRenderer.removeListener('update:progress', callback);
  },
  onDownloaded: (callback: (info: UpdateInfo) => void) => {
    ipcRenderer.on('update:downloaded', (_e, info) => callback(info));
    return () => ipcRenderer.removeListener('update:downloaded', callback);
  },
  onError: (callback: (error: string) => void) => {
    ipcRenderer.on('update:error', (_e, error) => callback(error));
    return () => ipcRenderer.removeListener('update:error', callback);
  },
  checkForUpdates: () => ipcRenderer.invoke('update:check'),
});

// React hook
function useAutoUpdates() {
  const [status, setStatus] = useState<'idle' | 'checking' | 'available' | 'downloading' | 'ready'>('idle');
  const [progress, setProgress] = useState(0);
  const [version, setVersion] = useState<string | null>(null);

  useEffect(() => {
    const unsubscribers = [
      window.updates.onChecking(() => setStatus('checking')),
      window.updates.onAvailable((info) => {
        setStatus('available');
        setVersion(info.version);
      }),
      window.updates.onNotAvailable(() => setStatus('idle')),
      window.updates.onProgress((p) => {
        setStatus('downloading');
        setProgress(p.percent);
      }),
      window.updates.onDownloaded(() => setStatus('ready')),
      window.updates.onError(() => setStatus('idle')),
    ];

    return () => unsubscribers.forEach(unsub => unsub());
  }, []);

  return { status, progress, version };
}
```

## electron-builder Configuration

```yaml
# electron-builder.yml
publish:
  provider: github
  owner: company
  repo: myapp
  releaseType: release

# Or use S3
publish:
  provider: s3
  bucket: my-releases
  region: us-east-1

# Or generic server
publish:
  provider: generic
  url: https://releases.example.com/
```

## Custom Update Server

### Using Hazel (Free, Vercel)

```bash
# Deploy to Vercel
npx degit zeit/hazel my-update-server
cd my-update-server
npm install
vercel
```

Configure updater:
```typescript
autoUpdater.setFeedURL({
  provider: 'generic',
  url: 'https://your-hazel.vercel.app',
});
```

### Using Nuts (Self-hosted)

```bash
# Deploy Nuts
git clone https://github.com/GitbookIO/nuts.git
cd nuts
npm install
# Configure with GitHub token
```

### Manual Server (Express)

```typescript
// update-server.ts
import express from 'express';
import path from 'path';
import fs from 'fs';

const app = express();

// Serve update files
app.use('/releases', express.static('releases'));

// Latest version endpoint (for generic provider)
app.get('/latest.yml', (req, res) => {
  res.sendFile(path.join(__dirname, 'releases/latest.yml'));
});

// Platform-specific latest
app.get('/update/:platform/:version', (req, res) => {
  const { platform, version } = req.params;

  // Read latest.yml to get current version
  const latestYml = fs.readFileSync('releases/latest.yml', 'utf-8');
  const latestVersion = latestYml.match(/version: (.+)/)?.[1];

  if (latestVersion && latestVersion !== version) {
    // Return update info
    res.json({
      url: `https://releases.example.com/MyApp-${latestVersion}.${platform === 'darwin' ? 'dmg' : 'exe'}`,
      name: `v${latestVersion}`,
      notes: 'Release notes here',
      pub_date: new Date().toISOString(),
    });
  } else {
    res.status(204).send();
  }
});

app.listen(3000);
```

## Update File Structure

```
releases/
├── latest.yml            # Windows/Linux update info
├── latest-mac.yml        # macOS update info
├── MyApp-1.0.0.exe       # Windows installer
├── MyApp-1.0.0.exe.blockmap
├── MyApp-1.0.0.dmg       # macOS installer
├── MyApp-1.0.0.dmg.blockmap
├── MyApp-1.0.0.zip       # macOS zip (for auto-update)
├── MyApp-1.0.0.zip.blockmap
├── MyApp-1.0.0.AppImage  # Linux
└── MyApp-1.0.0.deb       # Linux Debian
```

### latest.yml Example

```yaml
version: 1.0.0
files:
  - url: MyApp-1.0.0.exe
    sha512: abc123...
    size: 52428800
path: MyApp-1.0.0.exe
sha512: abc123...
releaseDate: '2024-01-15T12:00:00.000Z'
```

## Delta Updates

electron-builder supports delta updates for faster downloads:

```yaml
# electron-builder.yml
nsis:
  differentialPackage: true
```

## Staged Rollouts

Gradually roll out updates to a percentage of users:

```typescript
// Check if this user should receive update
autoUpdater.on('update-available', (info) => {
  // Use user ID or random to determine rollout
  const userId = getUniqueUserId();
  const rolloutPercentage = info.stagingPercentage || 100;

  // Hash user ID to get consistent percentage
  const userPercentage = hashToPercentage(userId);

  if (userPercentage <= rolloutPercentage) {
    // User is in rollout group
    showUpdateDialog(info);
  }
});
```

## Testing Updates

### Development Testing

```typescript
// Force update check in development
if (process.env.NODE_ENV === 'development') {
  autoUpdater.forceDevUpdateConfig = true;
  autoUpdater.updateConfigPath = path.join(__dirname, 'dev-app-update.yml');
}
```

### dev-app-update.yml

```yaml
owner: company
repo: myapp
provider: github
```

## Error Handling

```typescript
autoUpdater.on('error', (error) => {
  log.error('Update error:', error);

  // Categorize errors
  if (error.message.includes('net::')) {
    // Network error
    scheduleRetry();
  } else if (error.message.includes('signature')) {
    // Signature verification failed
    log.error('Update signature invalid - possible tampering');
  } else if (error.message.includes('ENOENT')) {
    // File not found
    log.error('Update file missing on server');
  }
});

function scheduleRetry(): void {
  // Retry after 30 minutes
  setTimeout(() => {
    autoUpdater.checkForUpdates();
  }, 30 * 60 * 1000);
}
```

## Security Considerations

1. **Always sign releases**: Updates should be code-signed
2. **Verify signatures**: electron-updater verifies signatures automatically
3. **Use HTTPS**: All update URLs should use HTTPS
4. **Validate server**: Consider certificate pinning for update server

## Checklist

```markdown
## Auto-Update Checklist

### Setup
- [ ] electron-updater installed
- [ ] Update server configured (GitHub/S3/custom)
- [ ] Code signing configured for all platforms

### Implementation
- [ ] Update check on startup (delayed)
- [ ] Periodic update checks
- [ ] User notification for available updates
- [ ] Download progress shown
- [ ] Restart prompt when ready
- [ ] Error handling with retry logic

### Testing
- [ ] Tested update flow end-to-end
- [ ] Verified signature verification
- [ ] Tested network error recovery
- [ ] Tested rollback if update fails

### Release Process
- [ ] Version bumped
- [ ] Changelog updated
- [ ] Release notes written
- [ ] Staged rollout configured (if needed)
```

## Reference

- [Auto-Update Tutorial](https://www.electronjs.org/docs/latest/tutorial/updates)
- [electron-updater](https://www.electron.build/auto-update)
- [update-electron-app](https://github.com/electron/update-electron-app)
- [Hazel](https://github.com/vercel/hazel)
