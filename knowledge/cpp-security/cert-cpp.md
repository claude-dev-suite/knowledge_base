# C++ Security - SEI CERT C++ Coding Standard

> **Source**: https://wiki.sei.cmu.edu/confluence/display/cplusplus/SEI+CERT+C%2B%2B+Coding+Standard
> **Skill**: dev-suite skill `security/cpp-security` — see SKILL.md for the always-loaded quick reference.

## What this covers

How the SEI CERT C++ ruleset is organized (rules vs recommendations, normative vs informative),
the high-impact rules organized by category, and the mapping from CERT rule IDs to clang-tidy
checks so you can enforce CERT compliance without manually grepping the wiki.

## Deep dive

### Rule structure

Each CERT rule has:
- A unique ID like `MEM50-CPP` (category prefix + sequential number + `-CPP`).
- A normative statement (a "shall" or "shall not").
- A non-compliant example.
- A compliant solution.
- A risk assessment: severity (Low/Medium/High), likelihood, remediation cost, priority (P1-P27),
  and a level (L1-L3).

L1 rules (P12-P27) are the highest impact — focus there first.

### Categories

| Prefix | Meaning |
|--------|---------|
| `DCL` | Declarations and Initialization |
| `EXP` | Expressions |
| `INT` | Integers |
| `CTR` | Containers |
| `STR` | Characters and Strings |
| `MEM` | Memory Management |
| `FIO` | Input/Output |
| `ERR` | Exceptions and Error Handling |
| `OOP` | Object Oriented Programming |
| `CON` | Concurrency |
| `MSC` | Miscellaneous |

### High-impact L1 rules with C++ idioms

#### `DCL30-CPP` — Declare objects with appropriate storage durations

```cpp
// BAD: returning a reference to a local
const std::string& greeting() {
    std::string s = "hello";
    return s;
}

// GOOD: return by value (RVO/NRVO eliminates the copy)
std::string greeting() { return "hello"; }
```

#### `EXP54-CPP` — Do not access an object outside of its lifetime

```cpp
// BAD: dangling reference to temporary's underlying buffer
auto&& s = std::string("temp").c_str();    // string is destroyed end-of-statement

// GOOD: bind by value
std::string owned = "temp";
const char* s = owned.c_str();
```

#### `INT30-C` (also applies to C++) — Ensure unsigned arithmetic does not wrap

```cpp
// BAD
size_t n = container.size() - 1;            // wraps when empty

// GOOD
if (!container.empty()) {
    size_t n = container.size() - 1;
}
```

#### `MEM50-CPP` — Do not access freed memory

Use RAII / smart pointers; never write code that accesses memory after `delete`. clang-tidy's
`bugprone-use-after-move` catches the move variant; sanitizers catch the runtime form.

#### `OOP54-CPP` — Gracefully handle self-copy assignment

```cpp
class Buffer {
    std::vector<int> data_;
public:
    Buffer& operator=(const Buffer& other) {
        if (this == &other) return *this;       // self-assign guard
        data_ = other.data_;
        return *this;
    }
    // Even better: rule-of-zero — let the compiler generate
};
```

The "rule of zero" approach (let `std::vector` handle copy) sidesteps OOP54 entirely.

#### `CON53-CPP` — Avoid deadlock by locking in a predefined order

```cpp
// BAD: thread A locks m1 then m2; thread B locks m2 then m1
{ std::lock_guard a{m1}; std::lock_guard b{m2}; ... }
{ std::lock_guard a{m2}; std::lock_guard b{m1}; ... }

// GOOD: std::scoped_lock acquires in deadlock-free order
std::scoped_lock locks{m1, m2};
```

#### `STR50-CPP` — Guarantee null-termination

When passing C strings across boundaries, never assume `strncpy`/`strncat` null-terminate:

```cpp
// std::string is null-terminated by guarantee; prefer it
std::string user_input(buf, n);    // may not be terminated; this constructor is fine
const char* c_str = user_input.c_str();   // always terminated
```

#### `ERR58-CPP` — Handle all exceptions thrown before main begins

Static initializers can throw. The terminate handler runs before `main`, with no debugger
attached. Wrap risky inits:

```cpp
namespace {
    const auto kConfig = []() noexcept {
        try { return load_config(); }
        catch (...) { std::terminate(); /* logged elsewhere */ }
    }();
}
```

### Mapping CERT to clang-tidy

Many CERT rules have direct clang-tidy aliases. Enable them all:

```yaml
Checks: '-*,cert-*'
```

Examples of the mapping:

| CERT rule | clang-tidy check |
|-----------|------------------|
| OOP54-CPP | `cert-oop54-cpp` |
| ERR58-CPP | `cert-err58-cpp` |
| ERR60-CPP | `cert-err60-cpp` |
| MSC32-C | `cert-msc32-c` |
| DCL58-CPP | `cert-dcl58-cpp` |

For rules without a direct mapping, the closest `bugprone-*` or `cppcoreguidelines-*` check
usually covers the same pattern. Compliance reports often cite both.

### When CERT compliance is a contractual requirement

For automotive (ISO 26262), aerospace (DO-178C), or medical-device code:

1. Enable `cert-*` in `.clang-tidy` with `WarningsAsErrors: 'cert-*'`.
2. Run `cppcheck --addon=cert` for the rules clang-tidy doesn't cover.
3. Document deviations: per-line `// NOLINT(cert-XXX-cpp)` with a reference to a tracked
   exemption.
4. Periodically audit the deviation list against the rule rationales.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Treating CERT as advisory | "Recommendations" can be skipped, "Rules" cannot — but priority matters | Focus on L1 (P12-P27) first |
| Suppressing rules without exemption record | No audit trail for compliance | Maintain a deviations doc |
| `cert-*` enabled, ruleset versions drifting | Rule numbers stable; check semantics evolve | Pin clang-tidy version in CI |
| OOP54 implementation that uses copy-and-swap | More elegant but `noexcept`-trap | If you can't make swap noexcept, fall back to self-check |
| ERR58 ignored because "the static can't throw" | Future maintainer breaks the assumption | Wrap with `noexcept` lambda or move to lazy init |

## See also

- https://wiki.sei.cmu.edu/confluence/display/cplusplus/Rules+versus+Recommendations
- https://clang.llvm.org/extra/clang-tidy/checks/cert/
