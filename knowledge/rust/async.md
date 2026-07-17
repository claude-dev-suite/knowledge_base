# Rust Async

> Official Documentation: https://doc.rust-lang.org/book/ch17-00-async-await.html

## Overview

Rust supports asynchronous programming with `async`/`await`. Async functions
return **futures** — lazy computations that make progress only when polled by a
runtime (executor). The standard library defines the core types (`Future`,
`async`/`await`, `Poll`) but **does not ship a runtime**; you pick one, most
commonly the third-party `tokio` crate.

Targets current stable Rust (1.97, edition 2024).

## `async` / `await`

An `async fn` (or `async {}` block) evaluates to a value implementing
`std::future::Future`. Futures are lazy — nothing runs until `.await`ed or
spawned on an executor.

```rust
async fn fetch_len(url: &str) -> usize {
    // pretend this awaits I/O
    url.len()
}

async fn run() {
    let n = fetch_len("https://example.com").await; // suspends until ready
    println!("length = {n}");
}
```

`.await` yields control back to the runtime while the future is not ready,
letting other tasks progress on the same thread. `.await` is only valid inside
an `async` context.

## Futures

`Future` is the foundational trait:

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

You rarely implement `poll` by hand — the compiler generates a state machine
from each `async fn`/block. `Poll` is either `Poll::Ready(value)` or
`Poll::Pending`.

## Runtimes: Tokio

**Tokio is a third-party crate** (`tokio`), the most widely used async runtime.
It provides the executor, async I/O, timers, tasks, and synchronization
primitives. Add it and enable features in `Cargo.toml`:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

Use the `#[tokio::main]` macro to set up a runtime and run an async `main`:

```rust
#[tokio::main]
async fn main() {
    let msg = say_hello().await;
    println!("{msg}");
}

async fn say_hello() -> String {
    "hello".to_string()
}
```

Other runtimes exist (e.g. `async-std`, `smol`), but tokio dominates the
ecosystem. Runtime choice is a project-wide decision because many async crates
target a specific runtime.

## Spawning Tasks

Spawn concurrent tasks with `tokio::spawn`. It returns a `JoinHandle` that
resolves to the task's output. Spawned tasks run concurrently and, on a
multi-threaded runtime, may run in parallel.

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // runs concurrently
        compute().await
    });

    do_other_work().await;

    let result = handle.await.unwrap(); // JoinHandle -> Result<T, JoinError>
    println!("task returned {result}");
}

async fn compute() -> i32 { 42 }
async fn do_other_work() {}
```

Data moved into a spawned task must be `Send + 'static` on the multi-threaded
runtime. For CPU-bound blocking work, use `tokio::task::spawn_blocking`.

## Concurrency: join and select

Run multiple futures concurrently on one task with `tokio::join!` (wait for
all) or `tokio::select!` (take whichever finishes first).

```rust
use tokio::join;

async fn demo_join() {
    let (a, b) = join!(fetch_a(), fetch_b()); // both concurrently, await both
    println!("{a} {b}");
}

async fn fetch_a() -> i32 { 1 }
async fn fetch_b() -> i32 { 2 }
```

`select!` polls several futures and runs the branch of the first to complete
(useful for timeouts and cancellation):

```rust
use tokio::select;
use tokio::time::{sleep, Duration};

async fn with_timeout() {
    select! {
        result = long_task() => println!("done: {result}"),
        _ = sleep(Duration::from_secs(5)) => println!("timed out"),
    }
}

async fn long_task() -> i32 { 7 }
```

## Channels

Tokio provides async channels in `tokio::sync`:

| Channel     | Use case                                  |
| ----------- | ----------------------------------------- |
| `mpsc`      | multi-producer, single-consumer stream    |
| `oneshot`   | single value sent once (request/response) |
| `broadcast` | multi-producer, multi-consumer fan-out    |
| `watch`     | latest-value updates to many receivers    |

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<i32>(32); // bounded, capacity 32

    tokio::spawn(async move {
        for i in 0..3 {
            tx.send(i).await.unwrap();
        }
    });

    while let Some(v) = rx.recv().await {
        println!("received {v}");
    }
}
```

## Async in Traits

As of Rust 1.75, **`async fn` in traits is stable** for many cases — you can
write `async fn` directly in a trait definition and impl:

```rust
trait Fetcher {
    async fn fetch(&self, url: &str) -> String;
}

struct Http;

impl Fetcher for Http {
    async fn fetch(&self, url: &str) -> String {
        format!("body of {url}")
    }
}
```

Caveats: `async fn` in *public* traits does not yet let you add explicit `Send`
bounds on the returned future, and such traits are not automatically usable as
`dyn Trait`. For those cases, the third-party `async-trait` crate (which boxes
the returned future) remains a common workaround. Return-position `impl Trait`
in traits (RPITIT) is also stable and underpins this feature.

## Blocking vs Async

- Never call blocking APIs (e.g. `std::thread::sleep`, blocking file/DB calls)
  directly inside async tasks — they stall the executor thread. Use the async
  equivalents (`tokio::time::sleep`) or `spawn_blocking`.
- Async shines for I/O-bound concurrency (network, files); for CPU-bound
  parallelism prefer threads or a pool like `rayon` (third-party).

## See Also

- Async Book: https://rust-lang.github.io/async-book/
- Tokio Tutorial: https://tokio.rs/tokio/tutorial
- `std::future`: https://doc.rust-lang.org/std/future/index.html
