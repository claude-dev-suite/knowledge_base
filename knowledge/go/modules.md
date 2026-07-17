# Go Modules

> Official Documentation: https://go.dev/ref/mod

## Overview

A **module** is a collection of Go packages versioned together as a unit and
defined by a `go.mod` file at its root. Modules are Go's dependency management
system and have been the default since Go 1.16. Examples target **Go 1.26**.

## Creating a module

```bash
go mod init example.com/myapp
```

This writes a `go.mod` file. The module path is typically the repository URL
(e.g. `github.com/user/repo`) so the module can be fetched by others.

## The go.mod file

```go
module example.com/myapp

go 1.26

require (
	github.com/google/uuid v1.6.0
	golang.org/x/text v0.16.0
)

require (
	github.com/some/indirect v1.2.0 // indirect
)
```

| Directive | Purpose |
| --- | --- |
| `module` | The module's import path |
| `go` | Minimum Go language version for the module |
| `require` | A dependency and its minimum version |
| `exclude` | Prevent use of a specific module version |
| `replace` | Redirect a module path/version (e.g. to a local fork) |
| `retract` | Mark this module's own versions as withdrawn |
| `tool` | Track an executable dependency (Go 1.24+) |

The `// indirect` comment marks dependencies not imported directly by your code.

## go.sum

`go.sum` records cryptographic checksums of module contents for every dependency
in the build graph, providing supply-chain integrity. Commit both `go.mod` and
`go.sum`. Checksums are also verified against the public checksum database
(`sum.golang.org`) by default.

## Managing dependencies

```bash
go get github.com/pkg/errors@v0.9.1   # add / pin a specific version
go get github.com/pkg/errors@latest   # upgrade to latest release
go get -u ./...                        # update dependencies to newer minors
go get example.com/pkg@none            # remove a dependency
go mod tidy                            # sync go.mod/go.sum with actual imports
go mod download                        # populate the module cache
go mod verify                          # check cached modules match go.sum
go list -m all                         # list the full module graph
```

Run `go mod tidy` before committing; it adds anything imported and removes
anything unused.

## Versioning and semantic import versioning

Go modules use **semantic versioning** (`vMAJOR.MINOR.PATCH`). The tooling
follows Minimal Version Selection: it picks the highest version required by any
module in the graph, not the newest available.

For **major version 2 and above**, the major version becomes part of the import
path:

```go
module github.com/user/lib/v2

go 1.26
```

Consumers then import `github.com/user/lib/v2/...`. Versions `v0` and `v1` use
the bare path with no suffix. Pre-release versions look like `v1.5.0-beta.1`,
and untagged commits get a pseudo-version like
`v0.0.0-20260115120000-abcdef123456`.

## Publishing a module

1. Ensure the module path matches the repository URL.
2. Ensure the code builds and `go mod tidy` is clean.
3. Tag a release and push the tag:

```bash
git tag v1.0.0
git push origin v1.0.0
```

For `v2+`, tag with the matching major (`v2.0.0`) and make sure the module path
carries the `/v2` suffix. The first time someone runs `go get`, the module proxy
(`proxy.golang.org`) fetches and caches it, and it becomes visible on
https://pkg.go.dev.

To withdraw a broken published version, add a `retract` directive and publish a
new patch:

```go
retract v1.0.1 // contains a critical bug
```

## Workspaces (go work)

Workspaces (Go 1.18+) let you develop several interdependent modules together
without editing each `go.mod` with `replace` directives.

```bash
go work init ./api ./worker      # create go.work referencing local modules
go work use ./newmodule          # add another module to the workspace
go work sync                      # sync workspace build list to modules
```

The generated `go.work`:

```go
go 1.26

use (
	./api
	./worker
)
```

When a `go.work` file is present, its `use` modules take precedence over
published versions. `go.work` is usually **not** committed (it is a local
developer convenience); commit only when the whole team shares the layout.

## Vendoring

Vendoring copies dependencies into a local `vendor/` directory so builds do not
need network access or the module cache.

```bash
go mod vendor            # write dependencies into ./vendor
go build -mod=vendor .   # build using the vendor directory
```

If a `vendor/` directory exists and is consistent, the `go` command uses it
automatically (`-mod=vendor` becomes the default). Run `go mod vendor` again
after changing dependencies to keep it in sync.

## Useful environment variables

| Variable | Purpose |
| --- | --- |
| `GOPROXY` | Module proxy URL(s); default `https://proxy.golang.org,direct` |
| `GOSUMDB` | Checksum database; `off` disables verification |
| `GOPRIVATE` | Glob patterns for private modules (skip proxy + sumdb) |
| `GOINSECURE` | Patterns fetched over HTTP without checksum verification |
| `GOFLAGS` | Default flags applied to every `go` command |

For private repositories:

```bash
go env -w GOPRIVATE=github.com/mycompany/*
```

This tells Go to fetch matching modules directly (via git) and skip the public
proxy and checksum database.
