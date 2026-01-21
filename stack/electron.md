# Electron (2025)

> **Last updated**: January 2026
> **Versions covered**: Electron 32+
> **Purpose**: Cross-platform desktop apps with web technologies (Chromium + Node.js)

---

## Philosophy (2025-2026)

Electron is the **established desktop framework** powering VS Code, Slack, Discord, and thousands of apps. While larger than Tauri, it offers full Node.js integration and the most mature ecosystem.

**Key philosophical shifts:**
- **Security first** — Context isolation, sandbox by default
- **Performance focus** — Reduced memory footprint per release
- **Chromium updates** — Regular security patches
- **Process isolation** — Main/renderer separation
- **Native integration** — Full OS API access
- **Mature ecosystem** — Battle-tested patterns
- **IPC security** — Validate all messages

---

## TL;DR

- Always enable `contextIsolation: true`
- Always enable `sandbox: true`
- Use `contextBridge` for safe IPC
- Validate all IPC message senders
- Never enable `nodeIntegration` in renderers
- Keep Electron updated (security patches)
- Use preload scripts for bridge
- Profile with Chrome DevTools
- Consider memory usage per process

---

## Best Practices

### Project Structure

```
my-electron-app/
├── src/
│   ├── main/                   # Main process
│   │   ├── index.ts
│   │   ├── ipc/
│   │   │   ├── handlers.ts
│   │   │   └── validators.ts
│   │   ├── windows/
│   │   │   └── main-window.ts
│   │   └── services/
│   │       └── database.ts
│   ├── preload/                # Preload scripts
│   │   └── index.ts
│   └── renderer/               # Frontend (React/Vue/etc)
│       ├── App.tsx
│       └── components/
├── electron-builder.json
├── package.json
└── tsconfig.json
```

### Main Process Setup

```typescript
// src/main/index.ts
import { app, BrowserWindow, ipcMain } from 'electron';
import path from 'path';
import { setupIpcHandlers } from './ipc/handlers';

// Handle creating/removing shortcuts on Windows when installing/uninstalling.
if (require('electron-squirrel-startup')) {
  app.quit();
}

let mainWindow: BrowserWindow | null = null;

const createWindow = () => {
  mainWindow = new BrowserWindow({
    width: 1200,
    height: 800,
    webPreferences: {
      // Security: Always use these settings
      contextIsolation: true,      // Required
      sandbox: true,               // Recommended
      nodeIntegration: false,      // Never enable in renderer
      preload: path.join(__dirname, '../preload/index.js'),
    },
  });

  // Load the app
  if (process.env.NODE_ENV === 'development') {
    mainWindow.loadURL('http://localhost:5173');
    mainWindow.webContents.openDevTools();
  } else {
    mainWindow.loadFile(path.join(__dirname, '../renderer/index.html'));
  }
};

app.whenReady().then(() => {
  setupIpcHandlers();
  createWindow();

  app.on('activate', () => {
    if (BrowserWindow.getAllWindows().length === 0) {
      createWindow();
    }
  });
});

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});
```

### Preload Script (Context Bridge)

```typescript
// src/preload/index.ts
import { contextBridge, ipcRenderer } from 'electron';

// Expose safe APIs to renderer
contextBridge.exposeInMainWorld('electronAPI', {
  // File operations
  readFile: (path: string) => ipcRenderer.invoke('file:read', path),
  writeFile: (path: string, content: string) =>
    ipcRenderer.invoke('file:write', path, content),
  selectFile: () => ipcRenderer.invoke('dialog:openFile'),

  // App info
  getVersion: () => ipcRenderer.invoke('app:version'),
  getPlatform: () => process.platform,

  // Window controls
  minimize: () => ipcRenderer.send('window:minimize'),
  maximize: () => ipcRenderer.send('window:maximize'),
  close: () => ipcRenderer.send('window:close'),

  // Events (subscribe pattern)
  onProgressUpdate: (callback: (progress: number) => void) => {
    const subscription = (_event: any, progress: number) => callback(progress);
    ipcRenderer.on('progress:update', subscription);
    return () => ipcRenderer.removeListener('progress:update', subscription);
  },
});

// TypeScript types for renderer
declare global {
  interface Window {
    electronAPI: {
      readFile: (path: string) => Promise<string>;
      writeFile: (path: string, content: string) => Promise<void>;
      selectFile: () => Promise<string | null>;
      getVersion: () => Promise<string>;
      getPlatform: string;
      minimize: () => void;
      maximize: () => void;
      close: () => void;
      onProgressUpdate: (callback: (progress: number) => void) => () => void;
    };
  }
}
```

### IPC Handlers with Validation

```typescript
// src/main/ipc/handlers.ts
import { ipcMain, dialog, app, BrowserWindow } from 'electron';
import fs from 'fs/promises';
import path from 'path';
import { validateFilePath } from './validators';

export function setupIpcHandlers() {
  // File operations
  ipcMain.handle('file:read', async (event, filePath: string) => {
    // Validate sender
    if (!validateSender(event)) {
      throw new Error('Unauthorized');
    }

    // Validate path
    if (!validateFilePath(filePath)) {
      throw new Error('Invalid file path');
    }

    return fs.readFile(filePath, 'utf-8');
  });

  ipcMain.handle('file:write', async (event, filePath: string, content: string) => {
    if (!validateSender(event)) {
      throw new Error('Unauthorized');
    }

    if (!validateFilePath(filePath)) {
      throw new Error('Invalid file path');
    }

    await fs.writeFile(filePath, content, 'utf-8');
  });

  ipcMain.handle('dialog:openFile', async (event) => {
    if (!validateSender(event)) {
      throw new Error('Unauthorized');
    }

    const result = await dialog.showOpenDialog({
      properties: ['openFile'],
      filters: [
        { name: 'Text Files', extensions: ['txt', 'md'] },
        { name: 'All Files', extensions: ['*'] },
      ],
    });

    return result.canceled ? null : result.filePaths[0];
  });

  ipcMain.handle('app:version', () => app.getVersion());

  // Window controls
  ipcMain.on('window:minimize', (event) => {
    BrowserWindow.fromWebContents(event.sender)?.minimize();
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
    BrowserWindow.fromWebContents(event.sender)?.close();
  });
}

// Validate IPC sender
function validateSender(event: Electron.IpcMainInvokeEvent): boolean {
  const senderUrl = event.senderFrame.url;

  // In development, allow localhost
  if (process.env.NODE_ENV === 'development') {
    return senderUrl.startsWith('http://localhost');
  }

  // In production, only allow file:// protocol
  return senderUrl.startsWith('file://');
}
```

### Path Validation

```typescript
// src/main/ipc/validators.ts
import path from 'path';
import { app } from 'electron';

const ALLOWED_DIRECTORIES = [
  app.getPath('documents'),
  app.getPath('downloads'),
  app.getPath('userData'),
];

export function validateFilePath(filePath: string): boolean {
  // Normalize path to prevent traversal
  const normalizedPath = path.normalize(filePath);

  // Check for path traversal attempts
  if (normalizedPath.includes('..')) {
    return false;
  }

  // Check if path is within allowed directories
  return ALLOWED_DIRECTORIES.some(allowedDir =>
    normalizedPath.startsWith(allowedDir)
  );
}
```

### Using APIs in Renderer

```typescript
// src/renderer/App.tsx
import { useState, useEffect } from 'react';

export function App() {
  const [fileContent, setFileContent] = useState('');
  const [progress, setProgress] = useState(0);

  useEffect(() => {
    // Subscribe to progress updates
    const unsubscribe = window.electronAPI.onProgressUpdate((prog) => {
      setProgress(prog);
    });

    return unsubscribe;
  }, []);

  const handleOpenFile = async () => {
    const filePath = await window.electronAPI.selectFile();
    if (filePath) {
      const content = await window.electronAPI.readFile(filePath);
      setFileContent(content);
    }
  };

  return (
    <div>
      <button onClick={handleOpenFile}>Open File</button>
      <textarea value={fileContent} readOnly />
      <progress value={progress} max={100} />
    </div>
  );
}
```

### Security Configuration

```typescript
// src/main/index.ts
import { app, session } from 'electron';

app.whenReady().then(() => {
  // Set Content Security Policy
  session.defaultSession.webRequest.onHeadersReceived((details, callback) => {
    callback({
      responseHeaders: {
        ...details.responseHeaders,
        'Content-Security-Policy': [
          "default-src 'self'",
          "script-src 'self'",
          "style-src 'self' 'unsafe-inline'",
          "img-src 'self' data: https:",
          "font-src 'self'",
        ].join('; '),
      },
    });
  });

  // Prevent new window creation
  app.on('web-contents-created', (_, contents) => {
    contents.setWindowOpenHandler(() => {
      return { action: 'deny' };
    });

    // Prevent navigation to external URLs
    contents.on('will-navigate', (event, url) => {
      const parsedUrl = new URL(url);
      if (parsedUrl.protocol !== 'file:' &&
          !url.startsWith('http://localhost')) {
        event.preventDefault();
      }
    });
  });
});
```

### Auto-Updater

```typescript
// src/main/updater.ts
import { autoUpdater } from 'electron-updater';
import { BrowserWindow } from 'electron';

export function setupAutoUpdater(mainWindow: BrowserWindow) {
  autoUpdater.autoDownload = false;
  autoUpdater.autoInstallOnAppQuit = true;

  autoUpdater.on('update-available', (info) => {
    mainWindow.webContents.send('update:available', info.version);
  });

  autoUpdater.on('download-progress', (progress) => {
    mainWindow.webContents.send('update:progress', progress.percent);
  });

  autoUpdater.on('update-downloaded', () => {
    mainWindow.webContents.send('update:ready');
  });

  // Check for updates
  autoUpdater.checkForUpdates();
}

// IPC handlers for updates
ipcMain.handle('update:download', () => {
  autoUpdater.downloadUpdate();
});

ipcMain.handle('update:install', () => {
  autoUpdater.quitAndInstall();
});
```

### Native Menus

```typescript
// src/main/menu.ts
import { Menu, shell, app, BrowserWindow } from 'electron';

export function createMenu(mainWindow: BrowserWindow) {
  const template: Electron.MenuItemConstructorOptions[] = [
    {
      label: 'File',
      submenu: [
        {
          label: 'Open',
          accelerator: 'CmdOrCtrl+O',
          click: () => mainWindow.webContents.send('menu:open'),
        },
        {
          label: 'Save',
          accelerator: 'CmdOrCtrl+S',
          click: () => mainWindow.webContents.send('menu:save'),
        },
        { type: 'separator' },
        { role: 'quit' },
      ],
    },
    {
      label: 'Edit',
      submenu: [
        { role: 'undo' },
        { role: 'redo' },
        { type: 'separator' },
        { role: 'cut' },
        { role: 'copy' },
        { role: 'paste' },
        { role: 'selectAll' },
      ],
    },
    {
      label: 'View',
      submenu: [
        { role: 'reload' },
        { role: 'forceReload' },
        { role: 'toggleDevTools' },
        { type: 'separator' },
        { role: 'resetZoom' },
        { role: 'zoomIn' },
        { role: 'zoomOut' },
        { type: 'separator' },
        { role: 'togglefullscreen' },
      ],
    },
    {
      label: 'Help',
      submenu: [
        {
          label: 'Documentation',
          click: () => shell.openExternal('https://example.com/docs'),
        },
        {
          label: 'About',
          click: () => {
            // Show about dialog
          },
        },
      ],
    },
  ];

  const menu = Menu.buildFromTemplate(template);
  Menu.setApplicationMenu(menu);
}
```

### System Tray

```typescript
// src/main/tray.ts
import { Tray, Menu, app, nativeImage, BrowserWindow } from 'electron';
import path from 'path';

let tray: Tray | null = null;

export function createTray(mainWindow: BrowserWindow) {
  const iconPath = path.join(__dirname, '../../assets/tray-icon.png');
  const icon = nativeImage.createFromPath(iconPath);

  tray = new Tray(icon);

  const contextMenu = Menu.buildFromTemplate([
    {
      label: 'Show App',
      click: () => {
        mainWindow.show();
        mainWindow.focus();
      },
    },
    {
      label: 'Hide App',
      click: () => mainWindow.hide(),
    },
    { type: 'separator' },
    {
      label: 'Quit',
      click: () => app.quit(),
    },
  ]);

  tray.setToolTip('My Electron App');
  tray.setContextMenu(contextMenu);

  tray.on('click', () => {
    mainWindow.isVisible() ? mainWindow.hide() : mainWindow.show();
  });
}
```

### Electron Builder Configuration

```json
// electron-builder.json
{
  "appId": "com.mycompany.myapp",
  "productName": "My App",
  "directories": {
    "output": "dist",
    "buildResources": "build"
  },
  "files": [
    "out/**/*",
    "package.json"
  ],
  "mac": {
    "category": "public.app-category.productivity",
    "target": ["dmg", "zip"],
    "hardenedRuntime": true,
    "gatekeeperAssess": false
  },
  "win": {
    "target": ["nsis", "portable"],
    "signingHashAlgorithms": ["sha256"]
  },
  "linux": {
    "target": ["AppImage", "deb"],
    "category": "Utility"
  },
  "publish": {
    "provider": "github",
    "releaseType": "release"
  }
}
```

---

## Anti-Patterns

### ❌ Enabling nodeIntegration

**Why it's bad**: XSS can execute arbitrary Node.js code.

```typescript
// ❌ DON'T — Never enable nodeIntegration
new BrowserWindow({
  webPreferences: {
    nodeIntegration: true,  // DANGEROUS!
    contextIsolation: false, // DANGEROUS!
  },
});

// ✅ DO — Always use contextIsolation
new BrowserWindow({
  webPreferences: {
    nodeIntegration: false,
    contextIsolation: true,
    sandbox: true,
    preload: path.join(__dirname, 'preload.js'),
  },
});
```

### ❌ Not Validating IPC Senders

**Why it's bad**: Malicious frames can send IPC messages.

```typescript
// ❌ DON'T — Trust all IPC messages
ipcMain.handle('delete-file', async (event, path) => {
  await fs.unlink(path);  // Anyone can delete files!
});

// ✅ DO — Validate sender
ipcMain.handle('delete-file', async (event, path) => {
  // Check sender origin
  const senderUrl = event.senderFrame.url;
  if (!senderUrl.startsWith('file://')) {
    throw new Error('Unauthorized');
  }

  // Validate path
  if (!isPathAllowed(path)) {
    throw new Error('Path not allowed');
  }

  await fs.unlink(path);
});
```

### ❌ Loading Remote Content Without Protection

**Why it's bad**: Remote content can be malicious.

```typescript
// ❌ DON'T — Load external URLs without protection
mainWindow.loadURL('https://untrusted-site.com');

// ✅ DO — If you must load external content, use partition
const externalWindow = new BrowserWindow({
  webPreferences: {
    partition: 'external',
    nodeIntegration: false,
    contextIsolation: true,
    sandbox: true,
    // No preload — no access to APIs
  },
});
```

### ❌ Exposing Too Much via contextBridge

**Why it's bad**: Increases attack surface.

```typescript
// ❌ DON'T — Expose raw Node.js APIs
contextBridge.exposeInMainWorld('node', {
  fs: require('fs'),  // Full filesystem access!
  exec: require('child_process').exec,  // Shell access!
});

// ✅ DO — Expose specific, validated functions
contextBridge.exposeInMainWorld('api', {
  readConfig: () => ipcRenderer.invoke('config:read'),
  saveConfig: (data: ConfigData) => ipcRenderer.invoke('config:save', data),
});
```

### ❌ Ignoring Navigation Events

**Why it's bad**: Attackers can redirect to malicious sites.

```typescript
// ❌ DON'T — Allow unrestricted navigation
// (Default behavior)

// ✅ DO — Restrict navigation
app.on('web-contents-created', (_, contents) => {
  contents.on('will-navigate', (event, url) => {
    const allowed = ['file://', 'http://localhost'];
    if (!allowed.some(prefix => url.startsWith(prefix))) {
      event.preventDefault();
    }
  });
});
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 32.0 | Aug 2025 | Chromium 128, Node.js 20.16, V8 12.8 |
| 31.0 | May 2025 | Performance improvements, security fixes |
| 30.0 | Feb 2025 | New APIs, Chromium 126 |

### Recent Security Focus

- **Sandbox by default**: New apps should always use sandbox
- **Context isolation required**: No longer optional for security
- **IPC validation**: Framework for validating senders
- **CSP enforcement**: Better tools for content security
- **Regular Chromium updates**: Critical for security patches

---

## Quick Reference

| Process | Purpose |
|---------|---------|
| Main | Node.js, system access, window management |
| Renderer | Web content, UI (sandboxed) |
| Preload | Bridge between main and renderer |

| Security Setting | Recommended |
|------------------|-------------|
| `contextIsolation` | `true` (required) |
| `sandbox` | `true` |
| `nodeIntegration` | `false` |
| `webSecurity` | `true` |
| `allowRunningInsecureContent` | `false` |

| IPC Method | Use Case |
|------------|----------|
| `ipcMain.handle` | Request/response (async) |
| `ipcMain.on` | Fire-and-forget |
| `webContents.send` | Main → renderer |
| `ipcRenderer.invoke` | Renderer → main (async) |
| `ipcRenderer.send` | Renderer → main (sync) |

| Command | Purpose |
|---------|---------|
| `npm run start` | Start in development |
| `npm run build` | Build for production |
| `npm run make` | Create distributables |
| `npm run publish` | Publish to update server |

---

## When to Use Electron

| Great For | Not Ideal For |
|-----------|---------------|
| Full Node.js integration | Bundle size sensitive |
| Mature ecosystem needed | Mobile apps |
| VS Code-like apps | Simple utilities |
| Cross-platform consistency | Memory constrained |
| Rapid prototyping | Security-critical (consider Tauri) |

---

## Resources

- [Official Electron Documentation](https://www.electronjs.org/docs/latest/)
- [Electron Security Guide](https://www.electronjs.org/docs/latest/tutorial/security)
- [Electron Performance Tips](https://www.electronjs.org/docs/latest/tutorial/performance)
- [Electron Builder](https://www.electron.build/)
- [Electron Forge](https://www.electronforge.io/)
- [Electron Security Vulnerabilities](https://www.cvedetails.com/vulnerability-list/vendor_id-17824/product_id-44696/Electronjs-Electron.html)
