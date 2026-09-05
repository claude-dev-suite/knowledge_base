# What Each Language's Default Analysis Actually Covers

> No upstream. Every tool documents its own rules; nobody publishes the
> comparison, because no vendor has a reason to tell you how much their
> defaults leave out.

## Why this page exists

The most useful question in a code review is not "what is wrong here" but
**"has something already said this?"** — because a comment repeating a linter
is noise, and a comment about a defect no tool reports is the whole value of
having a human read the diff.

The answer changes per language, and it changes more than people expect. Two
projects in different languages, both with "linting set up", can have wildly
different amounts actually checked. This page is the map.

## The short version

| Language | Out of the box | The trap |
|---|---|---|
| **Rust** | Very broad — the compiler plus clippy's default groups | Almost nothing left; reviewing it like C++ produces noise |
| **Go** | Broad and stable — `go vet` runs inside `go test` | The gap is small but real: nothing checks a *missing* call |
| **C#** | Analysers on by default since .NET 5 | Nullable reference types are a per-project switch |
| **TypeScript** | Depends entirely on `tsconfig.json` | Two settings that matter most are **not** in `strict` |
| **Kotlin** | Compiler is strong on null and exhaustiveness | Its guarantees stop at the Java boundary and at concurrency |
| **Swift** | Strong on null and exhaustiveness | Concurrency checking is a per-target setting |
| **Python** | Narrow — ruff's default is `E4, E7, E9, F` | Every famous footgun lives in an opt-in family |
| **Java** | Almost nothing | SpotBugs, Error Prone, NullAway are separate build steps |
| **C++** | Nothing, unless the build asks | Warnings are off by default |
| **SQL** | Nothing checks meaning | The parser accepts any well-formed query |

The ordering matters: it is roughly "how much of the review is already done for
you". At the top, restraint is the skill. At the bottom, the reviewer *is* the
analysis.

## Per language, and what to read to confirm it

### Rust — the most covered

`rustc` rejects use-after-move, dangling references and cross-thread data races
outright; `unused_must_use` makes an unhandled `Result` a warning. Clippy's
default groups (`correctness`, `suspicious`, `style`, `complexity`, `perf`) are
broad; `pedantic`, `nursery` and `cargo` are opt-in.

**Read**: `#![allow(...)]` at crate root, `clippy.toml`, `[lints]` in
`Cargo.toml`, and whether CI runs `cargo clippy -- -D warnings`. Also
`[profile.release] overflow-checks` — off by default, which is why arithmetic
behaves differently in the binary users run than under `cargo test`.

### Go — broad, and it runs whether you ask or not

A subset of `go vet` runs automatically as part of `go test`, so
`printf`, `copylocks`, `loopclosure` and friends are reported in any project
with tests. golangci-lint's default set adds `errcheck`, `staticcheck`,
`ineffassign` and `unused`.

**Read**: `.golangci.yml` for what was enabled or disabled, and the `go`
directive in `go.mod` — it, not the installed toolchain, selects language
semantics such as per-iteration loop variables (1.22+).

### C# — on by default, with one switch that is not

.NET SDK analysers are enabled by default for projects targeting .NET 5 and
later, so a subset of `CA` rules runs with no configuration.

Nullable reference types are separate: enabled in project templates created
from .NET 6 onward, and **absent from anything created earlier or upgraded in
place**. Without it there are no null guarantees at all.

**Read**: `<Nullable>`, `<AnalysisLevel>`, `<EnableNETAnalyzers>` and
`<TreatWarningsAsErrors>` in the `.csproj` and any `Directory.Build.props`.

### TypeScript — the compiler's strictness is a setting

`strict` turns on `strictNullChecks`, `noImplicitAny` and
`useUnknownInCatchVariables`. Two of the settings that matter most for review
are **not** in it:

- `noUncheckedIndexedAccess` — off, `rows[0]` is typed as present, so every
  array access is an unverified claim;
- `exactOptionalPropertyTypes` — off, an optional property silently accepts an
  explicit `undefined`.

Separately, typescript-eslint's most valuable rules — `no-floating-promises`,
`await-thenable` — are **type-aware** and need `parserOptions.project`. Many
repos never enable it, so a clean lint says nothing about async correctness.

**Read**: the `tsconfig.json` the reviewed file actually resolves to (monorepos
have several, and `extends` chains matter), and the ESLint config for whether
type-checked linting is on.

### Kotlin and Swift — strong compilers, bounded guarantees

Both close the null hole the languages they replaced left open, and both stop
at the same two places.

**Kotlin**: a platform type from unannotated Java (`String!`) is neither
nullable nor non-null — it opts *out* of the check, so Kotlin's central
guarantee does not apply to values from JPA entities, Jackson, or older SDKs.
`-Xjsr305=strict` changes that. detekt is a separate plugin, and `!!` is
reported by a rule that is **not** in its default ruleset.

**Swift**: SwiftLint's `force_unwrapping` is likewise **not** a default rule.
More importantly, data-race checking is a per-target setting
(`SWIFT_STRICT_CONCURRENCY`: `minimal` / `targeted` / `complete`) or the Swift 6
language mode. Under Swift 5 with minimal checking, none of it is enforced.

**Read**: `build.gradle.kts` (detekt plugin, `detekt.yml`, `-Xjsr305`) or the
target's Swift language mode and strict-concurrency level.

### Python — the narrowest default of any modern linter

Ruff's default `select` is `E4, E7, E9, F`. That is it. Every well-known Python
footgun lives in an opt-in family:

| Defect | Rule | Family | In default? |
|---|---|---|---|
| Mutable default argument | `B006` | bugbear | no |
| Naive `datetime.now()` | `DTZ005` | flake8-datetimez | no |
| Blocking call in `async def` | `ASYNC2xx` | flake8-async | no |
| `subprocess(shell=True)` | `S602` | bandit | no |

mypy adds nothing unless the project runs it, and an *unannotated* function is
not checked at all even when it does — so a wrong type hint elsewhere is never
contradicted.

**Read**: `[tool.ruff.lint] select` and `[tool.mypy]` in `pyproject.toml`.

### Java — the analysis is absent, not narrow

`javac` reports very little; `-Xlint` adds a few categories and is not on by
default. SpotBugs, Error Prone and NullAway are **separate build steps**, and a
plain Spring Boot starter has none of them. So `equals` without `hashCode`,
boxed `==` and string comparison by reference are review findings in most real
projects.

**Read**: `pom.xml` / `build.gradle` for the plugins, and
`maven.compiler.release` — not the installed JDK — for what language features
are even available.

### C++ — nothing, unless the build asks for it

A CMake project that never sets `-Wall -Wextra` compiles almost silently.
clang-tidy is a separate target. Sanitizers (ASan, UBSan) catch most lifetime
defects but **only on executed paths**, so an untested branch is exactly where
they survive.

**Read**: warning flags, a clang-tidy target or `.clang-tidy`, and whether CI
has a sanitizer build.

### SQL — nothing checks meaning

The parser accepts anything well-formed. `sqlfluff` checks layout and naming.
A query returning the wrong rows and one returning the right rows are equally
valid to every tool in the pipeline. See `sql-fundamentals/engine-differences`
for what varies underneath.

## How to use this in a review

1. **Establish the level once, from the config**, before reading the diff.
2. **State a weak configuration as one finding**, not as thirty instances of
   what it fails to catch. "`select` is the default set, so bugbear and
   datetimez never ran" is worth more than ten comments about mutable defaults.
3. **Spend the review on what nothing reports.** At the top of the table that
   is a short list; at the bottom it is most of the work.
