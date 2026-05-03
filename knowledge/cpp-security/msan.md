# C++ Security - MemorySanitizer (MSan)

> **Source**: https://clang.llvm.org/docs/MemorySanitizer.html
> **Skill**: dev-suite skill `security/cpp-security` — see SKILL.md for the always-loaded quick reference.

## What this covers

What MSan instruments to detect uninitialized reads, why it requires the *entire* runtime to be
instrumented (including libc++), how to build a usable MSan environment with `libcxx`+`libcxxabi`,
and when MSan is worth the operational cost vs alternatives.

## Deep dive

### Why MSan is harder to deploy than ASan

ASan only needs the executable instrumented — runtime hooks intercept malloc and read/write.
MSan tracks the *initialization state* of every byte; if any uninstrumented function (libc, libstdc++,
a vendored .so) writes to memory MSan thinks is uninitialized, MSan still considers it
uninitialized. Result: false positives every time you call into uninstrumented code.

**The rule:** every TU and every shared object linked into the process must be MSan-instrumented.
That includes libc++ (or libstdc++), libpthread, plus your dependencies.

### Build flags

```bash
clang++ -fsanitize=memory \
        -fsanitize-memory-track-origins=2 \
        -fno-omit-frame-pointer \
        -O1 -g \
        myapp.cpp -o myapp
```

| Flag | Effect |
|------|--------|
| `-fsanitize=memory` | Enable MSan |
| `-fsanitize-memory-track-origins=1` | Report where the uninitialized value was created |
| `-fsanitize-memory-track-origins=2` | Also track propagation (slower, more useful) |
| `-fsanitize-memory-use-after-dtor` | Read after destructor ran (Clang 8+) |

### Building a libc++ for MSan

```bash
git clone --depth 1 https://github.com/llvm/llvm-project
cd llvm-project
cmake -G Ninja -B build_msan -S runtimes \
      -DLLVM_ENABLE_RUNTIMES='libcxx;libcxxabi;libunwind' \
      -DLLVM_USE_SANITIZER=MemoryWithOrigins \
      -DCMAKE_BUILD_TYPE=Release
ninja -C build_msan cxx cxxabi unwind
```

Then point your project at it:

```bash
clang++ -fsanitize=memory \
        -stdlib=libc++ \
        -nostdinc++ \
        -isystem /path/to/llvm-project/build_msan/include/c++/v1 \
        -L /path/to/llvm-project/build_msan/lib \
        -Wl,-rpath,/path/to/llvm-project/build_msan/lib \
        myapp.cpp -o myapp
```

This is a non-trivial setup. Many teams skip MSan in favor of ASan + UBSan, which together catch
most issues MSan would.

### What MSan catches that nothing else does

```cpp
struct Point { int x; int y; };

void example() {
    Point p;
    p.x = 1;
    use(p.y);              // MSan: use-of-uninitialized-value
}
```

```cpp
char buf[16];
read_partial(buf, 8);      // wrote bytes 0..7
hash(buf, 16);             // MSan: bytes 8..15 uninitialized
```

```cpp
std::vector<int> v(10);    // value-initialized — fine
std::vector<int> v2;
v2.reserve(10);
v2[5] = 42;                // UB regardless; MSan flags the uninitialized read
```

UBSan won't catch any of these (they aren't UB in the sense of bad arithmetic — they're "use of
indeterminate value"). ASan won't catch them (memory is allocated and within bounds).

### Suppressions / blacklists

```
# msan_blacklist.txt
src:third_party/legacy.cpp
fun:^_Z9old_codev$
```

```bash
clang++ -fsanitize=memory -fsanitize-blacklist=msan_blacklist.txt ...
```

Functions/files in the blacklist are *not instrumented*, which means MSan won't track their
output's initialization state (assumes initialized). Use sparingly — large blacklists hide bugs.

### Manual unpoisoning

For interop with uninstrumented C libraries that initialize buffers:

```cpp
#include <sanitizer/msan_interface.h>

uint8_t buf[256];
read_from_kernel(buf, sizeof(buf));    // uninstrumented syscall wrapper
__msan_unpoison(buf, sizeof(buf));     // tell MSan: this is now defined

process(buf);
```

The opposite (`__msan_poison`) is rarely needed.

### Origin tracking output

```
==12345==WARNING: MemorySanitizer: use-of-uninitialized-value
    #0 0x4011a3 in process_data ./src/processor.cpp:42
    #1 0x4017f8 in main ./src/main.cpp:18

  Uninitialized value was created by an allocation of 'buf' in the stack frame of function 'main'
    #0 0x4017a0 in main ./src/main.cpp:15
```

With `track-origins=2` you also see the propagation chain through copies/casts.

### When MSan is worth it

- Cryptographic code (uninitialized read of "random" data is a classic disaster).
- Network parsers that fill structs from raw bytes.
- Code paths through unions where field validity depends on a tag.

When ASan + UBSan is enough:
- Most application code.
- Anything where libc++/libstdc++ instrumentation cost is prohibitive.
- Projects with many third-party shared libraries.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Using system libstdc++ | Uninstrumented → constant false positives | Build instrumented libc++ |
| Running MSan binaries with uninstrumented .so | Same issue | Either build the .so with MSan or unpoison its outputs |
| Combining with ASan | Mutually exclusive | Separate builds |
| `__msan_unpoison` on too large a region | Hides real bugs | Unpoison only the bytes you actually wrote |
| MSan + dlopen of arbitrary plugins | Unpredictable | Skip MSan for plugin-host architectures |
| Not setting `track-origins` | Reports give you the use site but not the cause | `track-origins=2` is worth the slowdown |

## See also

- https://github.com/google/sanitizers/wiki/MemorySanitizerLibcxxHowTo
- https://clang.llvm.org/docs/MemorySanitizer.html#origin-tracking
