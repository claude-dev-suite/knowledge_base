# C++ Security - GCC/Clang Hardening Flags

> **Source**: https://wiki.gentoo.org/wiki/Hardened/Toolchain
> **Skill**: dev-suite skill `security/cpp-security` — see SKILL.md for the always-loaded quick reference.

## What this covers

The Linux/macOS hardening flag set — `_FORTIFY_SOURCE`, stack protectors, RELRO, PIE, BIND_NOW,
control-flow protection — what each catches at runtime, what their performance costs are, and
how to verify a binary actually has them via `checksec` / `hardening-check`.

## Deep dive

### `_FORTIFY_SOURCE` — libc bounds checks

Replaces calls to memcpy/strcpy/sprintf/etc. with the `_chk` variant when the compiler can
determine a buffer size at compile time. On overflow, the program is aborted.

```
-D_FORTIFY_SOURCE=2     # default level (well established)
-D_FORTIFY_SOURCE=3     # GCC 12+ / glibc 2.34+: stronger size deduction (now standard)
```

Requires `-O1` or higher (the size analysis depends on optimizer information). Cost: negligible
in practice — a few extra instructions on libc calls.

Verify a specific call is fortified by looking for `__memcpy_chk` (etc.) in the symbol table:
`objdump -d myapp | grep _chk`.

### Stack protectors

```
-fstack-protector           # canary on functions with arrays > 8 bytes (rare)
-fstack-protector-strong    # canary on more functions (recommended)
-fstack-protector-all       # canary on ALL functions (overhead noticeable)
```

`-fstack-protector-strong` covers any function with a local array, alloca, or address-taken
local. ~1% overhead, catches the canonical buffer overflow.

### Stack clash protection

```
-fstack-clash-protection
```

Probes each guard page when growing the stack, preventing "stack-clash" attacks where a large
allocation jumps over the guard page into the heap. Required on hardened distros (Ubuntu Pro,
RHEL).

### Position-Independent Executable (PIE)

```
-fPIE -pie
```

Allows the executable itself to be loaded at a randomized base (full ASLR for the executable,
not just shared libraries). Required for Android NDK 16+. Modern distros ship PIE-default GCC.

Cost: small register pressure increase on x86; near-zero on x86_64/ARM64.

### RELRO and BIND_NOW

```
-Wl,-z,relro          # makes GOT read-only after relocation (partial RELRO)
-Wl,-z,now            # resolves all symbols at load (full RELRO when combined with relro)
```

Without `relro`, the GOT (Global Offset Table) stays writable after dynamic linking, giving
exploits a rich target. Full RELRO (`-z relro,-z now`) forces all dynamic symbols to be resolved
at load time, then makes the GOT read-only.

Cost: load-time slowdown (all symbols resolved up front). Negligible at runtime.

### Non-executable stack

```
-Wl,-z,noexecstack
```

Marks the stack as non-executable in the ELF header. Default on most modern toolchains, but
becomes "executable stack" if any linked object lacks the marker (one assembly file forgetting
`.section .note.GNU-stack,"",%progbits` is enough).

Verify: `execstack -q myapp` or `readelf -W -l myapp | grep STACK`.

### Control-flow protection (CET on x86)

```
-fcf-protection=full
```

Emits `endbr64` at function entries (Intel CET IBT) and uses the shadow stack for returns
(Intel CET SHSTK / AMD equivalent). Effective only on CET-capable CPUs (Intel Tiger Lake+,
AMD Zen 3+) running on a CET-aware kernel.

Cost: code-size increase (~3-5% from `endbr64` instructions); near-zero CPU cost.

### Format string warnings

```
-Wformat -Wformat=2 -Wformat-security
```

Compiler warnings, not runtime checks. `-Wformat-security` flags `printf(x)` where `x` is not a
literal — a classic format-string vulnerability. Combine with `-Werror=format-security` to make
this fatal at build time.

### Putting it together — production target

```cmake
if(NOT MSVC)
    target_compile_options(myapp PRIVATE
        -O2 -g
        -D_FORTIFY_SOURCE=3
        -fstack-protector-strong
        -fstack-clash-protection
        -fcf-protection=full
        -fPIE
        -Wformat -Wformat=2 -Wformat-security
        -Werror=format-security
    )
    target_link_options(myapp PRIVATE
        -pie
        -Wl,-z,relro,-z,now
        -Wl,-z,noexecstack
        -Wl,-z,separate-code      # split RX and R sections (binutils 2.31+)
    )
endif()
```

### Verifying with `checksec` / `hardening-check`

```bash
# checksec — comprehensive output
checksec --file=./myapp
# RELRO   STACK CANARY   NX   PIE   FORTIFY   FORTIFIED   RPATH   RUNPATH   SYMBOLS
# Full    Canary         NX   PIE   Yes        17 / 53     No      No        Yes

# hardening-check (Debian)
hardening-check ./myapp
```

Aim for: Full RELRO, Canary, NX, PIE, FORTIFIED count > 0, no RUNPATH/RPATH (avoids attacker
controlling library search).

### Per-flag cost summary

| Flag | Runtime cost |
|------|--------------|
| `-D_FORTIFY_SOURCE=3` | <1% |
| `-fstack-protector-strong` | ~1% |
| `-fstack-clash-protection` | <1% |
| `-fPIE -pie` | ~1% on x86; ~0 on x86_64/ARM64 |
| `-Wl,-z,relro,-z,now` | Load-time only |
| `-fcf-protection=full` | ~0% |

Combined: typically 2-3% total — well worth it.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `_FORTIFY_SOURCE` without `-O1+` | Silent no-op (no size info) | Always combine with optimization |
| Forgetting `-pie` (linker) when using `-fPIE` | Non-PIE binary, partial ASLR | Both needed |
| Missing `-Wl,-z,noexecstack` after assembly merge | Single asm file flips entire binary to executable stack | Add `.note.GNU-stack` to your asm files |
| `-D_FORTIFY_SOURCE` redefined warning | glibc already defines it for the build | `-U_FORTIFY_SOURCE -D_FORTIFY_SOURCE=3` |
| `-fstack-protector-all` reflexively | 5-10% slowdown on tight loops | `-strong` is the right default |
| `-fcf-protection=full` on old CPUs | No security benefit; just code bloat | Profile target CPU; conditionalize |

## See also

- https://wiki.debian.org/Hardening
- https://github.com/slimm609/checksec.sh
