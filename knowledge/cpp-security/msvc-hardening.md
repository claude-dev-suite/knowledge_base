# C++ Security - MSVC Hardening Flags

> **Source**: https://learn.microsoft.com/cpp/build/reference/sdl-enable-additional-security-checks
> **Skill**: dev-suite skill `security/cpp-security` — see SKILL.md for the always-loaded quick reference.

## What this covers

Detailed semantics of each MSVC hardening flag (`/sdl`, `/GS`, `/Qspectre`, `/guard:cf`,
`/CETCOMPAT`), interactions between them, what runtime cost each adds, and how to set them
correctly per-target in CMake without breaking interop with libraries built without them.

## Deep dive

### `/sdl` — Security Development Lifecycle checks

`/sdl` is an umbrella that enables multiple checks beyond `/GS`:

- Promotes specific `/W4` warnings to errors (e.g. C4146 unary minus on unsigned, C4308 negative
  integral constant converted to unsigned).
- Enables `/GS`-style buffer overrun detection on additional patterns.
- Adds runtime checks for unsafe pointer assignments after free.
- Enables strict const-correctness for some const-cast patterns.

The cost is small (single-digit % code size). Always enable for production code:

```
/sdl /W4 /WX /permissive-
```

`/permissive-` is independent of `/sdl` but goes well with it — it disables MSVC's
historical compiler relaxations and aligns with ISO C++.

### `/GS` — stack canary (default; explicit is good)

Inserts a random "cookie" before the return address in functions with stack buffers; checks it on
return. Detects classic stack-smashing buffer overflows. Default-on; the modern question is *which*
functions it instruments (heuristic by default; `/GS` always; `/GS-` to disable).

`/GS` adds canaries to functions that use:
- Arrays of `char`/`wchar_t`/`int`/`float`/`double` >= 4 bytes
- Local variables whose address is taken
- Anything with `_alloca`

You'll rarely need `/GS-`. Performance impact ~1-2% on stack-heavy code.

### `/Qspectre` — Spectre Variant 1 mitigation

Inserts speculation barriers (`lfence`) at indirect-branch sites that the compiler determines
might be Spectre-Variant-1 gadgets. Cost: noticeable on hot paths (5-15% on indirect-branch-heavy
code).

```
/Qspectre              # mitigation for Spectre v1 only
/Qspectre-load         # also for indirect loads (more aggressive, more cost)
/Qspectre-load-cf      # for indirect loads under control flow
```

Use `/Qspectre` always; the more aggressive variants only for code processing untrusted data
(parsers, sandboxes, web servers).

### `/guard:cf` — Control Flow Guard

Validates indirect calls go to known function entries. See the dedicated `control-flow-guard.md`
KB topic for architecture; here, the build flags:

```
cl /guard:cf myapp.cpp
link /guard:cf /DYNAMICBASE myapp.obj
```

Both compile and link must have it. Without the link flag, the metadata isn't emitted and
runtime checks are no-ops.

### `/guard:ehcont` — EH Continuation Metadata (CET-aware)

Emits metadata for valid exception-handler continuation targets. Combined with CETCOMPAT and CFG,
this hardens the exception-unwinding path against ROP. Available in MSVC 16.7+.

```
/guard:ehcont
link /guard:ehcont
```

### `/CETCOMPAT` — Indirect Branch Tracking

Marks the binary as compatible with Intel CET (Control-flow Enforcement Technology) shadow stack
and IBT. Enabled at link time:

```
link /CETCOMPAT myapp.obj
```

Costs nothing in software; gains protection on CET-capable hardware (Intel Tiger Lake+, AMD Zen 3+).
Should always be on for shipped binaries.

### `/DYNAMICBASE` and `/HIGHENTROPYVA` — ASLR

`/DYNAMICBASE` (default for new projects) marks the executable for address-space layout
randomization. `/HIGHENTROPYVA` extends ASLR to 64-bit address space (more entropy, harder to
brute force):

```
link /DYNAMICBASE /HIGHENTROPYVA myapp.obj
```

Both are linker flags. `/HIGHENTROPYVA` requires a 64-bit binary.

### `/NXCOMPAT` — Data Execution Prevention

Marks the binary as DEP-compatible (no-execute on data pages). Default for modern compilers.
Together with ASLR, the foundational pair against code injection.

### Putting it together — CMake target

```cmake
if(MSVC)
    target_compile_options(myapp PRIVATE
        /W4 /WX /permissive-
        /sdl
        /guard:cf
        /guard:ehcont
        /Qspectre
        /Zc:__cplusplus       # advertise correct __cplusplus value
        /Zc:preprocessor      # standards-compliant preprocessor
    )
    target_link_options(myapp PRIVATE
        /guard:cf
        /guard:ehcont
        /CETCOMPAT
        /DYNAMICBASE
        /HIGHENTROPYVA
        /NXCOMPAT
    )
endif()
```

### Interop with non-CFG libraries

Linking a `/guard:cf` binary against a library *without* CFG metadata: the unprotected library's
indirect calls are simply unchecked. No runtime breakage, but the protection has gaps. Rebuild
dependencies with `/guard:cf` if at all possible (vcpkg supports a triplet variable for this).

### Verifying

Use `dumpbin` to confirm the metadata is in the binary:

```
dumpbin /headers myapp.exe | findstr "GUARD"
dumpbin /loadconfig myapp.exe        # shows CFG, CETCOMPAT, EH continuation tables
```

Look for `Guard CF`, `CET Compat`, `EH Continuation` flags in the output.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `/guard:cf` on compile but not link | No CFG metadata → runtime check is no-op | Set both compile AND link |
| `/Qspectre` on cold/non-trusted code | Pays cost without security benefit | Apply selectively to attack-surface modules |
| Mixing `/MD` and `/MT` across binary | C runtime mismatch — heap corruption | Pick one (`/MD` recommended) |
| `/GS-` to "speed up" inner loops | Removes critical stack protection | Profile first; rarely worth it |
| Static-link OpenSSL without `/guard:cf` | CFG gap | Build deps with same flags or use a CFG-aware vcpkg triplet |
| Missing `/permissive-` | Ships non-portable code | Always enable for new projects |

## See also

- https://learn.microsoft.com/cpp/build/reference/qspectre
- https://learn.microsoft.com/cpp/build/reference/guard-enable-control-flow-guard
