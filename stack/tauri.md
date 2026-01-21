# Tauri 2.0 (2025)

> **Last updated**: January 2026
> **Versions covered**: Tauri 2.x
> **Purpose**: Secure, lightweight desktop and mobile apps with Rust backend and web frontend

---

## Philosophy (2025-2026)

Tauri 2.0 is the **secure-by-default desktop framework** that delivers Rust's safety guarantees with binaries 95% smaller than Electron. It's the bridge between web technologies and native performance.

**Key philosophical shifts:**
- **Principle of least privilege** — Only access OS APIs when needed
- **95% smaller binaries** — Uses system WebView, not bundled Chromium
- **Rust backend** — Memory safety and performance
- **Mobile support** — Android and iOS in Tauri 2.0
- **Trust boundaries** — Clear separation between frontend and backend
- **Web-first frontend** — React, Vue, Svelte, or vanilla JS
- **Security audited** — Every major release audited

---

## TL;DR

- Use Tauri commands for Rust ↔ TypeScript communication
- Enable isolation pattern for high-security apps
- Use type-safe IPC with tauri-specta
- Configure explicit permissions in `tauri.conf.json`
- Keep frontend and backend state separate
- Use events for async communication
- Mobile support is production-ready in 2.0
- Bundle size ~600KB vs Electron's ~150MB

---

## Best Practices

### Project Structure

```
my-tauri-app/
├── src/                        # Frontend (React/Vue/Svelte)
│   ├── components/
│   ├── hooks/
│   ├── stores/
│   └── App.tsx
├── src-tauri/                  # Rust backend
│   ├── src/
│   │   ├── main.rs
│   │   ├── lib.rs
│   │   ├── commands/           # Tauri commands
│   │   │   ├── mod.rs
│   │   │   ├── files.rs
│   │   │   └── system.rs
│   │   └── state/              # App state
│   │       └── mod.rs
│   ├── Cargo.toml
│   ├── tauri.conf.json
│   └── capabilities/           # Permission configs
│       └── default.json
├── package.json
└── vite.config.ts
```

### Configuration (tauri.conf.json)

```json
{
  "$schema": "https://tauri.app/v2/tauri.conf.json/schema.json",
  "productName": "My App",
  "version": "1.0.0",
  "identifier": "com.mycompany.myapp",
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
        "minWidth": 800,
        "minHeight": 600,
        "resizable": true,
        "center": true
      }
    ],
    "security": {
      "csp": "default-src 'self'; script-src 'self'"
    }
  },
  "bundle": {
    "active": true,
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ]
  }
}
```

### Basic Tauri Command

```rust
// src-tauri/src/commands/files.rs
use std::fs;
use std::path::PathBuf;
use tauri::command;

#[derive(serde::Serialize)]
pub struct FileInfo {
    name: String,
    size: u64,
    is_directory: bool,
}

#[command]
pub fn read_directory(path: String) -> Result<Vec<FileInfo>, String> {
    let entries = fs::read_dir(&path)
        .map_err(|e| format!("Failed to read directory: {}", e))?;

    let files: Vec<FileInfo> = entries
        .filter_map(|entry| {
            let entry = entry.ok()?;
            let metadata = entry.metadata().ok()?;
            Some(FileInfo {
                name: entry.file_name().to_string_lossy().to_string(),
                size: metadata.len(),
                is_directory: metadata.is_dir(),
            })
        })
        .collect();

    Ok(files)
}

#[command]
pub fn read_file(path: String) -> Result<String, String> {
    fs::read_to_string(&path)
        .map_err(|e| format!("Failed to read file: {}", e))
}

#[command]
pub fn write_file(path: String, content: String) -> Result<(), String> {
    fs::write(&path, content)
        .map_err(|e| format!("Failed to write file: {}", e))
}
```

### Register Commands (main.rs)

```rust
// src-tauri/src/main.rs
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

mod commands;

use commands::files;

fn main() {
    tauri::Builder::default()
        .plugin(tauri_plugin_shell::init())
        .invoke_handler(tauri::generate_handler![
            files::read_directory,
            files::read_file,
            files::write_file,
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### Invoke Commands from Frontend

```typescript
// src/hooks/useFiles.ts
import { invoke } from '@tauri-apps/api/core';

interface FileInfo {
  name: string;
  size: number;
  is_directory: boolean;
}

export async function readDirectory(path: string): Promise<FileInfo[]> {
  return invoke<FileInfo[]>('read_directory', { path });
}

export async function readFile(path: string): Promise<string> {
  return invoke<string>('read_file', { path });
}

export async function writeFile(path: string, content: string): Promise<void> {
  return invoke('write_file', { path, content });
}
```

### Type-Safe Commands with tauri-specta

```rust
// src-tauri/src/commands/mod.rs
use specta::Type;
use tauri_specta::{collect_commands, Builder};

#[derive(serde::Serialize, Type)]
pub struct User {
    id: i32,
    name: String,
    email: String,
}

#[tauri::command]
#[specta::specta]
pub fn get_user(id: i32) -> Result<User, String> {
    // Fetch user from database
    Ok(User {
        id,
        name: "John Doe".to_string(),
        email: "john@example.com".to_string(),
    })
}

#[tauri::command]
#[specta::specta]
pub fn create_user(name: String, email: String) -> Result<User, String> {
    // Create user in database
    Ok(User {
        id: 1,
        name,
        email,
    })
}

// Generate TypeScript bindings
pub fn generate_bindings() {
    let builder = Builder::<tauri::Wry>::new()
        .commands(collect_commands![get_user, create_user]);

    builder
        .export(
            specta_typescript::Typescript::default(),
            "../src/bindings.ts",
        )
        .expect("Failed to export typescript bindings");
}
```

### State Management

```rust
// src-tauri/src/state/mod.rs
use std::sync::Mutex;
use tauri::State;

pub struct AppState {
    pub db: Mutex<Database>,
    pub config: Config,
}

pub struct Database {
    connection_string: String,
}

impl Database {
    pub fn new(connection_string: &str) -> Self {
        Self {
            connection_string: connection_string.to_string(),
        }
    }

    pub fn query(&self, sql: &str) -> Result<Vec<String>, String> {
        // Execute query
        Ok(vec![])
    }
}

#[derive(Clone)]
pub struct Config {
    pub api_url: String,
    pub debug: bool,
}

// In main.rs
fn main() {
    let state = AppState {
        db: Mutex::new(Database::new("sqlite://app.db")),
        config: Config {
            api_url: "https://api.example.com".to_string(),
            debug: cfg!(debug_assertions),
        },
    };

    tauri::Builder::default()
        .manage(state)
        .invoke_handler(tauri::generate_handler![query_database])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}

#[tauri::command]
fn query_database(
    state: State<AppState>,
    sql: String,
) -> Result<Vec<String>, String> {
    let db = state.db.lock().map_err(|_| "Failed to lock database")?;
    db.query(&sql)
}
```

### Event System

```rust
// src-tauri/src/main.rs
use tauri::{AppHandle, Emitter, Manager};
use std::thread;
use std::time::Duration;

#[derive(Clone, serde::Serialize)]
struct ProgressPayload {
    progress: u32,
    message: String,
}

#[tauri::command]
fn start_long_task(app: AppHandle) -> Result<(), String> {
    thread::spawn(move || {
        for i in 0..=100 {
            thread::sleep(Duration::from_millis(50));

            app.emit("progress", ProgressPayload {
                progress: i,
                message: format!("Processing... {}%", i),
            }).unwrap();
        }

        app.emit("task-complete", "Done!").unwrap();
    });

    Ok(())
}
```

```typescript
// src/components/LongTask.tsx
import { listen } from '@tauri-apps/api/event';
import { invoke } from '@tauri-apps/api/core';
import { useEffect, useState } from 'react';

interface ProgressPayload {
  progress: number;
  message: string;
}

export function LongTask() {
  const [progress, setProgress] = useState(0);
  const [message, setMessage] = useState('');

  useEffect(() => {
    const unlistenProgress = listen<ProgressPayload>('progress', (event) => {
      setProgress(event.payload.progress);
      setMessage(event.payload.message);
    });

    const unlistenComplete = listen<string>('task-complete', (event) => {
      setMessage(event.payload);
    });

    return () => {
      unlistenProgress.then((fn) => fn());
      unlistenComplete.then((fn) => fn());
    };
  }, []);

  const startTask = async () => {
    await invoke('start_long_task');
  };

  return (
    <div>
      <button onClick={startTask}>Start Task</button>
      <progress value={progress} max={100} />
      <p>{message}</p>
    </div>
  );
}
```

### Permissions (Capabilities)

```json
// src-tauri/capabilities/default.json
{
  "$schema": "https://tauri.app/v2/schema/capabilities.json",
  "identifier": "default",
  "description": "Default capabilities for the app",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "shell:allow-open",
    {
      "identifier": "fs:allow-read",
      "allow": [
        { "path": "$HOME/**" },
        { "path": "$DOCUMENT/**" }
      ]
    },
    {
      "identifier": "fs:allow-write",
      "allow": [
        { "path": "$DOCUMENT/**" }
      ]
    },
    "dialog:allow-open",
    "dialog:allow-save",
    "notification:default"
  ]
}
```

### File Dialogs

```rust
// src-tauri/src/commands/dialogs.rs
use tauri_plugin_dialog::{DialogExt, FilePath};

#[tauri::command]
async fn open_file_dialog(app: tauri::AppHandle) -> Result<Option<String>, String> {
    let file = app.dialog()
        .file()
        .add_filter("Text Files", &["txt", "md"])
        .add_filter("All Files", &["*"])
        .blocking_pick_file();

    Ok(file.map(|f| f.to_string()))
}

#[tauri::command]
async fn save_file_dialog(app: tauri::AppHandle) -> Result<Option<String>, String> {
    let file = app.dialog()
        .file()
        .add_filter("Text Files", &["txt"])
        .set_file_name("document.txt")
        .blocking_save_file();

    Ok(file.map(|f| f.to_string()))
}
```

### Multi-Window Support

```rust
// src-tauri/src/main.rs
use tauri::{Manager, WebviewUrl, WebviewWindowBuilder};

#[tauri::command]
async fn open_settings_window(app: tauri::AppHandle) -> Result<(), String> {
    let _settings_window = WebviewWindowBuilder::new(
        &app,
        "settings",
        WebviewUrl::App("settings.html".into())
    )
    .title("Settings")
    .inner_size(600.0, 400.0)
    .resizable(false)
    .center()
    .build()
    .map_err(|e| e.to_string())?;

    Ok(())
}
```

### System Tray

```rust
// src-tauri/src/main.rs
use tauri::{
    menu::{Menu, MenuItem},
    tray::{MouseButton, MouseButtonState, TrayIconBuilder, TrayIconEvent},
    Manager,
};

fn main() {
    tauri::Builder::default()
        .setup(|app| {
            let quit = MenuItem::with_id(app, "quit", "Quit", true, None::<&str>)?;
            let show = MenuItem::with_id(app, "show", "Show Window", true, None::<&str>)?;
            let menu = Menu::with_items(app, &[&show, &quit])?;

            let _tray = TrayIconBuilder::new()
                .icon(app.default_window_icon().unwrap().clone())
                .menu(&menu)
                .on_menu_event(|app, event| match event.id.as_ref() {
                    "quit" => {
                        app.exit(0);
                    }
                    "show" => {
                        if let Some(window) = app.get_webview_window("main") {
                            window.show().unwrap();
                            window.set_focus().unwrap();
                        }
                    }
                    _ => {}
                })
                .on_tray_icon_event(|tray, event| {
                    if let TrayIconEvent::Click {
                        button: MouseButton::Left,
                        button_state: MouseButtonState::Up,
                        ..
                    } = event
                    {
                        let app = tray.app_handle();
                        if let Some(window) = app.get_webview_window("main") {
                            window.show().unwrap();
                            window.set_focus().unwrap();
                        }
                    }
                })
                .build(app)?;

            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

---

## Anti-Patterns

### ❌ Exposing Too Many Commands

**Why it's bad**: Increases attack surface.

```rust
// ❌ DON'T — Expose dangerous commands
#[tauri::command]
fn execute_shell(command: String) -> Result<String, String> {
    std::process::Command::new("sh")
        .arg("-c")
        .arg(&command)  // Arbitrary command execution!
        .output()
        .map(|o| String::from_utf8_lossy(&o.stdout).to_string())
        .map_err(|e| e.to_string())
}

// ✅ DO — Specific, validated commands
#[tauri::command]
fn open_in_browser(url: String) -> Result<(), String> {
    // Validate URL scheme
    if !url.starts_with("https://") && !url.starts_with("http://") {
        return Err("Invalid URL scheme".to_string());
    }

    open::that(&url).map_err(|e| e.to_string())
}
```

### ❌ Disabled CSP

**Why it's bad**: XSS attacks can execute arbitrary code.

```json
// ❌ DON'T — No CSP
{
  "app": {
    "security": {}
  }
}

// ✅ DO — Strict CSP
{
  "app": {
    "security": {
      "csp": "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'"
    }
  }
}
```

### ❌ Trusting Frontend Input

**Why it's bad**: Frontend can be manipulated.

```rust
// ❌ DON'T — Trust frontend paths
#[tauri::command]
fn read_any_file(path: String) -> Result<String, String> {
    std::fs::read_to_string(&path)  // Can read /etc/passwd!
        .map_err(|e| e.to_string())
}

// ✅ DO — Validate and restrict paths
#[tauri::command]
fn read_app_file(
    app: tauri::AppHandle,
    filename: String,
) -> Result<String, String> {
    // Validate filename (no path traversal)
    if filename.contains("..") || filename.contains('/') {
        return Err("Invalid filename".to_string());
    }

    // Restrict to app data directory
    let app_dir = app.path().app_data_dir()
        .map_err(|_| "Failed to get app directory")?;
    let file_path = app_dir.join(&filename);

    std::fs::read_to_string(file_path)
        .map_err(|e| e.to_string())
}
```

### ❌ Blocking the Main Thread

**Why it's bad**: UI freezes during long operations.

```rust
// ❌ DON'T — Block main thread
#[tauri::command]
fn heavy_computation() -> i64 {
    // This blocks the main thread!
    (0..1_000_000_000).sum()
}

// ✅ DO — Use async or spawn blocking
#[tauri::command]
async fn heavy_computation() -> Result<i64, String> {
    tokio::task::spawn_blocking(|| {
        (0..1_000_000_000i64).sum()
    })
    .await
    .map_err(|e| e.to_string())
}
```

### ❌ Not Using Isolation Pattern

**Why it's bad**: Untrusted frontend code can access IPC.

```json
// ❌ DON'T — Disabled isolation for apps with external content
{
  "app": {
    "security": {
      "pattern": { "use": "brownfield" }
    }
  }
}

// ✅ DO — Enable isolation for security-critical apps
{
  "app": {
    "security": {
      "pattern": {
        "use": "isolation",
        "options": {
          "dir": "../isolation"
        }
      }
    }
  }
}
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 2.0 | Oct 2024 | Mobile support, new plugin system, capabilities |
| 2.1 | Jan 2025 | Improved mobile stability, better error messages |
| 2.2 | Apr 2025 | Performance improvements, new APIs |

### Tauri 2.0 Key Features

- **Mobile Support**: Android and iOS production-ready
- **Capabilities System**: Fine-grained permissions per window
- **Plugin Ecosystem**: Official plugins for shell, dialog, fs, etc.
- **tauri-specta**: Type-safe TypeScript bindings
- **Improved IPC**: Better serialization performance
- **Security Audited**: Full audit for 2.0 release

---

## Quick Reference

| Concept | Description |
|---------|-------------|
| Command | Rust function callable from frontend |
| Event | Async message between frontend/backend |
| State | Shared app state managed by Tauri |
| Plugin | Reusable functionality modules |
| Capability | Permission set for a window |

| Plugin | Purpose |
|--------|---------|
| `tauri-plugin-shell` | Run shell commands |
| `tauri-plugin-dialog` | File/folder dialogs |
| `tauri-plugin-fs` | File system access |
| `tauri-plugin-http` | HTTP client |
| `tauri-plugin-notification` | System notifications |
| `tauri-plugin-store` | Persistent key-value store |

| Command | Purpose |
|---------|---------|
| `npm run tauri dev` | Start development |
| `npm run tauri build` | Build for production |
| `npm run tauri android dev` | Android development |
| `npm run tauri ios dev` | iOS development |
| `cargo tauri icon` | Generate app icons |

---

## When to Use Tauri

| Great For | Not Ideal For |
|-----------|---------------|
| Desktop apps | Apps needing full Node.js |
| Cross-platform (desktop + mobile) | Heavily browser-dependent features |
| Security-critical apps | Quick prototypes (Electron faster) |
| Small bundle size needed | Complex native UI |
| Rust backend logic | Teams unfamiliar with Rust |

---

## Resources

- [Official Tauri 2.0 Documentation](https://v2.tauri.app/)
- [Tauri Security Guide](https://v2.tauri.app/security/)
- [Tauri + React Template](https://github.com/dannysmith/tauri-template)
- [Tauri Plugin Ecosystem](https://v2.tauri.app/plugin/)
- [Building Desktop Apps with Tauri 2025](https://www.plutenium.com/blog/building-desktop-apps-with-rust-and-tauri)
- [Tauri IPC Communication](https://v2.tauri.app/concept/inter-process-communication/)
