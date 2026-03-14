# Electron Packaging & Distribution

## Overview

Packaging an Electron application involves bundling your source code with Electron binaries and creating platform-specific installers. The two main tools are **Electron Forge** (officially recommended) and **electron-builder**.

## Electron Forge

### Installation

```bash
# New project with Forge
npm init electron-app@latest my-app -- --template=vite-typescript

# Add to existing project
npm install --save-dev @electron-forge/cli
npx electron-forge import
```

### Configuration

```typescript
// forge.config.ts
import type { ForgeConfig } from '@electron-forge/shared-types';
import { MakerSquirrel } from '@electron-forge/maker-squirrel';
import { MakerZIP } from '@electron-forge/maker-zip';
import { MakerDeb } from '@electron-forge/maker-deb';
import { MakerRpm } from '@electron-forge/maker-rpm';
import { MakerDMG } from '@electron-forge/maker-dmg';
import { VitePlugin } from '@electron-forge/plugin-vite';

const config: ForgeConfig = {
  packagerConfig: {
    asar: true,
    icon: './resources/icon',
    appBundleId: 'com.company.myapp',
    appCategoryType: 'public.app-category.productivity',

    // macOS code signing
    osxSign: {
      identity: 'Developer ID Application: Company Name (TEAMID)',
      hardenedRuntime: true,
      entitlements: './build/entitlements.mac.plist',
      'entitlements-inherit': './build/entitlements.mac.plist',
    },

    // macOS notarization
    osxNotarize: {
      appleId: process.env.APPLE_ID!,
      appleIdPassword: process.env.APPLE_PASSWORD!,
      teamId: process.env.APPLE_TEAM_ID!,
    },
  },

  makers: [
    // Windows
    new MakerSquirrel({
      name: 'MyApp',
      setupIcon: './resources/icon.ico',
      certificateFile: process.env.WIN_CERT_PATH,
      certificatePassword: process.env.WIN_CERT_PASSWORD,
    }),

    // macOS
    new MakerZIP({}, ['darwin']),
    new MakerDMG({
      icon: './resources/icon.icns',
      format: 'ULFO',
    }),

    // Linux
    new MakerDeb({
      options: {
        maintainer: 'Company Name',
        homepage: 'https://example.com',
        icon: './resources/icon.png',
        categories: ['Utility'],
      },
    }),
    new MakerRpm({
      options: {
        homepage: 'https://example.com',
        icon: './resources/icon.png',
      },
    }),
  ],

  plugins: [
    new VitePlugin({
      build: [
        { entry: 'src/main/index.ts', config: 'vite.main.config.ts' },
        { entry: 'src/preload/index.ts', config: 'vite.preload.config.ts' },
      ],
      renderer: [
        { name: 'main_window', config: 'vite.renderer.config.ts' },
      ],
    }),
  ],

  publishers: [
    {
      name: '@electron-forge/publisher-github',
      config: {
        repository: { owner: 'company', name: 'myapp' },
        prerelease: false,
        draft: true,
      },
    },
  ],
};

export default config;
```

### Commands

```bash
# Development
npm start

# Package (creates executable)
npm run package

# Make (creates distributable)
npm run make

# Publish to GitHub
npm run publish
```

## electron-builder

### Installation

```bash
npm install --save-dev electron-builder
```

### Configuration

```yaml
# electron-builder.yml
appId: com.company.myapp
productName: My Application
copyright: Copyright © 2024 Company Name

directories:
  output: dist
  buildResources: resources

files:
  - "dist/**/*"
  - "package.json"

# ASAR packaging
asar: true
asarUnpack:
  - "**/*.node"  # Native modules
  - "**/ffmpeg*" # Large binaries

# macOS
mac:
  category: public.app-category.productivity
  icon: resources/icon.icns
  hardenedRuntime: true
  gatekeeperAssess: false
  entitlements: build/entitlements.mac.plist
  entitlementsInherit: build/entitlements.mac.plist
  target:
    - target: dmg
      arch: [x64, arm64]
    - target: zip
      arch: [x64, arm64]

# DMG options
dmg:
  contents:
    - x: 130
      y: 220
    - x: 410
      y: 220
      type: link
      path: /Applications
  window:
    width: 540
    height: 380

# Windows
win:
  icon: resources/icon.ico
  target:
    - target: nsis
      arch: [x64, ia32]
    - target: portable
      arch: [x64]
  certificateFile: ${WIN_CSC_LINK}
  certificatePassword: ${WIN_CSC_KEY_PASSWORD}
  publisherName: Company Name

# NSIS installer options
nsis:
  oneClick: false
  perMachine: false
  allowToChangeInstallationDirectory: true
  installerIcon: resources/icon.ico
  uninstallerIcon: resources/icon.ico
  installerHeaderIcon: resources/icon.ico
  createDesktopShortcut: true
  createStartMenuShortcut: true

# Linux
linux:
  icon: resources/icons
  category: Utility
  maintainer: support@company.com
  vendor: Company Name
  target:
    - AppImage
    - deb
    - rpm
    - snap

# Debian
deb:
  depends:
    - gconf2
    - gconf-service
    - libnotify4
    - libappindicator1
    - libxtst6
    - libnss3

# Snap
snap:
  confinement: strict
  grade: stable

# Publishing
publish:
  provider: github
  owner: company
  repo: myapp
  releaseType: release

# Auto-update
afterSign: scripts/notarize.js
```

### Package.json Scripts

```json
{
  "scripts": {
    "build": "vite build",
    "electron:build": "npm run build && electron-builder",
    "electron:build:mac": "npm run build && electron-builder --mac",
    "electron:build:win": "npm run build && electron-builder --win",
    "electron:build:linux": "npm run build && electron-builder --linux",
    "electron:publish": "npm run build && electron-builder --publish always"
  }
}
```

## Code Signing

### macOS

1. **Get Certificates**:
   - Developer ID Application certificate (for distribution)
   - Developer ID Installer certificate (for pkg installers)

2. **Environment Variables**:
```bash
export APPLE_ID="your@email.com"
export APPLE_PASSWORD="app-specific-password"
export APPLE_TEAM_ID="TEAMID"
```

3. **Entitlements File**:
```xml
<!-- build/entitlements.mac.plist -->
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
    <key>com.apple.security.automation.apple-events</key>
    <true/>
</dict>
</plist>
```

4. **Notarization Script** (electron-builder):
```javascript
// scripts/notarize.js
const { notarize } = require('@electron/notarize');

exports.default = async function notarizing(context) {
  const { electronPlatformName, appOutDir } = context;
  if (electronPlatformName !== 'darwin') return;

  const appName = context.packager.appInfo.productFilename;

  return await notarize({
    appBundleId: 'com.company.myapp',
    appPath: `${appOutDir}/${appName}.app`,
    appleId: process.env.APPLE_ID,
    appleIdPassword: process.env.APPLE_PASSWORD,
    teamId: process.env.APPLE_TEAM_ID,
  });
};
```

### Windows

1. **Get EV Certificate** (recommended for SmartScreen)

2. **Environment Variables**:
```bash
export WIN_CSC_LINK="path/to/certificate.pfx"
export WIN_CSC_KEY_PASSWORD="password"
```

3. **Sign with signtool** (if manual):
```bash
signtool sign /tr http://timestamp.digicert.com /td sha256 /fd sha256 /a "MyApp.exe"
```

## ASAR Archives

ASAR (Atom Shell Archive) packages your source files into a single archive, improving startup time on Windows and hiding source code.

### Configuration

```yaml
# electron-builder.yml
asar: true
asarUnpack:
  # Unpack native modules
  - "**/*.node"
  # Unpack large binaries
  - "**/ffmpeg*"
  - "**/sharp*"
```

### ASAR Integrity

```typescript
// forge.config.ts
packagerConfig: {
  asar: {
    integrity: true,
  },
},
```

## Platform-Specific Icons

```
resources/
├── icon.icns          # macOS (512x512 minimum)
├── icon.ico           # Windows (256x256 recommended)
├── icon.png           # Linux (512x512)
└── icons/             # Linux multiple sizes
    ├── 16x16.png
    ├── 32x32.png
    ├── 48x48.png
    ├── 64x64.png
    ├── 128x128.png
    ├── 256x256.png
    └── 512x512.png
```

### Generate Icons

```bash
# Using electron-icon-builder
npm install --save-dev electron-icon-builder

npx electron-icon-builder --input=./resources/icon.png --output=./resources

# Or using ImageMagick
convert icon.png -resize 512x512 icon.icns
convert icon.png -resize 256x256 icon.ico
```

## Multi-Architecture Builds

### Apple Silicon + Intel

```yaml
# electron-builder.yml
mac:
  target:
    - target: dmg
      arch: [x64, arm64]
    - target: zip
      arch: [universal]  # Universal binary
```

### Windows x64 + x86

```yaml
win:
  target:
    - target: nsis
      arch: [x64, ia32]
```

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    strategy:
      matrix:
        os: [macos-latest, windows-latest, ubuntu-latest]

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        env:
          GH_TOKEN: ${{ secrets.GH_TOKEN }}
          # macOS signing
          CSC_LINK: ${{ secrets.MAC_CERT_P12 }}
          CSC_KEY_PASSWORD: ${{ secrets.MAC_CERT_PASSWORD }}
          APPLE_ID: ${{ secrets.APPLE_ID }}
          APPLE_PASSWORD: ${{ secrets.APPLE_PASSWORD }}
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
          # Windows signing
          WIN_CSC_LINK: ${{ secrets.WIN_CERT_PFX }}
          WIN_CSC_KEY_PASSWORD: ${{ secrets.WIN_CERT_PASSWORD }}
        run: npm run electron:publish
```

## Distribution Channels

### GitHub Releases

```yaml
# electron-builder.yml
publish:
  provider: github
  owner: company
  repo: myapp
  releaseType: release  # or 'draft', 'prerelease'
```

### S3/Compatible Storage

```yaml
publish:
  provider: s3
  bucket: my-releases
  region: us-east-1
  acl: public-read
```

### Generic Server

```yaml
publish:
  provider: generic
  url: https://releases.example.com/
```

## Checklist

```markdown
## Pre-Release Packaging Checklist

### Configuration
- [ ] App ID set correctly
- [ ] Version bumped in package.json
- [ ] Product name and copyright updated
- [ ] Category set for app stores

### Icons
- [ ] icns file for macOS (512x512+)
- [ ] ico file for Windows (256x256)
- [ ] png files for Linux (multiple sizes)

### Code Signing
- [ ] macOS Developer ID certificate installed
- [ ] macOS notarization credentials configured
- [ ] Windows EV certificate available
- [ ] Entitlements file correct

### Build
- [ ] ASAR packaging enabled
- [ ] Native modules unpacked
- [ ] Source maps excluded from build
- [ ] Development dependencies excluded

### Testing
- [ ] Fresh install tested on macOS
- [ ] Fresh install tested on Windows
- [ ] Fresh install tested on Linux
- [ ] Auto-update flow tested
- [ ] Code signing verified
```

## Reference

- [Electron Forge](https://www.electronforge.io/)
- [electron-builder](https://www.electron.build/)
- [Distribution Tutorial](https://www.electronjs.org/docs/latest/tutorial/application-distribution)
- [Code Signing](https://www.electronjs.org/docs/latest/tutorial/code-signing)
- [macOS Notarization](https://www.electronjs.org/docs/latest/tutorial/notarization)
