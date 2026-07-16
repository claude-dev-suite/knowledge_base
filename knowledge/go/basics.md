# Go Basics

> Official Documentation: https://go.dev/doc/effective_go

## Overview

This document covers Go's core language building blocks: variables, types,
control flow, functions, structs, basic error handling, and the built-in
collection types (slices and maps). All examples target **Go 1.26**.

## Variables and constants

```go
var x int = 10      // explicit type
var y = 20          // type inferred (int)
z := 30             // short declaration (only inside functions)

var (
	name string = "Go"
	age  int    = 16
)

const Pi = 3.14159            // untyped constant
const Greeting = "hello"

// iota for enumerations
const (
	Sunday = iota // 0
	Monday        // 1
	Tuesday       // 2
)
```

Unused local variables and imports are compile errors. The zero value of a
variable is well defined: `0` for numbers, `""` for strings, `false` for bools,
and `nil` for pointers, slices, maps, channels, functions, and interfaces.

## Basic types

| Category | Types |
| --- | --- |
| Booleans | `bool` |
| Integers | `int`, `int8/16/32/64`, `uint`, `uint8/16/32/64`, `uintptr` |
| Aliases | `byte` (= `uint8`), `rune` (= `int32`) |
| Floats | `float32`, `float64` |
| Complex | `complex64`, `complex128` |
| Strings | `string` (immutable UTF-8 byte sequence) |

Conversions are always explicit — there is no implicit numeric coercion:

```go
var i int = 42
var f float64 = float64(i)
var u uint = uint(f)
s := strconv.Itoa(i) // int -> string via strconv
```

## Control flow

```go
// if (with optional init statement)
if v, err := compute(); err != nil {
	return err
} else if v > 100 {
	fmt.Println("large")
}

// for is the only loop keyword
for i := 0; i < 3; i++ {}     // classic
for cond {}                    // while-style
for {}                         // infinite

// range over slices, maps, strings, channels, and integers
for idx, val := range items {}
for range 5 { /* Go 1.22+: iterate 0..4 */ }

// switch (no fallthrough by default; cases can be expressions)
switch {
case age < 13:
	fmt.Println("child")
case age < 20:
	fmt.Println("teen")
default:
	fmt.Println("adult")
}
```

## Functions

Functions are first-class values and can return multiple results.

```go
func add(a, b int) int { return a + b }

// multiple return values (the idiomatic (result, error) pair)
func divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, errors.New("division by zero")
	}
	return a / b, nil
}

// named returns and defer
func readAll(path string) (data []byte, err error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, err
	}
	defer f.Close() // runs when the function returns
	return io.ReadAll(f)
}

// variadic parameters
func sum(nums ...int) int {
	total := 0
	for _, n := range nums {
		total += n
	}
	return total
}

// closures
counter := func() func() int {
	n := 0
	return func() int { n++; return n }
}()
```

`defer` statements run in LIFO order when the surrounding function returns and
are commonly used to release resources.

## Structs and methods

```go
type Point struct {
	X, Y int
}

// value receiver: operates on a copy
func (p Point) Add(q Point) Point {
	return Point{p.X + q.X, p.Y + q.Y}
}

// pointer receiver: can mutate the receiver
func (p *Point) Scale(f int) {
	p.X *= f
	p.Y *= f
}

p := Point{X: 1, Y: 2}
p.Scale(3) // Go auto-takes the address: (&p).Scale(3)
```

Struct embedding provides composition and method promotion:

```go
type Named struct{ Name string }
type User struct {
	Named        // embedded field
	Email string
}

u := User{Named: Named{Name: "Ada"}, Email: "ada@example.com"}
fmt.Println(u.Name) // promoted from Named
```

## Error handling basics

Errors are ordinary values implementing the `error` interface. Check them
explicitly rather than using exceptions.

```go
f, err := os.Open("config.yaml")
if err != nil {
	return fmt.Errorf("opening config: %w", err) // wrap with %w
}

// inspect wrapped errors
if errors.Is(err, os.ErrNotExist) { /* ... */ }

var pathErr *os.PathError
if errors.As(err, &pathErr) {
	fmt.Println(pathErr.Path)
}

// combine multiple errors (Go 1.20+)
err = errors.Join(err1, err2)
```

Use `panic`/`recover` only for truly unrecoverable situations, not for ordinary
error flow.

## Slices

A slice is a dynamically sized view over a backing array.

```go
s := []int{1, 2, 3}
s = append(s, 4, 5)          // grows as needed
sub := s[1:3]                // [2 3] shares backing array
made := make([]int, 0, 10)   // len 0, cap 10

fmt.Println(len(s), cap(s))

// copy elements
dst := make([]int, len(s))
copy(dst, s)

// clear zeroes all elements (Go 1.21+)
clear(s)
```

Because slices share backing arrays, mutating a sub-slice can affect the
original. Use `copy` or `append` to a fresh slice to avoid aliasing.

## Maps

```go
m := map[string]int{"a": 1, "b": 2}
m["c"] = 3

v, ok := m["a"] // ok is false if the key is absent
delete(m, "b")

for k, v := range m { // iteration order is randomized
	fmt.Println(k, v)
}

empty := make(map[string]int)
```

Maps must be initialized with a literal or `make` before writing; writing to a
`nil` map panics. Reading from a `nil` map returns the zero value.

## Pointers

```go
x := 10
p := &x   // *int
*p = 20    // x is now 20
```

Go has pointers but no pointer arithmetic. Use `new(T)` to allocate a zeroed
`T` and return its address.
