# Rust Cargo

> Official Documentation: https://doc.rust-lang.org/cargo/

## Overview

Cargo is Rust's build system and package manager. It creates projects, resolves
and downloads dependencies (from crates.io by default), compiles code, runs
tests and benchmarks, and publishes crates. A Rust package is defined by a
`Cargo.toml` manifest; `Cargo.lock` records exact resolved versions.

Targets current stable Rust (1.97, edition 2024).

## Common Commands

| Command             | Purpose                                             |
| ------------------- | --------------------------------------------------- |
| `cargo new <name>`  | Create a new package (binary by default)            |
| `cargo new --lib <name>` | Create a new library package                   |
| `cargo init`        | Initialize a package in the current directory       |
| `cargo build`       | Compile (debug profile) into `target/debug`         |
| `cargo build --release` | Optimized build into `target/release`           |
| `cargo run`         | Build and run the binary                            |
| `cargo check`       | Type-check without producing a binary (fast)        |
| `cargo test`        | Build and run tests                                 |
| `cargo bench`       | Run benchmarks (nightly for built-in `#[bench]`)    |
| `cargo doc --open`  | Build and open documentation                        |
| `cargo add <crate>` | Add a dependency to `Cargo.toml`                    |
| `cargo remove <crate>` | Remove a dependency                              |
| `cargo update`      | Update dependencies within semver constraints       |
| `cargo fmt`         | Format code (rustfmt)                               |
| `cargo clippy`      | Lint with Clippy                                     |
| `cargo publish`     | Publish a crate to crates.io                        |
| `cargo install <crate>` | Install a binary crate globally                 |
| `cargo clean`       | Remove the `target/` build directory                |

```bash
cargo new my_app
cd my_app
cargo add serde --features derive
cargo run
```

## Cargo.toml

The manifest describes the package and its dependencies.

```toml
[package]
name = "my_app"
version = "0.1.0"
edition = "2024"        # current edition
rust-version = "1.85"   # optional MSRV (minimum supported Rust version)
authors = ["Jane Doe <jane@example.com>"]
license = "MIT OR Apache-2.0"
description = "An example application"

[dependencies]
serde = { version = "1", features = ["derive"] }
rand = "0.8"

[dev-dependencies]         # only for tests, examples, benches
criterion = "0.5"

[build-dependencies]       # only for build.rs
cc = "1"
```

> Note: the `edition` (2015/2018/2021/2024) is independent of the compiler
> version. Edition 2024 is the current edition and requires Rust 1.85+.

## Dependencies

Specify versions using semver requirements:

```toml
[dependencies]
serde = "1"              # ^1: >=1.0.0, <2.0.0 (caret is the default)
regex = "1.10"           # ^1.10
exact = "=1.2.3"         # exactly 1.2.3
tokio = { version = "1", features = ["full"] }

# Non-registry sources
local = { path = "../local_crate" }
git_dep = { git = "https://github.com/user/repo", branch = "main" }
```

`cargo add` edits these for you: `cargo add tokio --features full`.

## Features

Features enable optional, conditional compilation and optional dependencies.

```toml
[dependencies]
serde = { version = "1", optional = true }

[features]
default = ["std"]          # enabled unless --no-default-features
std = []
serialization = ["dep:serde"]  # enabling this feature pulls in serde
```

```bash
cargo build --features serialization
cargo build --no-default-features
cargo build --all-features
```

In code, gate items with `cfg`:

```rust
#[cfg(feature = "serialization")]
fn to_json() { /* ... */ }
```

## Workspaces

A workspace groups multiple related packages that share one `Cargo.lock` and one
`target/` directory.

```toml
# Cargo.toml at the workspace root
[workspace]
members = ["app", "core", "utils"]
resolver = "2"

[workspace.dependencies]     # shared versions
serde = { version = "1", features = ["derive"] }
```

Member crates inherit shared dependencies:

```toml
# app/Cargo.toml
[dependencies]
serde = { workspace = true }
```

Run commands across the workspace:

```bash
cargo build --workspace
cargo test -p core          # target a single package
```

## Profiles

Profiles control compiler settings for different build types. Customize them in
`Cargo.toml`.

```toml
[profile.dev]        # cargo build
opt-level = 0
debug = true

[profile.release]    # cargo build --release
opt-level = 3
lto = true           # link-time optimization
codegen-units = 1
panic = "abort"      # smaller/faster; disables unwinding
strip = true         # strip symbols
```

| Profile   | Default use            | Optimization |
| --------- | ---------------------- | ------------ |
| `dev`     | `cargo build`/`run`    | none, debug  |
| `release` | `--release`            | full         |
| `test`    | `cargo test`           | none, debug  |
| `bench`   | `cargo bench`          | full         |

## Testing and Benchmarking

```bash
cargo test                      # run all tests
cargo test some_name            # run tests matching a filter
cargo test -- --nocapture       # show stdout from passing tests
cargo test --doc                # run only doctests
```

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn adds() {
        assert_eq!(2 + 2, 4);
    }
}
```

Benchmarks: stable Rust runs benches via third-party harnesses like `criterion`
(a dev-dependency). The built-in `#[bench]` attribute requires nightly.

## Publishing to crates.io

```bash
cargo login <API_TOKEN>   # get a token from crates.io account settings
cargo package             # build a distributable .crate and verify locally
cargo publish             # upload to crates.io (irreversible; versions are permanent)
```

Before publishing, ensure `Cargo.toml` has `name`, `version`, `license` (or
`license-file`), and `description`. Use `cargo yank --version X.Y.Z` to prevent
new projects from selecting a broken release (it does not delete it).

## Useful Extras

- `cargo tree` — show the dependency graph.
- `cargo search <term>` — search crates.io.
- `cargo doc` — generate docs from `///` comments.
- `rustup` manages toolchains/editions; a `rust-toolchain.toml` pins the
  toolchain per project.

## See Also

- The Cargo Book: https://doc.rust-lang.org/cargo/
- Manifest format: https://doc.rust-lang.org/cargo/reference/manifest.html
- Publishing guide: https://doc.rust-lang.org/cargo/reference/publishing.html
