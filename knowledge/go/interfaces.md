# Go Interfaces

> Official Documentation: https://go.dev/doc/effective_go#interfaces

## Overview

An interface is a set of method signatures. Any type that implements those
methods satisfies the interface — **implicitly**, with no `implements`
declaration. This structural typing keeps packages decoupled: consumers define
the interfaces they need. Examples target **Go 1.26**.

## Defining and satisfying interfaces

```go
type Shape interface {
	Area() float64
	Perimeter() float64
}

type Rectangle struct{ W, H float64 }

func (r Rectangle) Area() float64      { return r.W * r.H }
func (r Rectangle) Perimeter() float64 { return 2 * (r.W + r.H) }

// Rectangle satisfies Shape automatically — no explicit declaration.
var s Shape = Rectangle{W: 3, H: 4}
fmt.Println(s.Area()) // 12
```

Because satisfaction is implicit, you can define an interface that an existing
type (even from another package) already fulfills.

### Compile-time satisfaction check

To assert that a type implements an interface at compile time, use a blank
identifier assignment:

```go
var _ Shape = (*Rectangle)(nil)
```

## Idiomatic interface design

- **Keep interfaces small.** The most reusable interfaces have one method.
  *"The bigger the interface, the weaker the abstraction."*
- **Accept interfaces, return concrete types.** Functions take interfaces for
  flexibility but return concrete types so callers keep full information.
- **Define interfaces where they are used**, not where the implementation lives.

Single-method interfaces are conventionally named by the method plus `-er`:
`Reader`, `Writer`, `Stringer`, `Closer`.

## The empty interface and `any`

The empty interface has no methods, so every type satisfies it. Since Go 1.18
the built-in alias `any` is the idiomatic spelling of `interface{}`.

```go
func describe(v any) {
	fmt.Printf("value=%v type=%T\n", v, v)
}
```

Reach for `any` only when a value truly can be of any type (generic containers,
decoding arbitrary JSON). Prefer concrete types or generics otherwise.

## Type assertions

A type assertion extracts the concrete value stored in an interface.

```go
var i any = "hello"

s := i.(string)      // panics if i does not hold a string
s, ok := i.(string)  // comma-ok form: ok is false instead of panicking
if !ok {
	// not a string
}
```

## Type switches

A type switch dispatches on the dynamic type of an interface value.

```go
func stringify(v any) string {
	switch x := v.(type) {
	case nil:
		return "nil"
	case int:
		return strconv.Itoa(x)
	case string:
		return x
	case fmt.Stringer:
		return x.String()
	default:
		return fmt.Sprintf("%v", x)
	}
}
```

## Interface composition (embedding)

Interfaces can embed other interfaces to compose larger contracts. The standard
library uses this extensively:

```go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }

type ReadWriter interface {
	Reader
	Writer
}
```

A type satisfies `ReadWriter` if it implements both `Read` and `Write`.

## Common standard library interfaces

| Interface | Method | Purpose |
| --- | --- | --- |
| `error` | `Error() string` | Represent an error value |
| `fmt.Stringer` | `String() string` | Custom text form for printing |
| `io.Reader` | `Read([]byte) (int, error)` | Stream of bytes in |
| `io.Writer` | `Write([]byte) (int, error)` | Stream of bytes out |
| `io.Closer` | `Close() error` | Release a resource |
| `sort.Interface` | `Len/Less/Swap` | Custom sorting |
| `json.Marshaler` | `MarshalJSON() ([]byte, error)` | Custom JSON encoding |
| `http.Handler` | `ServeHTTP(w, r)` | Serve an HTTP request |

### Implementing `error` and `Stringer`

```go
type NotFoundError struct{ Key string }

func (e *NotFoundError) Error() string {
	return fmt.Sprintf("key %q not found", e.Key)
}

type Temperature float64

func (t Temperature) String() string {
	return fmt.Sprintf("%.1f°C", float64(t))
}
```

Any type implementing `fmt.Stringer` controls how it prints with `%v`/`%s`.

## Interfaces and nil

An interface value has two parts: a **type** and a **value**. It equals `nil`
only when *both* are nil. A common bug: returning a typed nil pointer through an
`error` interface makes the interface non-nil.

```go
func bad() error {
	var e *NotFoundError // nil pointer
	return e             // interface is NON-nil (type=*NotFoundError, value=nil)
}

if bad() != nil {
	// true! this branch runs unexpectedly
}
```

Return a literal `nil` for the no-error case, and only assign a concrete error
when one actually occurred.

## Generics vs. interfaces

Interfaces provide *dynamic* dispatch (runtime polymorphism). Generics (Go
1.18+) provide *static* polymorphism with type parameters and can express
constraints using interfaces. Use generics when you need to preserve concrete
types across a call; use interfaces when behavior, not the exact type, is what
matters. See the `advanced` doc for generics.
