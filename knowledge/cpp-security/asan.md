# C++ Security - AddressSanitizer (ASan)

> **Source**: https://clang.llvm.org/docs/AddressSanitizer.html
> **Skill**: dev-suite skill `security/cpp-security` — see SKILL.md for the always-loaded quick reference.

## What this covers

What ASan instruments (heap, stack, globals), tunable runtime options via `ASAN_OPTIONS`, the
suppression file format, what ASan **cannot** detect, and how to combine it with leak detection
(LSan) and debuggers without losing diagnostics.

## Deep dive

### What ASan detects

| Bug | Detected? |
|-----|-----------|
| Heap-buffer-overflow | Yes |
| Stack-buffer-overflow (incl. uses-after-return*) | Yes |
| Global-buffer-overflow | Yes |
| Use-after-free (heap) | Yes |
| Use-after-return (stack) | Yes (with `-fsanitize-address-use-after-return=always`) |
| Use-after-scope | Yes (with `-fsanitize-address-use-after-scope`) |
| Double-free / invalid-free | Yes |
| Memory leaks | Yes via integrated LSan (Linux/macOS) |
| Container OOB (`std::vector::operator[]`) | Yes if compiled with `_GLIBCXX_DEBUG`/`_LIBCPP_HARDENING_MODE` |
| Uninitialized reads | **No** — that's MSan |
| Data races | **No** — that's TSan |

The "use-after-return" detection requires `-fsanitize-address-use-after-return=always` (Clang 13+).
This uses fake stacks; runtime cost roughly 1.3x ASan baseline.

### Build flags

```bash
clang++ -fsanitize=address \
        -fsanitize-address-use-after-scope \
        -fno-omit-frame-pointer \
        -O1 -g \
        myapp.cpp -o myapp
```

`-O1` is recommended over `-O0` (faster, still debuggable). `-fno-omit-frame-pointer` is critical
for readable stack traces.

For shared libraries: link the **executable** with `-fsanitize=address`. Linking only the .so
will produce "ASan runtime missing" at startup.

### Runtime options

```bash
ASAN_OPTIONS="\
  detect_leaks=1:\
  abort_on_error=1:\
  strict_string_checks=1:\
  detect_stack_use_after_return=1:\
  check_initialization_order=1:\
  strict_init_order=1:\
  print_stacktrace=1:\
  halt_on_error=1\
" ./myapp
```

Highest-value options:

| Option | Effect |
|--------|--------|
| `detect_leaks=1` | Run LSan at exit (Linux/macOS) |
| `abort_on_error=1` | Crash with core dump on first error (debugger friendly) |
| `halt_on_error=1` | Stop after first error (default 1; set 0 to find more in one run) |
| `strict_string_checks=1` | Bounds-check `strlen`/`strcat` arguments |
| `check_initialization_order=1` | Detects static-init-order fiasco across TUs |
| `print_legend=1` | Annotates the shadow memory dump in error reports |
| `log_path=/tmp/asan` | Write to file instead of stderr |
| `symbolize=1` | Resolve frame addresses to source lines (default) |

### Suppressions

Two kinds of suppression files:

**ASan suppressions** (`asan.supp`): suppress *false positives* in the analyzer itself
(e.g. interceptor issues with strange libc). Rare.

**LSan suppressions** (`lsan.supp`): suppress known leaks in third-party libs:

```
# lsan.supp
leak:libGL.so
leak:vendor/legacy_init
leak:^xmlInitParser$
```

```bash
LSAN_OPTIONS=suppressions=lsan.supp:print_suppressions=1 ./myapp
```

### Symbolization

ASan needs `llvm-symbolizer` (or `addr2line`) on PATH. If frames show as `??:0`:

```bash
export ASAN_SYMBOLIZER_PATH=$(which llvm-symbolizer)
# or copy the binary next to your app
```

On macOS, atos is used automatically.

### Combining with debuggers

```bash
ASAN_OPTIONS=abort_on_error=1:disable_coredump=0 ./myapp
gdb -ex 'run' -ex 'bt' --args ./myapp

# Or have ASan trigger a debugger on error
ASAN_OPTIONS=abort_on_error=0:debug=1 lldb ./myapp
(lldb) break set -n __asan_on_error
```

### What ASan cannot detect

- Reads of uninitialized memory (use MSan)
- Data races (use TSan)
- Logic bugs (wrong index that's still in bounds)
- Memory corruption from kernel/syscalls (passes through)
- Issues in code linked from non-ASan-instrumented libraries (silent)

### Performance budget

| Workload | ASan slowdown |
|----------|---------------|
| CPU-bound | ~2x |
| Allocation-heavy | 3-5x |
| With UAR enabled | 2.5x |
| With LSan periodic scan | 2.2x |

Memory overhead: ~3x heap size (shadow memory). Not suitable for production; perfect for CI and
local dev.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `-fsanitize=address` on compile but not link | Missing ASan runtime → "interceptor not installed" | Add to both compile AND link |
| Mixed instrumented/uninstrumented TUs | Silent: bugs in uninstrumented code missed | Build entire binary with ASan |
| `_FORTIFY_SOURCE` + ASan | Conflicts (both intercept some libc) | Build ASan with `-U_FORTIFY_SOURCE` |
| ASan + dlopen of uninstrumented library | LD_PRELOAD ordering issues | `LD_PRELOAD=$(clang -print-file-name=libclang_rt.asan-x86_64.so)` |
| Frames show ??:0 | symbolizer not found | Set `ASAN_SYMBOLIZER_PATH` |
| ASan + jemalloc/tcmalloc | Two allocators fighting | Disable the alternate allocator under ASan |
| LSan suppression catching too much | Hides real leaks too | Use precise prefixes; verify with `print_suppressions=1` |

## See also

- https://github.com/google/sanitizers/wiki/AddressSanitizer
- https://clang.llvm.org/docs/AddressSanitizerFlags.html
