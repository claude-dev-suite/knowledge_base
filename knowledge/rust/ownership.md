# Rust Ownership

> Official Documentation: https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html

## Overview

Ownership is Rust's core memory-management model. It lets Rust guarantee memory
safety at compile time without a garbage collector. Every value has a single
*owner*; when the owner goes out of scope, the value is dropped (freed).

Targets current stable Rust (1.97, edition 2024).

## The Three Rules

1. Each value in Rust has an *owner*.
2. There can only be one owner at a time.
3. When the owner goes out of scope, the value is dropped.

```rust
fn main() {
    let s = String::from("hello"); // s owns the heap-allocated string
    // ... use s ...
} // scope ends: `s` is dropped, memory freed automatically
```

## Move vs Copy

Assigning or passing a value either **moves** it or **copies** it.

- Types with a fixed size stored entirely on the stack implement the `Copy`
  trait (integers, `bool`, `char`, `f64`, tuples/arrays of `Copy` types).
  Assignment copies the bits; both bindings stay valid.
- Types that own heap data (e.g. `String`, `Vec<T>`, `Box<T>`) are **moved**.
  The original binding is invalidated to preserve single ownership.

```rust
// Move: String owns heap data
let s1 = String::from("hi");
let s2 = s1;         // s1 is moved into s2
// println!("{s1}"); // ERROR: value borrowed after move

// Copy: i32 is Copy
let x = 5;
let y = x;           // x is copied
println!("{x} {y}"); // OK: 5 5
```

To get an independent copy of an owned type, call `.clone()` explicitly:

```rust
let s1 = String::from("hi");
let s2 = s1.clone(); // deep copy; both valid
println!("{s1} {s2}");
```

| Behavior | Trait      | Example types                     |
| -------- | ---------- | --------------------------------- |
| Copy     | `Copy`     | `i32`, `bool`, `char`, `f64`      |
| Move     | (no `Copy`) | `String`, `Vec<T>`, `Box<T>`     |
| Clone    | `Clone`    | most types; deep copy on demand   |

## Ownership and Functions

Passing a value to a function moves (or copies) it, just like assignment.
Returning a value moves ownership back out.

```rust
fn takes_ownership(s: String) {
    println!("{s}");
} // s dropped here

fn gives_ownership() -> String {
    String::from("owned")
}

fn main() {
    let s = gives_ownership();
    takes_ownership(s); // s moved in; no longer usable here
}
```

## Borrowing & References

Instead of transferring ownership, you can *borrow* a value with a reference
(`&`). The reference does not own the data, so nothing is dropped when it goes
out of scope.

- `&T` — a **shared (immutable) reference**; you may have many at once.
- `&mut T` — a **mutable reference**; exclusive, only one at a time.

```rust
fn calculate_len(s: &String) -> usize {
    s.len() // borrow, don't take ownership
}

fn append(s: &mut String) {
    s.push_str(" world");
}

fn main() {
    let mut s = String::from("hello");
    let len = calculate_len(&s); // shared borrow
    append(&mut s);              // mutable borrow
    println!("{s} has len {len}");
}
```

### The Borrowing Rules

At any given time, for a particular value, you can have **either**:

- any number of shared references (`&T`), **or**
- exactly one mutable reference (`&mut T`).

References must always be valid (no dangling). These rules are enforced at
compile time and prevent data races.

```rust
let mut s = String::from("hi");
let r1 = &s;
let r2 = &s;      // OK: multiple shared borrows
println!("{r1} {r2}");
let r3 = &mut s;  // OK: r1/r2 no longer used after this point (NLL)
r3.push('!');
```

Rust uses *non-lexical lifetimes* (NLL): a borrow ends at its last use, not at
the end of the enclosing block.

## Slices

A slice is a reference to a contiguous subsequence of a collection. It borrows,
so it does not own the data. String slices have type `&str`; array/vector slices
have type `&[T]`.

```rust
let s = String::from("hello world");
let hello: &str = &s[0..5];   // or &s[..5]
let world: &str = &s[6..11];  // or &s[6..]

let nums = [1, 2, 3, 4, 5];
let middle: &[i32] = &nums[1..4]; // [2, 3, 4]
```

String literals are `&'static str` slices baked into the binary:

```rust
let greeting: &str = "hello"; // &'static str
```

Prefer `&str` and `&[T]` in function parameters so callers can pass either an
owned value or a slice:

```rust
fn first_word(s: &str) -> &str {
    s.split_whitespace().next().unwrap_or("")
}
```

## Lifetimes Basics

Lifetimes are the compiler's way of tracking how long references stay valid.
Most are inferred; you annotate them only when the compiler cannot infer the
relationship between input and output references. Annotations are named with a
leading apostrophe (e.g. `'a`) and never change how long a value actually lives
— they only describe constraints.

```rust
// The returned reference lives as long as the shorter of x and y.
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

### Lifetime Elision

For many functions the compiler applies elision rules so you don't annotate:

- Each reference parameter gets its own lifetime.
- If there is exactly one input lifetime, it is assigned to all outputs.
- For methods, the lifetime of `&self` / `&mut self` is assigned to outputs.

```rust
// Elided; no explicit lifetimes needed
fn first(s: &str) -> &str {
    &s[..1]
}
```

### `'static`

The `'static` lifetime means a reference can live for the entire program.
String literals are `'static`. Do not add `'static` bounds just to silence the
borrow checker — usually the real fix is restructuring ownership.

## Common Pitfalls

- **Use after move**: reading a variable after it was moved. Fix by borrowing
  (`&`), cloning, or restructuring so ownership is not given away.
- **Simultaneous mutable + shared borrow**: split accesses or shorten borrow
  scope.
- **Returning a reference to a local**: the local is dropped at function end;
  return an owned value instead.

## See Also

- References and Borrowing: https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html
- Validating References with Lifetimes: https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html
