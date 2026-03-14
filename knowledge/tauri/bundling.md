# Tauri Bundling & Distribution

> Official Documentation: https://tauri.app/distribute/

## Overview

Tauri produces native installers for each platform. The bundler handles code signing, auto-updates, and platform-specific packaging formats.

---

## Build Commands

```bash
# Build for current platform
npm run tauri build

# Build with debug symbols
npm run tauri build --debug

# Build for specific target
npm run tauri build --target x86_64-pc-windows-msvc
npm run tauri build --target aarch64-apple-darwin
npm run tauri build --target x86_64-unknown-linux-gnu

# Build specific bundle type
npm run tauri build --bundles deb
npm run tauri build --bundles msi
npm run tauri build --bundles dmg

# Skip frontend build
npm run tauri build --no-bundle
```

---

## Bundle Configuration

```json
// tauri.conf.json
{
  "bundle": {
    "active": true,
    "targets": "all",
    "identifier": "com.company.myapp",
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/128x128@2x.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ],
    "resources": ["resources/*"],
    "copyright": "Copyright (c) 2024 My Company",
    "category": "Productivity",
    "shortDescription": "A short description of my app",
    "longDescription": "A longer description...",
    "externalBin": ["binaries/ffmpeg"],
    "windows": {
      "certificateThumbprint": null,
      "digestAlgorithm": "sha256",
      "timestampUrl": "http://timestamp.digicert.com",
      "wix": null,
      "nsis": {
        "license": "LICENSE.txt"
      }
    },
    "macOS": {
      "entitlements": null,
      "exceptionDomain": "",
      "frameworks": [],
      "providerShortName": null,
      "signingIdentity": null,
      "minimumSystemVersion": "10.13"
    },
    "linux": {
      "appimage": {
        "bundleMediaFramework": false
      },
      "deb": {
        "depends": ["libwebkit2gtk-4.1-0"],
        "files": {}
      }
    }
  }
}
```

---

## Platform-Specific Bundles

### Windows

| Format | Extension | Description |
|--------|-----------|-------------|
| MSI | `.msi` | Windows Installer package |
| NSIS | `.exe` | NSIS installer (recommended) |

```json
{
  "bundle": {
    "targets": ["msi", "nsis"],
    "windows": {
      "nsis": {
        "installerIcon": "icons/icon.ico",
        "installMode": "perUser",
        "languages": ["English", "Italian"],
        "displayLanguageSelector": true
      }
    }
  }
}
```

### macOS

| Format | Extension | Description |
|--------|-----------|-------------|
| DMG | `.dmg` | Disk image with drag-to-install |
| App Bundle | `.app` | Application bundle |

```json
{
  "bundle": {
    "targets": ["dmg", "app"],
    "macOS": {
      "minimumSystemVersion": "10.13",
      "dmg": {
        "appPosition": { "x": 180, "y": 170 },
        "applicationFolderPosition": { "x": 480, "y": 170 },
        "windowSize": { "width": 660, "height": 400 }
      }
    }
  }
}
```

### Linux

| Format | Extension | Description |
|--------|-----------|-------------|
| AppImage | `.AppImage` | Portable, no installation |
| DEB | `.deb` | Debian/Ubuntu package |
| RPM | `.rpm` | Red Hat/Fedora package |

```json
{
  "bundle": {
    "targets": ["appimage", "deb"],
    "linux": {
      "appimage": {
        "bundleMediaFramework": true
      },
      "deb": {
        "depends": ["libwebkit2gtk-4.1-0", "libssl3"],
        "section": "utils",
        "priority": "optional"
      }
    }
  }
}
```

---

## Code Signing

### Windows (Authenticode)

```bash
# Set environment variables
export TAURI_SIGNING_PRIVATE_KEY="path/to/private-key.pfx"
export TAURI_SIGNING_PRIVATE_KEY_PASSWORD="your-password"

# Or use certificate thumbprint (Windows Certificate Store)
```

```json
{
  "bundle": {
    "windows": {
      "certificateThumbprint": "YOUR_CERTIFICATE_THUMBPRINT",
      "digestAlgorithm": "sha256",
      "timestampUrl": "http://timestamp.digicert.com"
    }
  }
}
```

### macOS (Apple Developer)

```bash
# Set signing identity
export APPLE_SIGNING_IDENTITY="Developer ID Application: Your Name (TEAM_ID)"

# For notarization
export APPLE_ID="your@email.com"
export APPLE_PASSWORD="app-specific-password"
export APPLE_TEAM_ID="TEAM_ID"
```

```json
{
  "bundle": {
    "macOS": {
      "signingIdentity": "Developer ID Application: Your Name (TEAM_ID)",
      "providerShortName": "TEAM_ID",
      "entitlements": "entitlements.plist"
    }
  }
}
```

**entitlements.plist:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.cs.allow-jit</key>
    <true/>
    <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
    <true/>
    <key>com.apple.security.cs.disable-library-validation</key>
    <true/>
</dict>
</plist>
```

---

## Auto-Updates

### Configuration

```json
// tauri.conf.json
{
  "plugins": {
    "updater": {
      "active": true,
      "pubkey": "dW50cnVzdGVkIGNvbW1lbnQ6IG1pbmlzaWduIHB1YmxpYyBrZXk6...",
      "endpoints": [
        "https://releases.myapp.com/{{target}}/{{arch}}/{{current_version}}"
      ],
      "windows": {
        "installMode": "passive"
      }
    }
  }
}
```

### Generate Keys

```bash
# Generate signing keys
npm run tauri signer generate -- -w ~/.tauri/myapp.key

# This outputs:
# - Private key: ~/.tauri/myapp.key
# - Public key: ~/.tauri/myapp.key.pub
```

### Update Server Response

The endpoint should return JSON:

```json
{
  "version": "1.2.0",
  "notes": "Bug fixes and improvements",
  "pub_date": "2024-01-15T12:00:00Z",
  "platforms": {
    "darwin-x86_64": {
      "signature": "dW50cnVzdGVkIGNvbW1lbnQ6IHNpZ25hdHVyZSBmcm9tIHRhdXJpIHNlY3JldCBrZXkK...",
      "url": "https://releases.myapp.com/darwin/x86_64/myapp_1.2.0_x64.app.tar.gz"
    },
    "darwin-aarch64": {
      "signature": "...",
      "url": "https://releases.myapp.com/darwin/aarch64/myapp_1.2.0_aarch64.app.tar.gz"
    },
    "linux-x86_64": {
      "signature": "...",
      "url": "https://releases.myapp.com/linux/x86_64/myapp_1.2.0_amd64.AppImage.tar.gz"
    },
    "windows-x86_64": {
      "signature": "...",
      "url": "https://releases.myapp.com/windows/x86_64/myapp_1.2.0_x64-setup.nsis.zip"
    }
  }
}
```

### Sign Update Artifacts

```bash
# Sign the bundle
npm run tauri signer sign -- \
  -k ~/.tauri/myapp.key \
  -p "password" \
  target/release/bundle/macos/MyApp.app.tar.gz

# Output: signature string to use in update response
```

---

## GitHub Actions CI/CD

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    permissions:
      contents: write
    strategy:
      fail-fast: false
      matrix:
        include:
          - platform: 'macos-latest'
            args: '--target aarch64-apple-darwin'
          - platform: 'macos-latest'
            args: '--target x86_64-apple-darwin'
          - platform: 'ubuntu-22.04'
            args: ''
          - platform: 'windows-latest'
            args: ''

    runs-on: ${{ matrix.platform }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install Rust
        uses: dtolnay/rust-action@stable
        with:
          targets: ${{ matrix.platform == 'macos-latest' && 'aarch64-apple-darwin,x86_64-apple-darwin' || '' }}

      - name: Install dependencies (ubuntu)
        if: matrix.platform == 'ubuntu-22.04'
        run: |
          sudo apt-get update
          sudo apt-get install -y libwebkit2gtk-4.1-dev libappindicator3-dev librsvg2-dev patchelf

      - name: Install frontend dependencies
        run: npm install

      - name: Build Tauri app
        uses: tauri-apps/tauri-action@v0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          TAURI_SIGNING_PRIVATE_KEY: ${{ secrets.TAURI_PRIVATE_KEY }}
          TAURI_SIGNING_PRIVATE_KEY_PASSWORD: ${{ secrets.TAURI_KEY_PASSWORD }}
          APPLE_CERTIFICATE: ${{ secrets.APPLE_CERTIFICATE }}
          APPLE_CERTIFICATE_PASSWORD: ${{ secrets.APPLE_CERTIFICATE_PASSWORD }}
          APPLE_SIGNING_IDENTITY: ${{ secrets.APPLE_SIGNING_IDENTITY }}
          APPLE_ID: ${{ secrets.APPLE_ID }}
          APPLE_PASSWORD: ${{ secrets.APPLE_PASSWORD }}
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
        with:
          tagName: v__VERSION__
          releaseName: 'My App v__VERSION__'
          releaseBody: 'See the assets to download this version and install.'
          releaseDraft: true
          prerelease: false
          args: ${{ matrix.args }}
```

---

## Icons

### Generate Icons

```bash
# From a 1024x1024 PNG source
npm run tauri icon ./app-icon.png

# Output structure:
# src-tauri/icons/
# ├── 32x32.png
# ├── 128x128.png
# ├── 128x128@2x.png
# ├── icon.icns (macOS)
# ├── icon.ico (Windows)
# └── icon.png
```

### Icon Requirements

| Platform | Format | Sizes |
|----------|--------|-------|
| Windows | ICO | 16, 24, 32, 48, 64, 128, 256 |
| macOS | ICNS | 16, 32, 64, 128, 256, 512, 1024 |
| Linux | PNG | 32, 128, 256, 512 |

---

## Sidecar Binaries

Include external binaries with your app:

```json
{
  "bundle": {
    "externalBin": [
      "binaries/ffmpeg",
      "binaries/imagemagick"
    ]
  }
}
```

**Directory structure:**
```
src-tauri/
├── binaries/
│   ├── ffmpeg-x86_64-pc-windows-msvc.exe
│   ├── ffmpeg-x86_64-apple-darwin
│   ├── ffmpeg-aarch64-apple-darwin
│   └── ffmpeg-x86_64-unknown-linux-gnu
```

**Usage in Rust:**
```rust
use tauri_plugin_shell::ShellExt;

#[tauri::command]
async fn run_ffmpeg(app: tauri::AppHandle) -> Result<String, String> {
    let sidecar = app.shell().sidecar("ffmpeg").map_err(|e| e.to_string())?;

    let output = sidecar
        .args(["-version"])
        .output()
        .await
        .map_err(|e| e.to_string())?;

    Ok(String::from_utf8_lossy(&output.stdout).to_string())
}
```

---

## Resources

Include additional files in the bundle:

```json
{
  "bundle": {
    "resources": [
      "resources/*",
      "data/config.json",
      "assets/**/*"
    ]
  }
}
```

**Access in Rust:**
```rust
use tauri::Manager;

#[tauri::command]
fn get_resource_path(app: tauri::AppHandle) -> Result<String, String> {
    let resource_path = app.path()
        .resource_dir()
        .map_err(|e| e.to_string())?
        .join("config.json");

    Ok(resource_path.to_string_lossy().to_string())
}
```

---

## Build Optimization

### Reduce Binary Size

```toml
# Cargo.toml
[profile.release]
panic = "abort"
codegen-units = 1
lto = true
opt-level = "s"  # or "z" for smallest size
strip = true
```

### Strip Debug Symbols

```bash
# Linux
strip target/release/my-app

# macOS
strip -x target/release/bundle/macos/MyApp.app/Contents/MacOS/my-app
```

### Expected Sizes

| Component | Size |
|-----------|------|
| Tauri runtime | ~3-5 MB |
| Rust backend | ~1-5 MB |
| Frontend (bundled) | Variable |
| **Total (typical)** | **~10-30 MB** |

---

## Troubleshooting

### Windows Build Fails

```bash
# Install Visual Studio Build Tools
winget install Microsoft.VisualStudio.2022.BuildTools

# Select "Desktop development with C++"
```

### macOS Notarization Fails

```bash
# Check entitlements
codesign -d --entitlements :- /path/to/MyApp.app

# Verify signing
codesign -vvv --deep --strict /path/to/MyApp.app
spctl -a -vvv /path/to/MyApp.app
```

### Linux WebKitGTK Missing

```bash
# Ubuntu/Debian
sudo apt install libwebkit2gtk-4.1-dev

# Fedora
sudo dnf install webkit2gtk4.1-devel

# Arch
sudo pacman -S webkit2gtk-4.1
```

---

## Related Topics

- [Tauri Basics](basics.md) - Overview and setup
- [Tauri Commands](commands.md) - Custom Rust commands
- [Tauri Plugins](plugins.md) - Official and custom plugins
