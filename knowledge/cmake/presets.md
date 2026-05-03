# CMake - Presets (CMakePresets.json)

> **Source**: https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html
> **Skill**: dev-suite skill `cmake` — see SKILL.md for the always-loaded quick reference.

## What this covers

The full preset hierarchy (configure / build / test / package / workflow), inheritance and condition predicates, the `CMakeUserPresets.json` extension mechanism for personal overrides, environment substitution, and IDE integration (VS Code, CLion, Visual Studio).

## Deep dive

### Why presets exist

Before presets, every developer ran a different `cmake -DCMAKE_BUILD_TYPE=... -DCMAKE_TOOLCHAIN_FILE=...` invocation, captured loosely in a README. Presets put that configuration into a checked-in JSON file that IDEs and CLIs both understand.

### File pair: project + user

| File | Tracked in git? | Purpose |
|------|-----------------|---------|
| `CMakePresets.json` | yes | Shared, project-wide presets |
| `CMakeUserPresets.json` | no (`.gitignore`) | Per-developer additions/overrides |

User presets can `inherits` from project presets and add personal paths.

### Full preset hierarchy

```json
{
  "version": 6,
  "cmakeMinimumRequired": { "major": 3, "minor": 25, "patch": 0 },
  "configurePresets": [
    {
      "name": "base",
      "hidden": true,
      "binaryDir": "${sourceDir}/build/${presetName}",
      "generator": "Ninja",
      "cacheVariables": {
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON",
        "CMAKE_COLOR_DIAGNOSTICS": "ON"
      },
      "warnings": { "dev": true, "uninitialized": true }
    },
    {
      "name": "linux-clang-debug",
      "inherits": "base",
      "cacheVariables": {
        "CMAKE_C_COMPILER":   "clang",
        "CMAKE_CXX_COMPILER": "clang++",
        "CMAKE_BUILD_TYPE":   "Debug"
      },
      "condition": { "type": "equals", "lhs": "${hostSystemName}", "rhs": "Linux" }
    },
    {
      "name": "msvc-release",
      "inherits": "base",
      "generator": "Visual Studio 17 2022",
      "architecture": { "value": "x64", "strategy": "set" },
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Release" },
      "condition": { "type": "equals", "lhs": "${hostSystemName}", "rhs": "Windows" }
    },
    {
      "name": "vcpkg",
      "inherits": "base",
      "hidden": true,
      "cacheVariables": {
        "CMAKE_TOOLCHAIN_FILE": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"
      }
    },
    {
      "name": "ci-vcpkg-release",
      "inherits": ["base", "vcpkg"],
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Release" }
    }
  ],
  "buildPresets": [
    { "name": "linux-clang-debug", "configurePreset": "linux-clang-debug", "jobs": 0 },
    { "name": "ci-vcpkg-release", "configurePreset": "ci-vcpkg-release",
      "targets": ["all"], "configuration": "Release" }
  ],
  "testPresets": [
    { "name": "linux-clang-debug", "configurePreset": "linux-clang-debug",
      "output": { "outputOnFailure": true, "shortProgress": true },
      "execution": { "noTestsAction": "error", "stopOnFailure": false, "jobs": 0 } }
  ],
  "packagePresets": [
    { "name": "release-tarball", "configurePreset": "ci-vcpkg-release",
      "generators": ["TGZ", "ZIP"] }
  ],
  "workflowPresets": [
    {
      "name": "ci",
      "steps": [
        { "type": "configure", "name": "ci-vcpkg-release" },
        { "type": "build",     "name": "ci-vcpkg-release" },
        { "type": "test",      "name": "linux-clang-debug" },
        { "type": "package",   "name": "release-tarball" }
      ]
    }
  ]
}
```

Run a workflow: `cmake --workflow --preset ci`. This single command does configure + build + test + package.

### Inheritance rules

- `inherits` is a list; later entries override earlier ones.
- `cacheVariables` merge per-key; `environment` merges per-key.
- `hidden: true` means "don't show in `--list-presets`, only inherit from".
- A preset can inherit from another that inherits — chains are allowed.

### Conditions

```json
"condition": {
  "type": "allOf",
  "conditions": [
    { "type": "equals", "lhs": "${hostSystemName}", "rhs": "Linux" },
    { "type": "notEquals", "lhs": "$env{CI}", "rhs": "" }
  ]
}
```

Supported types: `const`, `equals`, `notEquals`, `inList`, `matches`, `notMatches`, `anyOf`, `allOf`, `not`. Useful for "this preset only applies in CI" or "Windows-only".

### Macros and environment substitution

| Macro | Value |
|-------|-------|
| `${sourceDir}` | Absolute path to `CMakePresets.json`'s directory |
| `${presetName}` | The preset's `name` field |
| `${generator}` | Resolved generator name |
| `$env{VAR}` | Environment variable (empty if unset) |
| `$penv{VAR}` | Like `$env` but resolved at preset parse time, not at CMake execution time |

### IDE integration

| IDE | Notes |
|-----|-------|
| VS Code (CMake Tools) | Picks up `CMakePresets.json` automatically; status bar shows active preset |
| Visual Studio 2022 | Native support; `CMakeSettings.json` is deprecated in favor of presets |
| CLion | 2023.2+ supports presets; older versions need manual import |
| Ninja CLI | `cmake --preset name` then `cmake --build --preset name` |

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Top-level `version` lower than features used | Older `version` rejects newer keys | Bump to 6+ when using workflow/package presets |
| Putting per-developer paths in `CMakePresets.json` | Breaks for everyone else | Use `CMakeUserPresets.json` (gitignored) |
| `cacheVariables` value not a string | Schema requires strings even for booleans | Quote: `"ON"` not `true` |
| Chaining `inherits` for unrelated config sets | Inheritance order surprises | Prefer composition with `inherits: [a, b]` |
| Missing `binaryDir` → builds in source | Default location may not be ignored by `.gitignore` | Always set to `${sourceDir}/build/${presetName}` |
| Workflow that references presets out of order | Workflow steps run linearly; later steps need earlier outputs | Order: configure → build → test → package |

## See also

- https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html — formal schema
- https://github.com/CLIUtils/CLI11/blob/main/CMakePresets.json — real-world example
- https://devblogs.microsoft.com/cppblog/cmake-presets-integration-in-visual-studio-and-visual-studio-code/ — VS integration
