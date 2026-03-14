# Tauri

> Official Documentation: https://tauri.app/

## Overview

Tauri is a framework for building cross-platform desktop applications using web technologies (HTML, CSS, JavaScript) with a Rust backend. It produces smaller binaries than Electron by using the system's native WebView instead of bundling Chromium.

Key advantages:
- **Small bundle size** - ~5MB vs Electron's ~100MB+
- **Native performance** - Rust backend for system operations
- **Security-first** - Sandboxed by default, no Node.js in renderer
- **Cross-platform** - Windows, macOS, Linux from single codebase

---

## Installation

### Prerequisites

```bash
# macOS
xcode-select --install

# Windows - Install Visual Studio Build Tools
# https://visualstudio.microsoft.com/visual-cpp-build-tools/

# Linux (Debian/Ubuntu)
sudo apt update
sudo apt install libwebkit2gtk-4.1-dev build-essential curl wget file \
  libssl-dev libayatana-appindicator3-dev librsvg2-dev

# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### Create New Project

```bash
# Using npm create
npm create tauri-app@latest

# With specific frontend framework
npm create tauri-app@latest my-app -- --template svelte-ts
npm create tauri-app@latest my-app -- --template react-ts
npm create tauri-app@latest my-app -- --template vue-ts

# Add Tauri to existing project
npm install -D @tauri-apps/cli@latest
npm run tauri init
```

---

## Project Structure

```
my-tauri-app/
├── src/                     # Frontend source (Svelte/React/Vue)
│   ├── App.svelte
│   ├── main.ts
│   └── styles.css
├── src-tauri/               # Rust backend
│   ├── Cargo.toml           # Rust dependencies
│   ├── tauri.conf.json      # Tauri configuration
│   ├── build.rs             # Build script
│   ├── icons/               # App icons
│   └── src/
│       ├── main.rs          # Entry point
│       ├── lib.rs           # Library code
│       └── commands/        # Custom commands (optional)
├── package.json
├── vite.config.ts
└── tsconfig.json
```

---

## Configuration (tauri.conf.json)

```json
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "My App",
  "version": "1.0.0",
  "identifier": "com.company.myapp",
  "build": {
    "beforeBuildCommand": "npm run build",
    "beforeDevCommand": "npm run dev",
    "devUrl": "http://localhost:5173",
    "frontendDist": "../dist"
  },
  "app": {
    "withGlobalTauri": false,
    "windows": [
      {
        "title": "My App",
        "width": 1200,
        "height": 800,
        "resizable": true,
        "fullscreen": false,
        "center": true
      }
    ],
    "security": {
      "csp": "default-src 'self'; script-src 'self'"
    }
  },
  "bundle": {
    "active": true,
    "targets": "all",
    "icon": ["icons/32x32.png", "icons/128x128.png", "icons/icon.icns", "icons/icon.ico"]
  }
}
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Rust Backend                          │
│  - System access (filesystem, processes, etc.)           │
│  - Custom commands (business logic)                      │
│  - Tauri runtime (window management, events)             │
│  - Plugins (dialog, shell, etc.)                         │
└──────────────────────┬──────────────────────────────────┘
                       │ IPC (invoke/events)
┌──────────────────────▼──────────────────────────────────┐
│                    WebView                               │
│  - HTML/CSS/JavaScript                                   │
│  - Svelte/React/Vue/etc.                                 │
│  - @tauri-apps/api (type-safe IPC)                      │
│  - No Node.js access (secure sandbox)                    │
└─────────────────────────────────────────────────────────┘
```

### Key Differences from Electron

| Aspect | Tauri | Electron |
|--------|-------|----------|
| Runtime | System WebView | Bundled Chromium |
| Backend | Rust | Node.js |
| Bundle size | ~5-10 MB | ~100-200 MB |
| Memory usage | Lower | Higher |
| Node.js in renderer | No | Optional |
| Security model | Sandboxed | Manual configuration |

---

## Basic Commands

```bash
# Development
npm run tauri dev

# Build for production
npm run tauri build

# Build for specific target
npm run tauri build --target x86_64-pc-windows-msvc
npm run tauri build --target aarch64-apple-darwin

# Generate icons from source image
npm run tauri icon ./src-tauri/icons/app-icon.png

# View info about the Tauri project
npm run tauri info
```

---

## Frontend Integration

### Using @tauri-apps/api

```typescript
// Install the API package
// npm install @tauri-apps/api

import { invoke } from '@tauri-apps/api/core';
import { listen, emit } from '@tauri-apps/api/event';
import { open, save } from '@tauri-apps/plugin-dialog';
import { readTextFile, writeTextFile } from '@tauri-apps/plugin-fs';

// Invoke a Rust command
const result = await invoke<string>('greet', { name: 'World' });

// Listen to events from Rust
const unlisten = await listen<string>('my-event', (event) => {
  console.log('Received:', event.payload);
});

// Emit event to Rust
await emit('frontend-event', { data: 'hello' });

// Open file dialog (requires plugin)
const selected = await open({
  multiple: false,
  filters: [{ name: 'Text', extensions: ['txt', 'md'] }]
});

// Read file (requires plugin)
if (selected) {
  const content = await readTextFile(selected);
}
```

### Type-Safe Invoke with Generics

```typescript
// Define command types
interface User {
  id: number;
  name: string;
  email: string;
}

// Type-safe invoke
const user = await invoke<User>('get_user', { id: 1 });
const users = await invoke<User[]>('list_users');
const success = await invoke<boolean>('delete_user', { id: 1 });
```

---

## Rust Backend Basics

### Entry Point (main.rs)

```rust
// src-tauri/src/main.rs
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

fn main() {
    tauri::Builder::default()
        .plugin(tauri_plugin_shell::init())
        .plugin(tauri_plugin_dialog::init())
        .plugin(tauri_plugin_fs::init())
        .invoke_handler(tauri::generate_handler![
            greet,
            get_user,
            save_settings
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}

#[tauri::command]
fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

#[tauri::command]
fn get_user(id: u32) -> Result<User, String> {
    // Implementation
}

#[tauri::command]
async fn save_settings(settings: Settings) -> Result<(), String> {
    // Async implementation
}
```

---

## Security Best Practices

### Content Security Policy

```json
{
  "app": {
    "security": {
      "csp": "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'"
    }
  }
}
```

### Capability-Based Permissions

```json
// src-tauri/capabilities/default.json
{
  "identifier": "default",
  "description": "Default permissions",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "shell:allow-open",
    "dialog:allow-open",
    "dialog:allow-save",
    "fs:allow-read-text-file",
    "fs:allow-write-text-file"
  ]
}
```

### Security Checklist

- ✅ Use allowlist for filesystem access
- ✅ Validate all IPC inputs in Rust
- ✅ Set appropriate CSP headers
- ✅ Use capability-based permissions (Tauri v2)
- ✅ Keep Tauri and dependencies updated
- ❌ Never expose raw shell commands
- ❌ Never disable security features for convenience
- ❌ Never trust frontend data without validation

---

## Common Patterns

### Window Management

```rust
use tauri::{Manager, Window};

#[tauri::command]
fn open_settings_window(app: tauri::AppHandle) -> Result<(), String> {
    let window = tauri::WebviewWindowBuilder::new(
        &app,
        "settings",
        tauri::WebviewUrl::App("settings.html".into())
    )
    .title("Settings")
    .inner_size(600.0, 400.0)
    .build()
    .map_err(|e| e.to_string())?;

    Ok(())
}

#[tauri::command]
fn close_window(window: Window) {
    window.close().unwrap();
}
```

### State Management

```rust
use std::sync::Mutex;
use tauri::State;

struct AppState {
    counter: Mutex<i32>,
    config: Mutex<Config>,
}

#[tauri::command]
fn increment(state: State<AppState>) -> i32 {
    let mut counter = state.counter.lock().unwrap();
    *counter += 1;
    *counter
}

fn main() {
    tauri::Builder::default()
        .manage(AppState {
            counter: Mutex::new(0),
            config: Mutex::new(Config::default()),
        })
        .invoke_handler(tauri::generate_handler![increment])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### Error Handling

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize)]
struct CommandError {
    message: String,
    code: String,
}

impl From<std::io::Error> for CommandError {
    fn from(err: std::io::Error) -> Self {
        CommandError {
            message: err.to_string(),
            code: "IO_ERROR".to_string(),
        }
    }
}

#[tauri::command]
fn read_config() -> Result<Config, CommandError> {
    let content = std::fs::read_to_string("config.json")?;
    let config: Config = serde_json::from_str(&content)
        .map_err(|e| CommandError {
            message: e.to_string(),
            code: "PARSE_ERROR".to_string(),
        })?;
    Ok(config)
}
```

---

## When to Use Tauri

**Good fit:**
- Desktop apps needing small bundle size
- Apps requiring native system access
- Security-conscious applications
- Apps using existing web frontend skills
- Cross-platform distribution requirements

**Consider alternatives when:**
- Need embedded Chromium features (Electron)
- Team has no Rust experience and can't learn
- App requires extensive Node.js ecosystem libraries
- Building browser extensions (not desktop apps)

---

## Related Topics

- [Tauri Commands](commands.md) - Deep dive into IPC
- [Tauri Plugins](plugins.md) - Official and custom plugins
- [Tauri Bundling](bundling.md) - Building and distributing
