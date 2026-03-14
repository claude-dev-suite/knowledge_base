# Tauri 1 → Commands Delta

## Differences from Tauri 2

### Entry Point

```rust
// Tauri 2 - lib.rs with mobile support
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

// src-tauri/src/main.rs
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

fn main() {
    app_lib::run();
}


// Tauri 1 - main.rs only
// src-tauri/src/main.rs
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

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

### State Management

```rust
// Tauri 2 - Same pattern, works on mobile
use std::sync::Mutex;

struct AppState {
    counter: Mutex<i32>,
}

#[tauri::command]
fn increment(state: tauri::State<AppState>) -> i32 {
    let mut counter = state.counter.lock().unwrap();
    *counter += 1;
    *counter
}

pub fn run() {
    tauri::Builder::default()
        .manage(AppState { counter: Mutex::new(0) })
        .invoke_handler(tauri::generate_handler![increment])
        .run(tauri::generate_context!())
        .expect("error");
}

// Tauri 1 - Same state management
// Works identically, but in main.rs
```

### Async Commands

```rust
// Both Tauri 1 and 2 - Async commands
#[tauri::command]
async fn fetch_data(url: String) -> Result<String, String> {
    reqwest::get(&url)
        .await
        .map_err(|e| e.to_string())?
        .text()
        .await
        .map_err(|e| e.to_string())
}
```

### Channel-Based IPC (Tauri 2 Only)

```rust
// Tauri 2 - Channels for streaming
use tauri::ipc::Channel;

#[tauri::command]
fn stream_data(channel: Channel<String>) {
    std::thread::spawn(move || {
        for i in 0..10 {
            channel.send(format!("Message {}", i)).unwrap();
            std::thread::sleep(std::time::Duration::from_secs(1));
        }
    });
}

// Frontend
import { Channel } from '@tauri-apps/api/core';

const channel = new Channel<string>();
channel.onmessage = (message) => {
    console.log('Received:', message);
};
await invoke('stream_data', { channel });


// Tauri 1 - Use events for streaming
use tauri::Manager;

#[tauri::command]
fn stream_data(window: tauri::Window) {
    std::thread::spawn(move || {
        for i in 0..10 {
            window.emit("stream-message", format!("Message {}", i)).unwrap();
            std::thread::sleep(std::time::Duration::from_secs(1));
        }
    });
}

// Frontend
import { listen } from '@tauri-apps/api/event';

const unlisten = await listen('stream-message', (event) => {
    console.log('Received:', event.payload);
});
await invoke('stream_data');
```

## Still Current in Tauri 1

- Basic #[tauri::command] macro
- State management with tauri::State
- Async/await support
- Error handling with Result
- Window parameter injection
- AppHandle parameter injection

## Frontend Invocation

```typescript
// Both Tauri 1 and 2
import { invoke } from '@tauri-apps/api/core';
// or in Tauri 1: import { invoke } from '@tauri-apps/api/tauri';

const result = await invoke<string>('greet', { name: 'World' });
```

