# Tauri Commands

> Official Documentation: https://tauri.app/develop/calling-rust/

## Overview

Commands are the primary way for the frontend to communicate with the Rust backend. They use a type-safe IPC (Inter-Process Communication) mechanism that serializes data as JSON.

---

## Basic Commands

### Simple Command

```rust
// src-tauri/src/main.rs

#[tauri::command]
fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

// Register in main()
fn main() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![greet])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

```typescript
// Frontend
import { invoke } from '@tauri-apps/api/core';

const message = await invoke<string>('greet', { name: 'World' });
console.log(message); // "Hello, World!"
```

### Command with Multiple Parameters

```rust
#[tauri::command]
fn calculate(a: i32, b: i32, operation: &str) -> Result<i32, String> {
    match operation {
        "add" => Ok(a + b),
        "subtract" => Ok(a - b),
        "multiply" => Ok(a * b),
        "divide" => {
            if b == 0 {
                Err("Division by zero".to_string())
            } else {
                Ok(a / b)
            }
        }
        _ => Err(format!("Unknown operation: {}", operation)),
    }
}
```

```typescript
const result = await invoke<number>('calculate', {
  a: 10,
  b: 5,
  operation: 'add'
});
```

---

## Async Commands

### Basic Async

```rust
#[tauri::command]
async fn fetch_data(url: String) -> Result<String, String> {
    let response = reqwest::get(&url)
        .await
        .map_err(|e| e.to_string())?;

    let body = response.text()
        .await
        .map_err(|e| e.to_string())?;

    Ok(body)
}
```

### Async with State

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

struct Database {
    pool: sqlx::SqlitePool,
}

#[tauri::command]
async fn get_items(
    db: tauri::State<'_, Arc<Mutex<Database>>>
) -> Result<Vec<Item>, String> {
    let db = db.lock().await;

    let items = sqlx::query_as!(Item, "SELECT * FROM items")
        .fetch_all(&db.pool)
        .await
        .map_err(|e| e.to_string())?;

    Ok(items)
}
```

---

## Complex Data Types

### Structs with Serde

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: u32,
    name: String,
    email: String,
    #[serde(default)]
    active: bool,
}

#[derive(Debug, Serialize, Deserialize)]
struct CreateUserRequest {
    name: String,
    email: String,
}

#[tauri::command]
fn create_user(request: CreateUserRequest) -> Result<User, String> {
    // Validate input
    if request.name.is_empty() {
        return Err("Name cannot be empty".to_string());
    }

    if !request.email.contains('@') {
        return Err("Invalid email format".to_string());
    }

    Ok(User {
        id: 1, // Would be from database
        name: request.name,
        email: request.email,
        active: true,
    })
}

#[tauri::command]
fn get_users() -> Vec<User> {
    vec![
        User { id: 1, name: "Alice".into(), email: "alice@example.com".into(), active: true },
        User { id: 2, name: "Bob".into(), email: "bob@example.com".into(), active: false },
    ]
}
```

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  active: boolean;
}

interface CreateUserRequest {
  name: string;
  email: string;
}

const newUser = await invoke<User>('create_user', {
  request: { name: 'Charlie', email: 'charlie@example.com' }
});

const users = await invoke<User[]>('get_users');
```

### Enums

```rust
#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "type")]
enum Message {
    Text { content: String },
    Image { url: String, width: u32, height: u32 },
    File { path: String, size: u64 },
}

#[tauri::command]
fn process_message(message: Message) -> String {
    match message {
        Message::Text { content } => format!("Text: {}", content),
        Message::Image { url, .. } => format!("Image: {}", url),
        Message::File { path, size } => format!("File: {} ({} bytes)", path, size),
    }
}
```

```typescript
type Message =
  | { type: 'Text'; content: string }
  | { type: 'Image'; url: string; width: number; height: number }
  | { type: 'File'; path: string; size: number };

await invoke('process_message', {
  message: { type: 'Text', content: 'Hello!' }
});
```

---

## State Management

### Basic State

```rust
use std::sync::Mutex;
use tauri::State;

struct AppState {
    counter: Mutex<i32>,
}

#[tauri::command]
fn get_count(state: State<AppState>) -> i32 {
    *state.counter.lock().unwrap()
}

#[tauri::command]
fn increment(state: State<AppState>) -> i32 {
    let mut counter = state.counter.lock().unwrap();
    *counter += 1;
    *counter
}

#[tauri::command]
fn set_count(state: State<AppState>, value: i32) {
    let mut counter = state.counter.lock().unwrap();
    *counter = value;
}

fn main() {
    tauri::Builder::default()
        .manage(AppState {
            counter: Mutex::new(0),
        })
        .invoke_handler(tauri::generate_handler![
            get_count,
            increment,
            set_count
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### Multiple States

```rust
struct Config {
    theme: Mutex<String>,
    language: Mutex<String>,
}

struct Database {
    // Database connection
}

#[tauri::command]
fn get_theme(config: State<Config>) -> String {
    config.theme.lock().unwrap().clone()
}

#[tauri::command]
async fn save_to_db(
    db: State<'_, Database>,
    config: State<'_, Config>
) -> Result<(), String> {
    // Use both states
    Ok(())
}

fn main() {
    tauri::Builder::default()
        .manage(Config {
            theme: Mutex::new("dark".to_string()),
            language: Mutex::new("en".to_string()),
        })
        .manage(Database { /* ... */ })
        // ...
}
```

---

## Error Handling

### Custom Error Type

```rust
use serde::Serialize;

#[derive(Debug, Serialize)]
pub struct CommandError {
    pub code: String,
    pub message: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub details: Option<String>,
}

impl CommandError {
    pub fn new(code: &str, message: &str) -> Self {
        Self {
            code: code.to_string(),
            message: message.to_string(),
            details: None,
        }
    }

    pub fn with_details(mut self, details: &str) -> Self {
        self.details = Some(details.to_string());
        self
    }
}

impl From<std::io::Error> for CommandError {
    fn from(err: std::io::Error) -> Self {
        CommandError::new("IO_ERROR", &err.to_string())
    }
}

impl From<serde_json::Error> for CommandError {
    fn from(err: serde_json::Error) -> Self {
        CommandError::new("JSON_ERROR", &err.to_string())
    }
}

#[tauri::command]
fn load_config() -> Result<Config, CommandError> {
    let path = "config.json";

    if !std::path::Path::new(path).exists() {
        return Err(CommandError::new("NOT_FOUND", "Config file not found")
            .with_details(path));
    }

    let content = std::fs::read_to_string(path)?;
    let config: Config = serde_json::from_str(&content)?;

    Ok(config)
}
```

```typescript
interface CommandError {
  code: string;
  message: string;
  details?: string;
}

try {
  const config = await invoke<Config>('load_config');
} catch (error) {
  const err = error as CommandError;
  console.error(`[${err.code}] ${err.message}`);
  if (err.details) {
    console.error(`Details: ${err.details}`);
  }
}
```

---

## Window and App Handle

### Accessing Window

```rust
use tauri::Window;

#[tauri::command]
fn minimize(window: Window) {
    window.minimize().unwrap();
}

#[tauri::command]
fn toggle_fullscreen(window: Window) {
    let is_fullscreen = window.is_fullscreen().unwrap();
    window.set_fullscreen(!is_fullscreen).unwrap();
}

#[tauri::command]
fn set_title(window: Window, title: String) {
    window.set_title(&title).unwrap();
}
```

### Accessing AppHandle

```rust
use tauri::{AppHandle, Manager};

#[tauri::command]
fn open_new_window(app: AppHandle) -> Result<(), String> {
    let window = tauri::WebviewWindowBuilder::new(
        &app,
        "secondary",
        tauri::WebviewUrl::App("secondary.html".into())
    )
    .title("Secondary Window")
    .build()
    .map_err(|e| e.to_string())?;

    Ok(())
}

#[tauri::command]
fn get_app_version(app: AppHandle) -> String {
    app.package_info().version.to_string()
}

#[tauri::command]
fn emit_to_all(app: AppHandle, event: String, payload: String) {
    app.emit(&event, payload).unwrap();
}
```

---

## Command Organization

### Module Structure

```
src-tauri/src/
├── main.rs
├── lib.rs
├── commands/
│   ├── mod.rs
│   ├── auth.rs
│   ├── files.rs
│   └── settings.rs
└── models/
    ├── mod.rs
    └── user.rs
```

```rust
// src-tauri/src/commands/mod.rs
pub mod auth;
pub mod files;
pub mod settings;

// Re-export all commands
pub use auth::*;
pub use files::*;
pub use settings::*;
```

```rust
// src-tauri/src/commands/auth.rs
use crate::models::User;

#[tauri::command]
pub fn login(username: String, password: String) -> Result<User, String> {
    // Implementation
}

#[tauri::command]
pub fn logout() -> Result<(), String> {
    // Implementation
}

#[tauri::command]
pub fn get_current_user() -> Option<User> {
    // Implementation
}
```

```rust
// src-tauri/src/main.rs
mod commands;
mod models;

fn main() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![
            // Auth commands
            commands::login,
            commands::logout,
            commands::get_current_user,
            // File commands
            commands::read_file,
            commands::write_file,
            // Settings commands
            commands::get_settings,
            commands::save_settings,
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

---

## Performance Tips

### Large Data Transfer

```rust
// Bad: Returning large arrays directly
#[tauri::command]
fn get_all_logs() -> Vec<LogEntry> {
    // Could be thousands of entries
}

// Good: Pagination
#[tauri::command]
fn get_logs(page: u32, page_size: u32) -> PaginatedResult<LogEntry> {
    PaginatedResult {
        items: /* ... */,
        total: /* ... */,
        page,
        page_size,
    }
}

// Good: Streaming via events
#[tauri::command]
async fn stream_logs(window: Window) {
    for chunk in log_chunks {
        window.emit("log-chunk", chunk).unwrap();
        tokio::time::sleep(Duration::from_millis(10)).await;
    }
    window.emit("log-complete", ()).unwrap();
}
```

### Avoiding Blocking

```rust
// Bad: Blocking the main thread
#[tauri::command]
fn heavy_computation() -> i32 {
    // This blocks the event loop
    expensive_calculation()
}

// Good: Use async + spawn_blocking
#[tauri::command]
async fn heavy_computation() -> Result<i32, String> {
    let result = tokio::task::spawn_blocking(|| {
        expensive_calculation()
    })
    .await
    .map_err(|e| e.to_string())?;

    Ok(result)
}
```

---

## Testing Commands

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_greet() {
        let result = greet("World");
        assert_eq!(result, "Hello, World!");
    }

    #[test]
    fn test_calculate() {
        assert_eq!(calculate(10, 5, "add").unwrap(), 15);
        assert_eq!(calculate(10, 5, "subtract").unwrap(), 5);
        assert!(calculate(10, 0, "divide").is_err());
    }

    #[tokio::test]
    async fn test_async_command() {
        // Test async commands
    }
}
```

---

## Related Topics

- [Tauri Basics](basics.md) - Overview and setup
- [Tauri Plugins](plugins.md) - Official and custom plugins
- [Tauri Bundling](bundling.md) - Building and distributing
