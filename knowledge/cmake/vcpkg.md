# CMake - vcpkg (Manifest Mode)

> **Source**: https://learn.microsoft.com/vcpkg/users/manifests
> **Skill**: dev-suite skill `cmake` — see SKILL.md for the always-loaded quick reference.

## What this covers

vcpkg's manifest mode (per-project `vcpkg.json` instead of the global classic mode), baselines and version pinning, custom triplets, registry overlays for proprietary or patched ports, and CI integration with binary caching.

## Deep dive

### Classic vs manifest mode

- **Classic** (legacy): `vcpkg install fmt spdlog` installs into a global location; every project shares the same versions. Hard to reproduce.
- **Manifest** (recommended): `vcpkg.json` in the project root declares dependencies; vcpkg installs them into a per-project `vcpkg_installed/` directory tied to the build tree.

Manifest mode is activated automatically when CMake detects `vcpkg.json` next to `CMakeLists.txt` *and* the toolchain file is set.

### Minimal vcpkg.json

```json
{
  "$schema": "https://raw.githubusercontent.com/microsoft/vcpkg-tool/main/docs/vcpkg.schema.json",
  "name": "myapp",
  "version": "1.0.0",
  "dependencies": [
    "fmt",
    "spdlog",
    {
      "name": "boost-asio",
      "version>=": "1.84.0"
    },
    {
      "name": "qtbase",
      "default-features": false,
      "features": ["network", "concurrent"]
    }
  ],
  "builtin-baseline": "8a3a9d8b6e3c0f9a3a9b6e3c0f9a3a9b6e3c0f9a"
}
```

### Baselines pin the universe

`builtin-baseline` is a commit SHA in the vcpkg repo. *Every* dependency resolves to the version that was current in vcpkg at that SHA (unless overridden). This makes builds reproducible across machines and time.

Update with: `vcpkg x-update-baseline --add-initial-baseline`.

### Version overrides

```json
{
  "dependencies": ["fmt", "spdlog"],
  "overrides": [
    { "name": "fmt", "version": "10.2.1" }
  ],
  "builtin-baseline": "8a3a9d8b..."
}
```

`overrides` pins exact versions for select packages without abandoning the baseline for everything else.

### Toolchain integration

```bash
cmake -S . -B build \
  -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake \
  -DVCPKG_TARGET_TRIPLET=x64-linux
cmake --build build
```

The toolchain hooks into `find_package` so your project code is unchanged:

```cmake
find_package(fmt CONFIG REQUIRED)
find_package(spdlog CONFIG REQUIRED)
target_link_libraries(myapp PRIVATE fmt::fmt spdlog::spdlog)
```

In a `CMakePresets.json`:

```json
{
  "name": "vcpkg",
  "hidden": true,
  "cacheVariables": {
    "CMAKE_TOOLCHAIN_FILE": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"
  }
}
```

### Triplets

A triplet describes the target ABI: `x64-windows`, `x64-windows-static`, `x64-linux`, `arm64-osx`. Static-linkage triplets are common for redistributable binaries:

```bash
cmake -DVCPKG_TARGET_TRIPLET=x64-windows-static ...
```

Custom triplets live in `triplets/` next to `vcpkg.json`. Useful to override `VCPKG_BUILD_TYPE release` (skip debug builds in CI) or to set `VCPKG_C_FLAGS`.

### Features

A feature is an optional component of a port. Declare them in your manifest:

```json
{
  "dependencies": [
    {
      "name": "qtbase",
      "default-features": false,
      "features": ["network", "concurrent", "gui"]
    }
  ]
}
```

Setting `default-features: false` then opting in keeps the install minimal.

### Custom registries (private packages)

```json
{
  "registries": [
    {
      "kind": "git",
      "repository": "https://github.example.com/team/vcpkg-registry",
      "baseline": "abc123...",
      "packages": ["company-core", "company-net"]
    }
  ],
  "dependencies": ["company-core", "fmt"]
}
```

Hosts internal ports without forking the entire vcpkg repo.

### Binary caching for CI

```bash
# Read+write to a NuGet feed
export VCPKG_BINARY_SOURCES="clear;nuget,https://nuget.example.com/v3/index.json,readwrite"

# Or files-based cache (S3, Azure Blob, local)
export VCPKG_BINARY_SOURCES="clear;files,/cache/vcpkg-bincache,readwrite"

# GitHub Actions native cache
export VCPKG_BINARY_SOURCES="clear;x-gha,readwrite"
```

Without binary caching, CI builds every dependency from source each run — measure in tens of minutes for Boost+Qt. With it, cache hits restore in seconds.

### vcpkg + CMakePresets

```json
{
  "configurePresets": [
    {
      "name": "ci",
      "binaryDir": "${sourceDir}/build/ci",
      "generator": "Ninja",
      "cacheVariables": {
        "CMAKE_TOOLCHAIN_FILE":   "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake",
        "VCPKG_TARGET_TRIPLET":   "x64-linux",
        "VCPKG_HOST_TRIPLET":     "x64-linux",
        "VCPKG_INSTALL_OPTIONS":  "--clean-after-build",
        "CMAKE_BUILD_TYPE":       "Release"
      }
    }
  ]
}
```

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Missing `builtin-baseline` | Versions float silently | Always pin a baseline |
| Wrong triplet for static linking | DLLs leak into the deployment | Use `x64-windows-static` triplet (and matching `set(VCPKG_TARGET_TRIPLET ... CACHE)` early) |
| `find_package(fmt)` without `CONFIG` | Picks up an unrelated `FindFmt.cmake` | Always specify `CONFIG REQUIRED` |
| Forgot to set `VCPKG_ROOT` env var in CI | Toolchain fails to find vcpkg | Export before configure |
| Bumping baseline blows up build | New transitive deps | Update baseline only when ready to test the diff |
| `default-features: true` (default) installs huge optional deps | Long install, large footprint | Disable defaults, opt in to specific features |

## See also

- https://learn.microsoft.com/vcpkg/concepts/manifest-mode — concept docs
- https://learn.microsoft.com/vcpkg/users/binarycaching — binary caching options
- https://learn.microsoft.com/vcpkg/users/triplets — triplet authoring
