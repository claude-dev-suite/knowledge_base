# C++ - Move Semantics, Rvalue References, and Perfect Forwarding

> **Source**: https://en.cppreference.com/w/cpp/language/move_constructor
> **Skill**: dev-suite skill `cpp` — see SKILL.md for the always-loaded quick reference.

## What this covers

The mechanics behind move semantics: value categories (lvalue/xvalue/prvalue), reference collapsing, perfect forwarding via `std::forward`, when the compiler synthesizes moves, the rule of zero/three/five, and why `noexcept` on moves is load-bearing for STL containers.

## Deep dive

### Value categories

Every C++ expression has a *type* and a *value category*. The categories that matter for moves:

- **lvalue** — has a name, can be on the LHS of `=`. Bind to `T&`.
- **prvalue** — pure rvalue, the "result" of a temporary. Bind to `T&&` (and `const T&`).
- **xvalue** — eXpiring value, the result of `std::move(x)` or a function returning `T&&`. Also binds to `T&&`.

`std::move(x)` does not move anything — it's `static_cast<T&&>(x)`. The actual move happens when an overload taking `T&&` runs.

### Reference collapsing and forwarding references

In a template, `T&&` is a *forwarding reference*, not an rvalue reference. It deduces:

- `T = U&` when called with an lvalue → collapses to `U&`
- `T = U` when called with an rvalue → stays `U&&`

```cpp
template <typename... Args>
auto make_widget(Args&&... args) {
    // Forward args preserving their value category
    return std::make_unique<Widget>(std::forward<Args>(args)...);
}
```

Use `std::forward<T>` only on names that were declared as `T&&` in a template. Use `std::move` everywhere else when transferring ownership from a named lvalue.

### Synthesized moves and the rule of five/zero

The compiler generates copy/move members based on what *isn't* user-declared. Declaring any of `~T`, copy ctor, copy assign, move ctor, or move assign suppresses move generation. This is the "rule of five": if you write one, audit the others.

Better is **rule of zero**: design types from `std::vector`, `std::string`, `std::unique_ptr`, etc. and let the compiler do everything.

```cpp
class HttpRequest {           // Rule of zero — no special members needed
    std::string url_;
    std::vector<Header> headers_;
    std::optional<Body> body_;
public:
    HttpRequest(std::string url) : url_(std::move(url)) {}
    HttpRequest& with_header(std::string k, std::string v) & {
        headers_.push_back({std::move(k), std::move(v)});
        return *this;
    }
};
```

### noexcept moves matter

`std::vector<T>` will move elements during reallocation only if `T`'s move constructor is `noexcept`. Otherwise it copies, silently destroying performance.

```cpp
class Buffer {
    std::unique_ptr<std::byte[]> data_;
    std::size_t size_{};
public:
    Buffer(Buffer&&) noexcept = default;
    Buffer& operator=(Buffer&&) noexcept = default;
};
static_assert(std::is_nothrow_move_constructible_v<Buffer>);
```

### Pass-by-value-then-move

For sink parameters where you'll store the argument:

```cpp
class User {
    std::string name_;
public:
    explicit User(std::string name) : name_(std::move(name)) {}  // optimal for lvalues + rvalues
};
```

This is one overload that handles both lvalue (one copy + one move) and rvalue (two moves) callers.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `return std::move(local)` | Disables NRVO/RVO copy elision | Just `return local;` — the compiler moves automatically |
| `std::move` on a `const T` | Falls back to copy silently | Don't make sink-parameter sources `const` |
| `T&&` parameter in non-template | That's an rvalue reference, won't bind to lvalues | Use `T&` or pass-by-value-then-move |
| Forgetting `noexcept` on move ops | Vector reallocation copies instead | Mark moves and swap `noexcept` |
| Using `std::forward` outside a template | Template magic doesn't apply | Use `std::move` |
| Moving from a member then reading it | Object is in valid-but-unspecified state | Never read a moved-from object except to assign or destroy |

## See also

- https://en.cppreference.com/w/cpp/utility/forward — std::forward reference
- https://en.cppreference.com/w/cpp/language/value_category — formal value-category rules
- https://en.cppreference.com/w/cpp/language/copy_elision — RVO and NRVO mandatory cases
