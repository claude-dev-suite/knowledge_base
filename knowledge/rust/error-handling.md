# Rust Error Handling

> Official Documentation: https://doc.rust-lang.org/book/ch09-00-error-handling.html

## Overview

Rust groups errors into two categories:

- **Recoverable** errors, represented by `Result<T, E>` — expected failures a
  caller should handle (file not found, parse failure).
- **Unrecoverable** errors, represented by `panic!` — bugs and broken
  invariants that should abort the current thread.

There are no exceptions. Fallibility is encoded in the type system.

Targets current stable Rust (1.97, edition 2024).

## `Option<T>`

Represents an optional value: `Some(T)` or `None`. Use it for absence, not for
errors that carry a reason.

```rust
fn find(v: &[i32], target: i32) -> Option<usize> {
    v.iter().position(|&x| x == target)
}

match find(&[1, 2, 3], 2) {
    Some(i) => println!("found at {i}"),
    None => println!("not found"),
}
```

Handy combinators: `map`, `and_then`, `unwrap_or`, `unwrap_or_else`,
`unwrap_or_default`, `ok_or` (convert to `Result`), `filter`, `is_some`.

## `Result<T, E>`

Represents success `Ok(T)` or failure `Err(E)`.

```rust
use std::num::ParseIntError;

fn parse(s: &str) -> Result<i32, ParseIntError> {
    s.parse::<i32>()
}

match parse("42") {
    Ok(n) => println!("got {n}"),
    Err(e) => eprintln!("error: {e}"),
}
```

Common combinators: `map`, `map_err`, `and_then`, `ok` (to `Option`),
`unwrap_or`, `unwrap_or_else`, `is_ok`.

## Panic vs Recoverable

| Situation                              | Use            |
| -------------------------------------- | -------------- |
| Expected, caller can react             | `Result`/`Option` |
| Broken invariant / programmer bug      | `panic!`       |
| Prototype / example / test assertion   | `unwrap`, `expect` |

```rust
panic!("invariant violated: index {} out of range", i);

let cfg = std::fs::read_to_string("config.toml")
    .expect("config.toml must exist"); // panics with message on Err
```

`unwrap()` panics on `None`/`Err`; `expect("msg")` does the same but with a
custom message — prefer `expect` so panics are self-documenting. Avoid both in
library code and long-running services.

By default a panic unwinds the stack; you can switch to abort in `Cargo.toml`
(`panic = "abort"` under a profile).

## The `?` Operator

`?` propagates errors: on `Ok(v)`/`Some(v)` it unwraps to `v`; on `Err(e)`/`None`
it returns early from the function. It works in any function whose return type
is a `Result`/`Option` (or another type implementing `Try`).

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username() -> Result<String, io::Error> {
    let mut s = String::new();
    File::open("user.txt")?.read_to_string(&mut s)?; // ? on each fallible step
    Ok(s)
}
```

On an `Err`, `?` converts the error via `From`, so the returned error type just
needs to implement `From<SourceError>`. This is what makes custom error enums
and `Box<dyn Error>` so ergonomic.

`?` also works in `main` if `main` returns `Result`:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let contents = std::fs::read_to_string("data.txt")?;
    println!("{contents}");
    Ok(())
}
```

## The `Error` Trait

`std::error::Error` is the standard trait for error types. Implement it (plus
`Display` and `Debug`) so your errors interoperate with `?`, `Box<dyn Error>`,
and error-reporting tools. `source()` exposes the underlying cause.

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct ConfigError {
    detail: String,
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "config error: {}", self.detail)
    }
}

impl Error for ConfigError {}
```

## Custom Error Types (enum)

A common pattern is an enum with one variant per failure mode, plus `From`
impls so `?` converts automatically.

```rust
use std::fmt;
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(ParseIntError),
    NotFound,
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Io(e) => write!(f, "I/O error: {e}"),
            AppError::Parse(e) => write!(f, "parse error: {e}"),
            AppError::NotFound => write!(f, "not found"),
        }
    }
}

impl std::error::Error for AppError {}

impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self { AppError::Io(e) }
}
impl From<ParseIntError> for AppError {
    fn from(e: ParseIntError) -> Self { AppError::Parse(e) }
}

fn load(path: &str) -> Result<i32, AppError> {
    let text = std::fs::read_to_string(path)?; // io::Error -> AppError
    let n = text.trim().parse::<i32>()?;        // ParseIntError -> AppError
    Ok(n)
}
```

## Error Conversion

- `?` uses `From` to convert error types on the fly.
- `.map_err(|e| ...)` transforms an error explicitly.
- `Box<dyn std::error::Error>` erases the concrete type — convenient for apps
  and prototypes where you don't need to match on variants.
- `TryFrom`/`TryInto` provide fallible value conversions returning `Result`.

```rust
let n: i32 = "5".parse().map_err(|_| AppError::NotFound)?;
```

## Popular Third-Party Crates

These are **external crates** (not part of the standard library); add them to
`Cargo.toml`.

### `thiserror` (library errors)

Derive macro that removes the `Display`/`From`/`Error` boilerplate for custom
error enums. Ideal for libraries that expose typed errors.

```rust
// third-party crate: thiserror
use thiserror::Error;

#[derive(Debug, Error)]
enum DataError {
    #[error("I/O failure")]
    Io(#[from] std::io::Error),
    #[error("invalid record at line {0}")]
    Invalid(usize),
}
```

### `anyhow` (application errors)

Provides `anyhow::Result<T>` (alias for `Result<T, anyhow::Error>`) — a boxed,
dynamic error type with context. Great for applications/binaries where you just
want easy propagation and readable reports.

```rust
// third-party crate: anyhow
use anyhow::{Context, Result};

fn run() -> Result<()> {
    let cfg = std::fs::read_to_string("app.toml")
        .context("failed to read app.toml")?;
    println!("{cfg}");
    Ok(())
}
```

Rule of thumb: `thiserror` for libraries (typed errors), `anyhow` for
applications (opaque errors + context). They compose well together.

## See Also

- `std::result`: https://doc.rust-lang.org/std/result/index.html
- `std::error::Error`: https://doc.rust-lang.org/std/error/trait.Error.html
