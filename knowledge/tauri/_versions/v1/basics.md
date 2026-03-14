# Tauri 1 → Basics Delta

## Not Available in Tauri 1

- iOS/Android support (Tauri 2+)
- Multiwebview support
- Channel-based IPC
- New permission system
- Swift/Kotlin mobile bindings
- Tray icon improvements

## Syntax Differences

### Project Structure

```
// Tauri 2
src-tauri/
├── Cargo.toml
├── tauri.conf.json
├── capabilities/           # NEW: Permission system
│   └── default.json
├── src/
│   ├── main.rs
│   └── lib.rs             # NEW: Library entry point
└── gen/                    # NEW: Generated mobile code
    ├── android/
    └── ios/

// Tauri 1
src-tauri/
├── Cargo.toml
├── tauri.conf.json
├── src/
│   └── main.rs
└── icons/
```

### Command Definition

```rust
// Tauri 2 - Commands in lib.rs
// src-tauri/src/lib.rs
#[tauri::command]
fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![greet])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}

// Tauri 1 - Commands in main.rs
// src-tauri/src/main.rs
#[tauri::command]
fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

fn main() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![greet])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### Plugin System

```rust
// Tauri 2 - New plugin architecture
// tauri.conf.json
{
  "plugins": {
    "fs": {
      "scope": ["$APP/*"]
    },
    "shell": {
      "open": true
    }
  }
}

// Cargo.toml
[dependencies]
tauri-plugin-fs = "2"
tauri-plugin-shell = "2"

// src-tauri/src/lib.rs
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_fs::init())
        .plugin(tauri_plugin_shell::init())
        .run(tauri::generate_context!())
        .expect("error");
}


// Tauri 1 - Built-in APIs
// tauri.conf.json
{
  "tauri": {
    "allowlist": {
      "fs": {
        "scope": ["$APP/*"],
        "readFile": true,
        "writeFile": true
      },
      "shell": {
        "open": true
      }
    }
  }
}
```

### Frontend Invocation

```typescript
// Tauri 2 - Plugin-based imports
import { readTextFile } from '@tauri-apps/plugin-fs';
import { open } from '@tauri-apps/plugin-shell';

await readTextFile('file.txt');
await open('https://tauri.app');

// Tauri 1 - Single package imports
import { readTextFile } from '@tauri-apps/api/fs';
import { open } from '@tauri-apps/api/shell';

await readTextFile('file.txt');
await open('https://tauri.app');
```

### Permission System

```json
// Tauri 2 - Capabilities
// src-tauri/capabilities/default.json
{
  "identifier": "default",
  "description": "Default capability",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "fs:default",
    "shell:allow-open"
  ]
}

// Tauri 1 - Allowlist
// tauri.conf.json
{
  "tauri": {
    "allowlist": {
      "all": false,
      "fs": {
        "all": false,
        "readFile": true
      }
    }
  }
}
```

### Window Management

```rust
// Tauri 2
use tauri::Manager;

#[tauri::command]
async fn create_window(app: tauri::AppHandle) {
    tauri::WebviewWindowBuilder::new(
        &app,
        "external",
        tauri::WebviewUrl::External("https://tauri.app".parse().unwrap())
    )
    .title("Tauri Website")
    .build()
    .unwrap();
}

// Tauri 1
use tauri::Manager;

#[tauri::command]
async fn create_window(app: tauri::AppHandle) {
    tauri::WindowBuilder::new(
        &app,
        "external",
        tauri::WindowUrl::External("https://tauri.app".parse().unwrap())
    )
    .title("Tauri Website")
    .build()
    .unwrap();
}
```

## Still Current in Tauri 1

- Core IPC with invoke()
- Custom protocol (tauri://)
- Window events and management
- App lifecycle hooks
- Auto-updater
- System tray (different API)
- Custom window decorations

## Recommendations for Tauri 1 Users

1. **Plan mobile support** - Only available in Tauri 2
2. **Use allowlist** for API access control
3. **Single main.rs entry** for commands
4. **Consider upgrading** for new permission system

## Migration Path

1. Restructure to lib.rs entry point
2. Convert allowlist to capabilities
3. Update plugin imports
4. Replace WindowBuilder with WebviewWindowBuilder
5. Update @tauri-apps/api imports to plugin packages

