# C++ Security - Control Flow Guard (CFG)

> **Source**: https://learn.microsoft.com/windows/win32/secbp/control-flow-guard
> **Skill**: dev-suite skill `security/cpp-security` — see SKILL.md for the always-loaded quick reference.

## What this covers

The architecture of Windows Control Flow Guard — how the OS, compiler, and loader cooperate to
validate indirect calls — what CFG protects against (and doesn't), and the C++ patterns that
play badly with CFG and require explicit annotations.

## Deep dive

### The threat CFG addresses

Memory-corruption exploits often hijack an indirect call (vtable pointer, function pointer in a
struct on the heap) to redirect execution to attacker-chosen code (e.g. ROP gadgets or shellcode).
CFG validates that an indirect call goes to a *function start that the compiler knew about at
build time*.

CFG does not protect *direct* calls (already statically verifiable), nor does it protect against
arbitrary code execution if the attacker can JIT new code. It also doesn't protect *return*
addresses — that's the shadow stack via Intel CET (`/CETCOMPAT`).

### How it works

1. **Compile time:** `/guard:cf` makes the compiler emit a small check before every indirect call.
2. **Link time:** `/guard:cf` makes the linker emit a bitmap (the CFG bitmap) listing every valid
   call target in the image.
3. **Load time:** Windows maps the CFG bitmap and replaces the per-call check function with a fast
   path that consults the bitmap.
4. **Runtime:** for each indirect call site, the inserted check looks up the target address in the
   bitmap. If absent → process termination via `__guard_check_icall_fptr`.

The bitmap has 1 bit per 16 bytes of executable memory. Function entries are aligned and marked.
Cost: ~1-3% on indirect-call-heavy code; bitmap adds ~1.5% to file size.

### Build flags

```
cl /guard:cf myapp.cpp
link /guard:cf /DYNAMICBASE myapp.obj
```

`/DYNAMICBASE` is required (and is the default).

### Verifying the binary

```
dumpbin /loadconfig myapp.exe
```

Look for:
```
Guard CF Function Table
Guard CF Check Function Pointer
Guard CF Dispatch Function Pointer
0500 0000 Image flags
        100 ImageGuardCfInstrumented
        200 ImageGuardCfWInstrumented
        400 ImageGuardCfFunctionTablePresent
```

If `Guard CF Function Table` is empty or `Image flags` lacks the CF bits, CFG is *not* in effect.

### CFG-incompatible patterns

#### JIT compilers and dynamic code

```cpp
auto code = VirtualAlloc(nullptr, len, MEM_COMMIT, PAGE_EXECUTE_READWRITE);
memcpy(code, machine_code, len);
((void(*)())code)();         // CFG: target not in bitmap → process killed
```

To call dynamically generated code under CFG, you must register it:

```cpp
SetProcessValidCallTargets(GetCurrentProcess(),
                           code, len,
                           1, &offset);
```

`SetProcessValidCallTargets` requires the page to be `MEM_COMMIT`-allocated with
`PAGE_TARGETS_INVALID` and the process started with the appropriate mitigation policy.

#### Hot-patching and detours

CFG-marked functions can't be in-place hot-patched (the bitmap has the original address).
Workaround: the hooking library (Detours, MinHook) must call `SetProcessValidCallTargets` on the
trampoline.

#### Member function pointers via reinterpret_cast

```cpp
void (Foo::*p)() = ...;
auto raw = reinterpret_cast<void(*)(Foo*)>(p);    // CFG may not have this
```

The compiler can't always emit a CFG entry for raw conversions. Avoid the cast; use member-pointer
invocation syntax.

### Disabling CFG per-function

For functions that legitimately need to call into untrusted-but-controlled targets (a JIT host):

```cpp
__declspec(guard(nocf)) void run_jit_code(void* code) {
    ((void(*)())code)();
}
```

This disables CFG check at this call site only. Use sparingly and with a comment justifying why.

### CFG vs Intel CET (`/CETCOMPAT`)

| Mechanism | Protects | Implementation |
|-----------|----------|----------------|
| `/guard:cf` | Indirect *forward* calls | Software bitmap |
| `/CETCOMPAT` IBT | Indirect calls to non-`endbr64` targets | Hardware (CET IBT) |
| `/CETCOMPAT` SHSTK | Return-address corruption | Hardware shadow stack |
| `/guard:ehcont` | Exception-handler continuation | Software metadata table |

These are complementary. Modern Windows binaries should enable all four.

### Performance budget

| Workload | CFG overhead |
|----------|--------------|
| Computation-bound | <1% |
| OOP-heavy (vtables) | 2-5% |
| Plugin-host with many indirect calls | up to 8% |

The cost is dominated by call-site instructions, not the bitmap lookup (which is L1-cache hot).

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `/guard:cf` only at compile | Bitmap not emitted; runtime check is no-op | Always at link too |
| Linking a non-CFG static library into a CFG executable | Indirect calls in the library are unprotected | Rebuild the library with `/guard:cf` |
| JIT'd code crashing on first call | Not in CFG bitmap | `SetProcessValidCallTargets` |
| Missing `/DYNAMICBASE` | CFG requires ASLR | Default-on; don't add `/DYNAMICBASE:NO` |
| Trusting `dumpbin` to "look enabled" | The flag is set but the table is empty | Verify the bitmap entries, not just the flag |
| `__declspec(guard(nocf))` to silence "false positives" | Almost always real bugs | Investigate the indirect target instead |

## See also

- https://learn.microsoft.com/windows/win32/secbp/control-flow-guard
- https://learn.microsoft.com/windows/win32/api/processthreadsapi/nf-processthreadsapi-setprocessvalidcalltargets
