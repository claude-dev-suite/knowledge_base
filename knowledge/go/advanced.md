# Go Advanced

> Official Documentation: https://go.dev/ref/spec

## Overview

This document covers advanced Go features: generics, reflection, `unsafe`, cgo,
profiling with pprof, build constraints, and the memory model. These are
powerful but should be used sparingly and only when simpler code will not do.
Examples target **Go 1.26**.

## Generics

Type parameters (Go 1.18+) let functions and types operate over many types while
keeping full static type safety.

```go
// A type parameter T constrained by a union of ordered types.
type Ordered interface {
	~int | ~int64 | ~float64 | ~string
}

func Max[T Ordered](a, b T) T {
	if a > b {
		return a
	}
	return b
}

// Generic container.
type Stack[T any] struct {
	items []T
}

func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }

func (s *Stack[T]) Pop() (T, bool) {
	var zero T
	if len(s.items) == 0 {
		return zero, false
	}
	v := s.items[len(s.items)-1]
	s.items = s.items[:len(s.items)-1]
	return v, true
}

m := Max(3, 7)          // T inferred as int
s := Max("a", "b")      // T inferred as string
```

Constraints are interfaces that may include method sets and **type sets**
(unions with the `~` approximation for named types). The `constraints` package
(`golang.org/x/exp/constraints`) provides ready-made constraints like `Ordered`.
The standard library ships generic helpers in the `slices`, `maps`, and `cmp`
packages:

```go
import ("slices"; "cmp")

nums := []int{3, 1, 2}
slices.Sort(nums)                        // [1 2 3]
i, found := slices.BinarySearch(nums, 2)
best := slices.MaxFunc(people, func(a, b Person) int {
	return cmp.Compare(a.Age, b.Age)
})
```

Generics do not support method-level type parameters and are not a full
substitute for interfaces — prefer interfaces when dynamic dispatch is what you
need.

## Reflection

The `reflect` package inspects and manipulates values at runtime. It powers
encoders like `encoding/json`, but it is slow and bypasses compile-time checks —
use it only when necessary.

```go
import "reflect"

func dump(v any) {
	t := reflect.TypeOf(v)
	val := reflect.ValueOf(v)
	if t.Kind() == reflect.Struct {
		for i := 0; i < t.NumField(); i++ {
			f := t.Field(i)
			fmt.Printf("%s (%s) = %v [tag=%q]\n",
				f.Name, f.Type, val.Field(i), f.Tag.Get("json"))
		}
	}
}
```

Struct tags (the backtick strings after fields) are read via reflection and
drive libraries such as JSON and database mappers:

```go
type User struct {
	Name  string `json:"name"`
	Email string `json:"email,omitempty"`
}
```

The reflection laws: you can go from value to `reflect.Value` and back, and you
can only modify a value through reflection if it is *addressable* (obtained via a
pointer with `Elem()`).

## unsafe (use with caution)

The `unsafe` package steps outside the type system for low-level memory access.
It is not covered by the Go 1 compatibility promise and can break across
releases or platforms.

```go
import "unsafe"

var f float64 = 3.14
bits := *(*uint64)(unsafe.Pointer(&f)) // reinterpret the bit pattern
size := unsafe.Sizeof(f)               // 8
```

Prefer the typed helpers `unsafe.Slice`, `unsafe.String`, and
`unsafe.SliceData`/`unsafe.StringData` over manual pointer arithmetic. Reach for
`unsafe` only for performance-critical interop or serialization, and isolate it
behind a safe API.

## cgo (calling C)

cgo lets Go call C code. Import the pseudo-package `"C"`; the comment
immediately above the import is compiled as C.

```go
/*
#include <stdlib.h>
#include <math.h>
*/
import "C"
import "fmt"

func main() {
	x := C.sqrt(C.double(16))
	fmt.Println(float64(x)) // 4
}
```

Caveats: cgo requires a C toolchain, slows builds, complicates
cross-compilation, and adds call overhead at the Go/C boundary (reduced by ~30%
in Go 1.26 but still non-trivial). Memory allocated with `C.malloc` must be
freed with `C.free`. Avoid cgo unless a C library is genuinely required; set
`CGO_ENABLED=0` for fully static binaries.

## Profiling with pprof

Go has built-in CPU, memory, block, mutex, and goroutine profiling via
`runtime/pprof` and `net/http/pprof`.

Profile from tests:

```bash
go test -cpuprofile cpu.out -memprofile mem.out -bench .
go tool pprof cpu.out          # interactive: top, list, web
```

Expose live profiles from a running server by importing for side effects:

```go
import _ "net/http/pprof" // registers handlers on the default mux

go func() {
	log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

```bash
go tool pprof http://localhost:6060/debug/pprof/heap
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

For lightweight execution timelines use the runtime tracer:

```bash
go test -trace trace.out -bench .
go tool trace trace.out
```

## Build constraints (build tags)

Build constraints control which files compile for which platforms or
configurations. Use the `//go:build` line at the top of a file (above the
`package` clause, followed by a blank line).

```go
//go:build linux && amd64

package sysinfo
```

File-name suffixes also imply constraints: `cache_windows.go` builds only on
Windows; `parser_test.go` builds only under `go test`. Combine tags with
boolean operators `&&`, `||`, and `!`. Enable custom tags at build time:

```bash
go build -tags "integration jsoniter" ./...
```

Other `//go:` directives worth knowing:

| Directive | Purpose |
| --- | --- |
| `//go:embed file.txt` | Embed files into the binary (`embed` package) |
| `//go:generate cmd` | Command run by `go generate` |
| `//go:noinline` | Prevent inlining of a function (rarely needed) |

## Memory model highlights

Go's memory model (https://go.dev/ref/mem) defines when a read in one goroutine
is guaranteed to observe a write from another.

- If a program has **data races** (concurrent unsynchronized access with at
  least one write), behavior is undefined. Always run `go test -race`.
- Synchronization establishes *happens-before* ordering. Channel operations, the
  `sync` primitives (`Mutex`, `WaitGroup`, `Once`), and `sync/atomic` all create
  these guarantees.
- A send on a channel happens before the corresponding receive completes; a
  `close` happens before a receive that observes the closed channel.
- `sync.Once.Do(f)` guarantees `f` completes before any `Do` call returns.
- Do not rely on the order of independent goroutines without explicit
  synchronization — the compiler and CPU may reorder unsynchronized operations.

The practical rule: share data through channels or protect it with a mutex or
atomics; never assume ordering that you have not explicitly synchronized.
