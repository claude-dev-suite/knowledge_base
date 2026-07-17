# Go Testing

> Official Documentation: https://pkg.go.dev/testing

## Overview

Go ships with a first-class testing framework in the standard library `testing`
package and the `go test` command — no external test runner required. Tests,
benchmarks, fuzz targets, and examples all live in `_test.go` files. Examples
target **Go 1.26**.

## Test files and functions

- Test files end in `_test.go` and are excluded from normal builds.
- Test functions have the signature `func TestXxx(t *testing.T)`.
- A test fails when it calls `t.Error`/`t.Errorf` (continue) or
  `t.Fatal`/`t.Fatalf` (stop the current test).

```go
// math.go
package mathx

func Abs(x int) int {
	if x < 0 {
		return -x
	}
	return x
}
```

```go
// math_test.go
package mathx

import "testing"

func TestAbs(t *testing.T) {
	got := Abs(-3)
	if got != 3 {
		t.Errorf("Abs(-3) = %d, want 3", got)
	}
}
```

Run tests:

```bash
go test ./...                # all packages
go test -v ./mathx           # verbose
go test -run TestAbs ./mathx # filter by regexp
```

## Table-driven tests

The idiomatic Go pattern is a slice of test cases iterated in a loop, using
subtests via `t.Run` so each case is named and reported independently.

```go
func TestAbs(t *testing.T) {
	tests := []struct {
		name string
		in   int
		want int
	}{
		{"positive", 3, 3},
		{"negative", -3, 3},
		{"zero", 0, 0},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := Abs(tt.in); got != tt.want {
				t.Errorf("Abs(%d) = %d, want %d", tt.in, got, tt.want)
			}
		})
	}
}
```

Subtests can be selected individually and run in parallel:

```bash
go test -run TestAbs/negative ./mathx
```

Call `t.Parallel()` at the top of a subtest to run cases concurrently.

## Helpers, setup, and cleanup

| Method | Purpose |
| --- | --- |
| `t.Helper()` | Mark a function as a test helper (correct line numbers) |
| `t.Cleanup(fn)` | Register cleanup run when the test finishes |
| `t.TempDir()` | Create a temp directory auto-removed after the test |
| `t.Setenv(k, v)` | Set an env var, restored after the test |
| `t.Skip()` | Skip the current test |
| `testing.M` + `TestMain` | Package-wide setup/teardown |

```go
func TestMain(m *testing.M) {
	// global setup
	code := m.Run()
	// global teardown
	os.Exit(code)
}
```

## Benchmarks

Benchmark functions are `func BenchmarkXxx(b *testing.B)`. As of Go 1.24 the
preferred loop is `for b.Loop()`, which runs the body the right number of times
and keeps the benchmarked code from being optimized away.

```go
func BenchmarkAbs(b *testing.B) {
	for b.Loop() {
		Abs(-42)
	}
}
```

The older idiom `for i := 0; i < b.N; i++ { ... }` still works. Run benchmarks
(they are skipped by default) and measure allocations:

```bash
go test -bench=. -benchmem ./mathx
go test -bench=BenchmarkAbs -count=10 ./mathx
```

Use `b.ReportAllocs()`, `b.ResetTimer()`, and `b.StopTimer()`/`b.StartTimer()`
to control measurement.

## Fuzzing

Fuzz targets (`func FuzzXxx(f *testing.F)`) automatically generate inputs to
find edge cases and crashes (built into `go test` since Go 1.18).

```go
func FuzzReverse(f *testing.F) {
	f.Add("hello")           // seed corpus entry
	f.Fuzz(func(t *testing.T, s string) {
		r := Reverse(Reverse(s))
		if r != s {
			t.Errorf("double reverse = %q, want %q", r, s)
		}
	})
}
```

```bash
go test -run=xxx -fuzz=FuzzReverse ./...   # fuzz continuously
go test -fuzz=FuzzReverse -fuzztime=30s ./...
```

Failing inputs are written to `testdata/fuzz/` and become permanent regression
seeds.

## Coverage

```bash
go test -cover ./...                       # summary percentage
go test -coverprofile=cover.out ./...      # write a profile
go tool cover -func=cover.out              # per-function breakdown
go tool cover -html=cover.out              # annotated HTML report
```

## Examples as tests

Example functions document usage *and* run as tests when they have an `// Output:`
comment; `go test` verifies the printed output matches.

```go
func ExampleAbs() {
	fmt.Println(Abs(-5))
	// Output: 5
}
```

## Testify (third-party)

The standard library is deliberately assertion-free — you write comparisons and
call `t.Errorf`. Many teams add [`stretchr/testify`](https://github.com/stretchr/testify)
for expressive assertions, mocks, and suites:

```go
import "github.com/stretchr/testify/assert"

func TestWithTestify(t *testing.T) {
	assert.Equal(t, 3, Abs(-3))
	assert.NoError(t, err)
}
```

Testify is optional; table-driven tests with plain `if` checks remain fully
idiomatic and are what the standard library itself uses.

## Useful flags

| Flag | Effect |
| --- | --- |
| `-race` | Enable the data race detector |
| `-v` | Verbose per-test output |
| `-run <re>` | Run only tests matching the regexp |
| `-count=1` | Disable test result caching |
| `-timeout 30s` | Fail if a test package runs longer than the duration |
| `-shuffle=on` | Randomize test/benchmark execution order |
