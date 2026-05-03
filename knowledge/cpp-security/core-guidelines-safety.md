# C++ Security - Core Guidelines Safety Section

> **Source**: https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-safety
> **Skill**: dev-suite skill `security/cpp-security` — see SKILL.md for the always-loaded quick reference.

## What this covers

The C++ Core Guidelines safety profiles (Type, Bounds, Lifetime), how the GSL (Guidelines
Support Library) provides runtime-checked types like `gsl::span` and `gsl::not_null`, and how
to enforce the safety subset with clang-tidy and `cppcoreguidelines-*` checks.

## Deep dive

### The three safety profiles

The Core Guidelines define enforceable subsets — restrict your code to these and many classes of
bugs become impossible:

#### Type safety profile

> Every operation on a typed object is performed using an operation defined for that type.

Forbids: `reinterpret_cast`, C-style casts, unions used outside variant patterns, naked enums
implicitly convertible to int.

```cpp
// BAD
int* p = (int*)opaque_handle;
union U { int i; float f; } u; u.i = 42; use(u.f);   // type pun via union — UB

// GOOD
auto* p = static_cast<int*>(opaque_handle);
int i = 42;
float f;
std::memcpy(&f, &i, sizeof(f));     // explicit byte reinterpretation; or std::bit_cast (C++20)
```

#### Bounds safety profile

> Every operation on a buffer is in-bounds.

Forbids: pointer arithmetic outside `gsl::span`/`std::span`, arrays decaying to pointers across
boundaries, `operator[]` on raw pointers.

```cpp
// BAD
void process(int* data, std::size_t n) {
    for (std::size_t i = 0; i <= n; ++i) data[i] = 0;     // off-by-one
}

// GOOD
void process(std::span<int> data) {
    for (auto& x : data) x = 0;
}
```

#### Lifetime safety profile

> Every reference (pointer, reference, iterator, view) refers to a live object.

Forbids: returning pointers/references to locals, dangling iterators, capturing by reference into
asynchronous callbacks. Implementations include the Lifetime checker (clang's `-Wlifetime`,
experimental, and Microsoft's MSVC `/W4 /Wlifetime` warnings).

```cpp
// BAD
auto f() {
    std::string s = "x";
    return std::string_view{s};   // dangling — string destroyed
}

// GOOD: return by value, let RVO elide the copy
std::string f() { return "x"; }
```

### GSL — runtime-checked types

The Guidelines Support Library (GSL) provides:

| Type | Purpose |
|------|---------|
| `gsl::span<T>` | Pointer + size; replaced by `std::span` in C++20 |
| `gsl::not_null<T*>` | Pointer that cannot be null (constructed value is checked) |
| `gsl::owner<T*>` | Annotation: this raw pointer owns the resource |
| `gsl::czstring` | Null-terminated C string view |
| `Expects(cond)` / `Ensures(cond)` | Pre/post-condition contracts (will become `[[contract]]`) |

```cpp
#include <gsl/gsl>

void process(gsl::not_null<User*> user) {     // null impossible — checked at construction
    Expects(user->id() > 0);
    user->update();
    Ensures(user->is_dirty());
}

void caller(User* maybe_null) {
    if (!maybe_null) return;
    process(maybe_null);    // implicit conversion checks at the boundary
}
```

`Expects`/`Ensures` map to `assert()` in debug, no-op in release by default. They're documentation
that the checker can verify.

### Enforcement via clang-tidy

```yaml
Checks: >
  -*,
  cppcoreguidelines-*,
  -cppcoreguidelines-avoid-magic-numbers,
  -cppcoreguidelines-pro-bounds-array-to-pointer-decay,
  -cppcoreguidelines-pro-type-vararg
```

The `-pro-` checks are the safety profile enforcement specifically:
- `cppcoreguidelines-pro-bounds-pointer-arithmetic`
- `cppcoreguidelines-pro-bounds-constant-array-index`
- `cppcoreguidelines-pro-bounds-array-to-pointer-decay`
- `cppcoreguidelines-pro-type-cstyle-cast`
- `cppcoreguidelines-pro-type-reinterpret-cast`
- `cppcoreguidelines-pro-type-static-cast-downcast`
- `cppcoreguidelines-pro-type-union-access`
- `cppcoreguidelines-pro-type-vararg`

### The "Rule of Zero, Five, or Three"

Resource ownership rules:

- **Rule of Zero:** if your class doesn't manage a resource, don't define any of the special
  member functions. Let the compiler generate. This is the right default.
- **Rule of Five:** if you must manage a resource manually, define all five (destructor, copy
  ctor, copy assign, move ctor, move assign).
- **Rule of Three:** legacy version (no move) — relevant for pre-C++11 only.

The Core Guidelines `cppcoreguidelines-special-member-functions` check enforces this.

### Initialization safety

```cpp
// BAD: indeterminate value
int x;
if (cond) x = 1;
return x;            // UB if !cond

// GOOD
int x{};             // value-initialized to 0
int y = cond ? 1 : 0;   // explicit
```

`cppcoreguidelines-init-variables` enforces "every variable initialized at declaration."

### Concurrency safety section

The Core Guidelines have rules CP.1-CP.42 covering threads, atomics, condition variables, and
shared state. Highlights:

- **CP.20:** Use RAII, never plain `lock()`/`unlock()`.
- **CP.21:** Use `std::lock`/`std::scoped_lock` to acquire multiple mutexes.
- **CP.32:** To share ownership across unrelated threads, use `shared_ptr`.
- **CP.41:** Minimize thread creation and destruction.

`concurrency-mt-unsafe` and `cppcoreguidelines-avoid-non-const-global-variables` cover most
of these.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Adopting GSL piecemeal at internal boundaries | Inconsistent guarantees across modules | Adopt at one layer (e.g. all public APIs) |
| `not_null<T*>` with raw pointer arithmetic afterward | Bypasses the guarantee | Pass through `not_null` consistently or use `span` |
| `Expects`/`Ensures` on a release build | Compiled out — silent failure | Configure GSL to assert always for boundary checks |
| Disabling `pro-*` checks because they "complain a lot" | The checks are the safety profile | Suppress per call site with comment, not globally |
| Using `std::span` from a temp-bound source | Dangling span | Same lifetime issues as raw ptr+size |

## See also

- https://github.com/microsoft/GSL
- https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-profile
