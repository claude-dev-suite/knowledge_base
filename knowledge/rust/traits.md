# Rust Traits

> Official Documentation: https://doc.rust-lang.org/book/ch10-02-traits.html

## Overview

A trait defines shared behavior — a set of methods a type can implement. Traits
are Rust's mechanism for interfaces, generic bounds, and polymorphism. They are
similar to interfaces or typeclasses in other languages.

Targets current stable Rust (1.97, edition 2024).

## Defining and Implementing a Trait

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}

pub struct Article {
    pub title: String,
    pub body: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}: {}", self.title, &self.body[..20.min(self.body.len())])
    }
}
```

### The Orphan Rule (Coherence)

You may implement a trait for a type only if the trait **or** the type is local
to your crate. This prevents conflicting implementations across crates.

## Default Methods

A trait can provide default method bodies. Implementors may override them or
rely on the default. Default methods can call other (possibly required) methods
of the same trait.

```rust
pub trait Summary {
    fn summarize_author(&self) -> String;

    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}
```

## Trait Bounds and Generics

Constrain generic type parameters with trait bounds so the body can call trait
methods.

```rust
// impl Trait syntax (concise)
fn notify(item: &impl Summary) {
    println!("Breaking! {}", item.summarize());
}

// Full trait-bound syntax (equivalent)
fn notify_generic<T: Summary>(item: &T) {
    println!("Breaking! {}", item.summarize());
}

// Multiple bounds with +
fn describe<T: Summary + Clone>(item: &T) { /* ... */ }

// where clause for readability
fn process<T, U>(t: &T, u: &U) -> String
where
    T: Summary + Clone,
    U: Summary,
{
    format!("{} / {}", t.summarize(), u.summarize())
}
```

### Conditional Implementations (blanket impls)

Implement methods only for types that satisfy bounds, or implement a trait for
every type that satisfies a bound (a *blanket impl*).

```rust
use std::fmt::Display;

struct Pair<T> { x: T, y: T }

impl<T: Display + PartialOrd> Pair<T> {
    fn largest(&self) -> &T {
        if self.x >= self.y { &self.x } else { &self.y }
    }
}
```

## Trait Objects (`dyn`)

Trait bounds/generics are resolved at compile time (static dispatch). For
runtime polymorphism — e.g. a heterogeneous collection — use a **trait object**
`dyn Trait` behind a pointer (`&dyn`, `Box<dyn>`, `Rc<dyn>`). This uses dynamic
dispatch via a vtable.

```rust
pub struct Screen {
    pub components: Vec<Box<dyn Summary>>,
}

impl Screen {
    pub fn run(&self) {
        for component in &self.components {
            println!("{}", component.summarize());
        }
    }
}
```

A trait must be *object safe* (dyn compatible) to be used as `dyn Trait`:
roughly, its methods must not return `Self` and must not have generic type
parameters.

| Dispatch | Syntax             | When                                   |
| -------- | ------------------ | -------------------------------------- |
| Static   | `impl T` / `<T: >` | one concrete type; zero-cost, inlined  |
| Dynamic  | `dyn T`            | mixed types at runtime; small overhead |

## Associated Types

An associated type is a type placeholder tied to a trait implementation. It
avoids extra generic parameters. The canonical example is `Iterator`:

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

struct Counter { count: u32 }

impl Iterator for Counter {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}
```

## Generic Traits and Associated Constants

Traits may have generic parameters and associated constants:

```rust
trait Container {
    const CAPACITY: usize;
    fn len(&self) -> usize;
}
```

## Deriving Traits

The `derive` attribute auto-generates implementations for common traits when
all fields support them.

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Default)]
struct Point {
    x: i32,
    y: i32,
}
```

Commonly derivable std traits: `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`,
`PartialOrd`, `Ord`, `Hash`, `Default`. (`serde::Serialize` / `Deserialize` are
derivable too, but come from the third-party `serde` crate.)

## Common Standard-Library Traits

### `Display` and `Debug`

`Debug` (`{:?}`) is for developers and derivable. `Display` (`{}`) is for
user-facing output and must be implemented manually.

```rust
use std::fmt;

struct Celsius(f64);

impl fmt::Display for Celsius {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}°C", self.0)
    }
}
```

### `From` and `Into`

Implement `From` and you get `Into` for free via a blanket impl. Powers ergonomic
conversions and the `?` operator's error conversion.

```rust
struct Wrapper(String);

impl From<&str> for Wrapper {
    fn from(s: &str) -> Self {
        Wrapper(s.to_string())
    }
}

let w: Wrapper = "hello".into();     // via Into
let w2 = Wrapper::from("world");     // via From
```

### `Iterator`

Implementing `next` unlocks dozens of default adapter methods (`map`, `filter`,
`collect`, `sum`, ...).

```rust
let doubled: Vec<i32> = (1..=3).map(|x| x * 2).collect(); // [2, 4, 6]
```

### Other Useful Traits

| Trait          | Purpose                                   |
| -------------- | ----------------------------------------- |
| `Default`      | a sensible default value (`T::default()`) |
| `Clone`/`Copy` | duplicating values                        |
| `PartialOrd`/`Ord` | comparison and sorting                |
| `Deref`        | smart-pointer dereferencing               |
| `Drop`         | custom cleanup on scope exit              |
| `TryFrom`/`TryInto` | fallible conversions (`-> Result`)   |

## Supertraits

Require another trait as a prerequisite:

```rust
use std::fmt::Display;

trait Labeled: Display {
    fn label(&self) -> String {
        format!("[{}]", self) // can use Display because of the supertrait
    }
}
```

## See Also

- Advanced Traits: https://doc.rust-lang.org/book/ch20-02-advanced-traits.html
- `std::convert`: https://doc.rust-lang.org/std/convert/index.html
