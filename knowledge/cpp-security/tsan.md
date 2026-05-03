# C++ Security - ThreadSanitizer (TSan)

> **Source**: https://clang.llvm.org/docs/ThreadSanitizer.html
> **Skill**: dev-suite skill `security/cpp-security` — see SKILL.md for the always-loaded quick reference.

## What this covers

How TSan models the happens-before relationship to detect races, why it cannot run together with
ASan/MSan, common patterns that look like races but aren't (and vice versa), suppressions, and
strategies for making race detection deterministic in CI.

## Deep dive

### What TSan actually checks

TSan instruments every memory access and every synchronization primitive (`std::mutex`,
`std::atomic`, `pthread_*`, condition variables) to build a happens-before graph. A *data race*
is reported when:

1. Two threads access the same memory location.
2. At least one is a write.
3. There is no happens-before edge between them.

This is the C++ memory model definition. TSan reports both **races** and **deadlocks** (with
`detect_deadlocks=1`).

### Build flags

```bash
clang++ -fsanitize=thread \
        -fno-omit-frame-pointer \
        -O1 -g \
        myapp.cpp -o myapp
```

Compile and link with `-fsanitize=thread`. Cannot combine with ASan, MSan, or memory hardware
sanitizer — they all hijack the same memory subsystem.

### Runtime options

```bash
TSAN_OPTIONS="\
  halt_on_error=1:\
  second_deadlock_stack=1:\
  history_size=7:\
  report_atomic_races=1:\
  io_sync=2\
" ./myapp
```

| Option | Effect |
|--------|--------|
| `halt_on_error=1` | Stop on first race (default 0) |
| `history_size` | 0-7; bigger → catches races with longer time gap, more memory |
| `second_deadlock_stack=1` | Show stacks of both lock acquisitions in deadlock report |
| `io_sync` | 0/1/2 — model file I/O as synchronization (2 = full, slowest, most accurate) |
| `report_atomic_races=1` | Catches misuse of `memory_order_relaxed` mixing |

### Common false positives (and what they actually mean)

**1. "Race on `errno`":** if you've set `errno` from one thread and read it from another.
`errno` is thread-local in C11/C++11 — TSan rarely flags this unless you're using a non-conforming
libc or `__attribute__((no_thread_safety_analysis))`-stripped wrappers.

**2. "Race on initialization":** static local variables are thread-safe under C++11 (Itanium ABI),
but TSan may flag a mid-init read if compilers have inlined the guard. Modern Clang elides this.

**3. "Race on free in non-aliased pool":** custom allocators that recycle memory across threads
without releasing the synchronization (e.g. a TLS pool) genuinely have UB by the C++ memory model
even if the algorithm is correct. Annotate or refactor.

### Suppressions

```
# tsan.supp
race:vendor/third_party_lib.so
race:^my_legacy_class::counter$
mutex:^libfoo_init$
deadlock:bar.cpp
```

```bash
TSAN_OPTIONS=suppressions=tsan.supp:print_suppressions=1 ./myapp
```

### Annotation API for hand-rolled synchronization

Custom lock-free structures (e.g. SPSC queue with `std::atomic` + `memory_order_acquire/release`)
sometimes confuse TSan because the synchronization happens through a pattern it can't model.
Annotate explicitly:

```cpp
#include <sanitizer/tsan_interface.h>

void publish() {
    data_ = compute();
    __tsan_release(&ready_);
    ready_.store(1, std::memory_order_release);
}

void consume() {
    while (!ready_.load(std::memory_order_acquire)) ;
    __tsan_acquire(&ready_);
    use(data_);
}
```

Or use `std::atomic` exclusively — TSan understands `std::atomic` orderings natively. Manual
annotation is for cases the type system can't express.

### Determinism for CI

Race tests are flaky by nature. Three techniques:

1. **`--gtest_repeat=20 --gtest_shuffle`** in CI for any test that exercises threading.
2. **Stress-test wrappers** like `rr` (record-and-replay debugger) for postmortem.
3. **`TSAN_OPTIONS=halt_on_error=0`** in long stress runs to surface multiple races at once.

A test that "occasionally" passes under TSan is not race-free; a single failed run is the bug.

### Performance

Slowdown 5-15x; memory overhead 5-10x. Not for production. Carve out a TSan-only CI job; don't
run the full test suite under TSan unless threading is core to the codebase.

### What TSan cannot detect

- Races in code that wasn't compiled with TSan (your dependencies are invisible).
- Races mediated by syscalls / kernel state.
- Logic bugs that look correct under the memory model but violate domain invariants.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Combining `-fsanitize=address,thread` | Mutually exclusive — link error | Two separate build configs / CI jobs |
| Running TSan with un-instrumented .so | Races inside libfoo invisible | Build everything (or relevant libs) with TSan |
| `volatile` to "fix" a race | volatile has nothing to do with threads in C++ | Use `std::atomic` |
| TSan on production binary | 5-15x slowdown | TSan in CI only |
| Treating TSan flake as "intermittent" | Races that surface 1% of runs are still bugs | One report = one bug |
| Suppressing races without understanding | Hides real corruption | Triage every race; suppress only with comment + ticket |

## See also

- https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual
- https://en.cppreference.com/w/cpp/atomic/memory_order
