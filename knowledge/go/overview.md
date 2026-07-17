# Go Overview

> Official Documentation: https://go.dev/doc/

## Overview

Go (often called *Golang*) is an open-source, statically typed, compiled
programming language created at Google by Robert Griesemer, Rob Pike, and Ken
Thompson. It emphasizes simplicity, fast compilation, built-in concurrency, and
a large standard library. Programs compile to a single self-contained native
binary with no external runtime dependency.

This documentation targets **Go 1.26** (the current stable release).

### Key characteristics

| Feature | Description |
| --- | --- |
| Compiled | Produces native machine code; no VM required |
| Statically typed | Types checked at compile time, with type inference |
| Garbage collected | Automatic memory management (low-latency collector) |
| Concurrency | Goroutines and channels built into the language |
| Fast builds | Designed for very fast compilation |
| Cross-compilation | Build for other OS/arch with `GOOS`/`GOARCH` |
| Tooling | Formatting, testing, and dependency management built in |

## Installation

Download installers from https://go.dev/dl/ or use a package manager.

```bash
# macOS (Homebrew)
brew install go

# Debian/Ubuntu
sudo apt install golang-go

# Windows (winget)
winget install GoLang.Go
```

Verify the installation:

```bash
go version
# go version go1.26.0 <os>/<arch>
```

Check your environment configuration:

```bash
go env
go env GOPATH GOMODCACHE GOBIN
```

Since modules became the default, `GOPATH` is only used to store the module
cache and installed binaries. Your source code can live anywhere.

## Hello, World

Create a module and a program:

```bash
mkdir hello && cd hello
go mod init example.com/hello
```

```go
// main.go
package main

import "fmt"

func main() {
	fmt.Println("Hello, World!")
}
```

Run it, or build a binary:

```bash
go run .
# Hello, World!

go build -o hello .
./hello
```

## Tooling

The `go` command is the single entry point for building, testing, and managing
Go code. No external build system is required.

| Command | Purpose |
| --- | --- |
| `go run .` | Compile and run the current package |
| `go build` | Compile packages into a binary |
| `go test ./...` | Run tests in all packages |
| `go fmt ./...` | Format source with `gofmt` |
| `go vet ./...` | Report suspicious constructs |
| `go mod tidy` | Add missing / remove unused dependencies |
| `go get pkg@version` | Add or update a dependency |
| `go install pkg@latest` | Build and install a binary tool |
| `go doc fmt.Println` | Show documentation for a symbol |
| `go work` | Manage multi-module workspaces |

### Formatting

Go has a single canonical style enforced by `gofmt`; there are no style
debates. Most editors run it on save. Imports can be organized with
`goimports` (from `golang.org/x/tools`).

### Static analysis

`go vet` catches common mistakes (bad `Printf` verbs, lock copies, unreachable
code). For deeper linting the community standard is
[`golangci-lint`](https://golangci-lint.run/), which aggregates many linters.

## When to use Go

Go is a strong fit for:

- Network services, APIs, and microservices
- CLI tools and developer tooling
- Cloud infrastructure (Docker, Kubernetes, Terraform are written in Go)
- High-concurrency backends and data pipelines

It is less commonly chosen for:

- GUI desktop / mobile apps (limited native ecosystem)
- Heavy numeric / scientific computing (Python, Julia dominate)
- Hard real-time systems (garbage collection introduces pauses)

## Learning resources

- A Tour of Go — https://go.dev/tour/
- Effective Go — https://go.dev/doc/effective_go
- Standard library reference — https://pkg.go.dev/std
- Go by Example — https://gobyexample.com/

## Next steps

Continue with the focused topic docs: `basics`, `concurrency`, `interfaces`,
`modules`, `testing`, and `advanced`.
