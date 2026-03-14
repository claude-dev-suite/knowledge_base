# Tauri Plugins

> Official Documentation: https://tauri.app/develop/plugins/

## Overview

Tauri v2 uses a plugin system for system APIs. Official plugins provide secure, cross-platform access to OS features like dialogs, filesystem, shell, and more. Each plugin requires explicit permission configuration.

---

## Installing Plugins

### Using Cargo and npm

```bash
# Add Rust dependency
cargo add tauri-plugin-dialog

# Add JavaScript bindings
npm install @tauri-apps/plugin-dialog
```

### Register in main.rs

```rust
fn main() {
    tauri::Builder::default()
        .plugin(tauri_plugin_dialog::init())
        .plugin(tauri_plugin_fs::init())
        .plugin(tauri_plugin_shell::init())
        .plugin(tauri_plugin_os::init())
        .plugin(tauri_plugin_clipboard_manager::init())
        .plugin(tauri_plugin_notification::init())
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

---

## Official Plugins

### Dialog (`tauri-plugin-dialog`)

File open/save dialogs and message boxes.

```bash
cargo add tauri-plugin-dialog
npm install @tauri-apps/plugin-dialog
```

```typescript
import { open, save, message, ask, confirm } from '@tauri-apps/plugin-dialog';

// Open file
const selected = await open({
  multiple: false,
  directory: false,
  filters: [{
    name: 'Documents',
    extensions: ['pdf', 'doc', 'docx']
  }]
});

// Open multiple files
const files = await open({
  multiple: true,
  filters: [{
    name: 'Images',
    extensions: ['png', 'jpg', 'gif']
  }]
});

// Open directory
const dir = await open({
  directory: true,
  recursive: true
});

// Save file
const savePath = await save({
  defaultPath: 'document.txt',
  filters: [{
    name: 'Text',
    extensions: ['txt']
  }]
});

// Message box
await message('Operation completed!', { title: 'Success', kind: 'info' });

// Confirmation dialog
const confirmed = await confirm('Are you sure?', {
  title: 'Confirm Action',
  kind: 'warning'
});

// Yes/No question
const answer = await ask('Do you want to save changes?', {
  title: 'Unsaved Changes',
  kind: 'warning'
});
```

**Permissions:**
```json
{
  "permissions": [
    "dialog:allow-open",
    "dialog:allow-save",
    "dialog:allow-message",
    "dialog:allow-ask",
    "dialog:allow-confirm"
  ]
}
```

---

### Filesystem (`tauri-plugin-fs`)

Read, write, and manage files and directories.

```bash
cargo add tauri-plugin-fs
npm install @tauri-apps/plugin-fs
```

```typescript
import {
  readTextFile,
  writeTextFile,
  readDir,
  mkdir,
  remove,
  rename,
  copyFile,
  exists,
  stat,
  BaseDirectory
} from '@tauri-apps/plugin-fs';

// Read text file
const content = await readTextFile('config.json', {
  baseDir: BaseDirectory.AppConfig
});

// Write text file
await writeTextFile('data.txt', 'Hello, World!', {
  baseDir: BaseDirectory.AppData
});

// Read binary file
const bytes = await readFile('image.png', {
  baseDir: BaseDirectory.Resource
});

// List directory contents
const entries = await readDir('documents', {
  baseDir: BaseDirectory.Home
});

for (const entry of entries) {
  console.log(entry.name, entry.isDirectory);
}

// Create directory
await mkdir('my-app/data', {
  baseDir: BaseDirectory.AppData,
  recursive: true
});

// Check if file exists
const fileExists = await exists('config.json', {
  baseDir: BaseDirectory.AppConfig
});

// Get file info
const info = await stat('document.pdf', {
  baseDir: BaseDirectory.Download
});
console.log(info.size, info.mtime);

// Delete file
await remove('temp.txt', {
  baseDir: BaseDirectory.Temp
});
```

**Base Directories:**
- `AppConfig` - App configuration directory
- `AppData` - App data directory
- `AppLocalData` - App local data
- `Desktop` - User's desktop
- `Document` - User's documents
- `Download` - User's downloads
- `Home` - User's home directory
- `Resource` - App resources (read-only)
- `Temp` - System temp directory

**Permissions:**
```json
{
  "permissions": [
    "fs:allow-read-text-file",
    "fs:allow-write-text-file",
    "fs:allow-read-dir",
    "fs:allow-mkdir",
    "fs:allow-remove",
    {
      "identifier": "fs:scope",
      "allow": ["$APPDATA/**", "$DOCUMENT/**"]
    }
  ]
}
```

---

### Shell (`tauri-plugin-shell`)

Execute commands and open URLs/files with default apps.

```bash
cargo add tauri-plugin-shell
npm install @tauri-apps/plugin-shell
```

```typescript
import { Command, open } from '@tauri-apps/plugin-shell';

// Open URL in default browser
await open('https://example.com');

// Open file with default application
await open('/path/to/document.pdf');

// Execute command (requires allowlist)
const output = await Command.create('echo', ['Hello, World!']).execute();
console.log(output.stdout);

// Execute with sidecar
const sidecar = Command.sidecar('binaries/my-tool');
const result = await sidecar.execute();

// Spawn process and stream output
const command = Command.create('npm', ['run', 'build']);

command.on('close', (data) => {
  console.log(`Process exited with code ${data.code}`);
});

command.on('error', (error) => {
  console.error(`Error: ${error}`);
});

command.stdout.on('data', (line) => {
  console.log(`stdout: ${line}`);
});

command.stderr.on('data', (line) => {
  console.error(`stderr: ${line}`);
});

const child = await command.spawn();

// Kill process
await child.kill();
```

**Permissions:**
```json
{
  "permissions": [
    "shell:allow-open",
    {
      "identifier": "shell:allow-execute",
      "allow": [
        { "name": "echo", "cmd": "echo", "args": true }
      ]
    }
  ]
}
```

---

### OS (`tauri-plugin-os`)

System information and platform detection.

```bash
cargo add tauri-plugin-os
npm install @tauri-apps/plugin-os
```

```typescript
import {
  platform,
  arch,
  type,
  version,
  hostname,
  locale,
  tempdir
} from '@tauri-apps/plugin-os';

const platformName = await platform();  // 'darwin', 'windows', 'linux'
const architecture = await arch();       // 'x86_64', 'aarch64'
const osType = await type();            // 'Darwin', 'Windows_NT', 'Linux'
const osVersion = await version();       // '10.15.7', '10.0.19041'
const host = await hostname();           // 'my-computer'
const userLocale = await locale();       // 'en-US'
const tmpDir = await tempdir();          // '/tmp' or 'C:\Users\...\Temp'
```

**Permissions:**
```json
{
  "permissions": [
    "os:allow-platform",
    "os:allow-arch",
    "os:allow-type",
    "os:allow-version",
    "os:allow-hostname",
    "os:allow-locale"
  ]
}
```

---

### Clipboard (`tauri-plugin-clipboard-manager`)

Read and write clipboard contents.

```bash
cargo add tauri-plugin-clipboard-manager
npm install @tauri-apps/plugin-clipboard-manager
```

```typescript
import { writeText, readText, writeImage, readImage } from '@tauri-apps/plugin-clipboard-manager';

// Write text
await writeText('Hello, clipboard!');

// Read text
const text = await readText();

// Write image (base64)
await writeImage(base64ImageData);

// Read image
const imageData = await readImage();
```

**Permissions:**
```json
{
  "permissions": [
    "clipboard-manager:allow-write-text",
    "clipboard-manager:allow-read-text",
    "clipboard-manager:allow-write-image",
    "clipboard-manager:allow-read-image"
  ]
}
```

---

### Notification (`tauri-plugin-notification`)

System notifications.

```bash
cargo add tauri-plugin-notification
npm install @tauri-apps/plugin-notification
```

```typescript
import {
  sendNotification,
  requestPermission,
  isPermissionGranted
} from '@tauri-apps/plugin-notification';

// Check permission
let permission = await isPermissionGranted();
if (!permission) {
  permission = await requestPermission();
}

if (permission) {
  // Simple notification
  sendNotification('Hello from Tauri!');

  // With options
  sendNotification({
    title: 'New Message',
    body: 'You have a new message from Alice',
    icon: 'icons/notification.png'
  });
}
```

**Permissions:**
```json
{
  "permissions": [
    "notification:allow-send-notification",
    "notification:allow-request-permission",
    "notification:allow-is-permission-granted"
  ]
}
```

---

### Store (`tauri-plugin-store`)

Persistent key-value storage.

```bash
cargo add tauri-plugin-store
npm install @tauri-apps/plugin-store
```

```typescript
import { Store } from '@tauri-apps/plugin-store';

// Create or load store
const store = new Store('settings.json');

// Set values
await store.set('theme', 'dark');
await store.set('user', { name: 'Alice', id: 1 });

// Get values
const theme = await store.get<string>('theme');
const user = await store.get<{ name: string; id: number }>('user');

// Check if key exists
const hasTheme = await store.has('theme');

// Delete key
await store.delete('theme');

// Get all keys
const keys = await store.keys();

// Get all values
const values = await store.values();

// Get all entries
const entries = await store.entries();

// Clear store
await store.clear();

// Save to disk (auto-saves on set by default)
await store.save();

// Load from disk
await store.load();

// Listen to changes
await store.onKeyChange('theme', (value) => {
  console.log('Theme changed:', value);
});
```

**Permissions:**
```json
{
  "permissions": [
    "store:allow-get",
    "store:allow-set",
    "store:allow-delete",
    "store:allow-keys",
    "store:allow-values",
    "store:allow-entries",
    "store:allow-clear",
    "store:allow-save",
    "store:allow-load"
  ]
}
```

---

### HTTP (`tauri-plugin-http`)

HTTP client for making requests.

```bash
cargo add tauri-plugin-http
npm install @tauri-apps/plugin-http
```

```typescript
import { fetch } from '@tauri-apps/plugin-http';

// GET request
const response = await fetch('https://api.example.com/data', {
  method: 'GET',
  headers: {
    'Authorization': 'Bearer token'
  }
});

const data = await response.json();

// POST request
const postResponse = await fetch('https://api.example.com/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    name: 'Alice',
    email: 'alice@example.com'
  })
});

// With timeout
const timeoutResponse = await fetch('https://api.example.com/data', {
  method: 'GET',
  connectTimeout: 30000 // 30 seconds
});
```

**Permissions:**
```json
{
  "permissions": [
    {
      "identifier": "http:default",
      "allow": [
        { "url": "https://api.example.com/**" }
      ]
    }
  ]
}
```

---

### Updater (`tauri-plugin-updater`)

Auto-update functionality.

```bash
cargo add tauri-plugin-updater
npm install @tauri-apps/plugin-updater
```

```typescript
import { check, installUpdate, onUpdaterEvent } from '@tauri-apps/plugin-updater';
import { relaunch } from '@tauri-apps/plugin-process';

// Check for updates
const update = await check();

if (update?.available) {
  console.log(`Update available: ${update.currentVersion} -> ${update.version}`);

  // Download and install
  await update.downloadAndInstall();

  // Restart app
  await relaunch();
}

// Listen to update events
onUpdaterEvent(({ status, error }) => {
  console.log('Updater status:', status);
  if (error) {
    console.error('Update error:', error);
  }
});
```

**Configuration (tauri.conf.json):**
```json
{
  "plugins": {
    "updater": {
      "pubkey": "YOUR_PUBLIC_KEY",
      "endpoints": [
        "https://releases.example.com/{{target}}/{{arch}}/{{current_version}}"
      ]
    }
  }
}
```

---

## Creating Custom Plugins

### Rust Side

```rust
// src-tauri/src/plugins/my_plugin.rs
use tauri::{
    plugin::{Builder, TauriPlugin},
    Runtime,
};

#[tauri::command]
fn my_command() -> String {
    "Hello from plugin!".into()
}

pub fn init<R: Runtime>() -> TauriPlugin<R> {
    Builder::new("my-plugin")
        .invoke_handler(tauri::generate_handler![my_command])
        .build()
}
```

```rust
// main.rs
mod plugins;

fn main() {
    tauri::Builder::default()
        .plugin(plugins::my_plugin::init())
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### JavaScript Side

```typescript
import { invoke } from '@tauri-apps/api/core';

export async function myCommand(): Promise<string> {
  return invoke('plugin:my-plugin|my_command');
}
```

---

## Capability Configuration

```json
// src-tauri/capabilities/default.json
{
  "$schema": "https://tauri.app/schema/capability.json",
  "identifier": "default",
  "description": "Default app permissions",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "dialog:default",
    "fs:default",
    "shell:allow-open",
    "os:default",
    "clipboard-manager:allow-write-text",
    "clipboard-manager:allow-read-text",
    "notification:default",
    "store:default"
  ]
}
```

---

## Related Topics

- [Tauri Basics](basics.md) - Overview and setup
- [Tauri Commands](commands.md) - Custom Rust commands
- [Tauri Bundling](bundling.md) - Building and distributing
