# C++ - std::expected and std::format/std::print (C++23)

> **Source**: https://en.cppreference.com/w/cpp/utility/expected
> **Skill**: dev-suite skill `cpp` — see SKILL.md for the always-loaded quick reference.

## What this covers

Composing `std::expected` chains with `transform`/`and_then`/`or_else`, designing error types that work well with it, building format spec strings (fill/align/sign/width/precision/type), writing custom `std::formatter` specializations, and the print-vs-cout performance/locale story.

## Deep dive

### std::expected is a sum type, not a try/catch replacement

`std::expected<T, E>` is a value-based discriminated union of "the result" or "the error". It's monadic: chain operations without nested ifs.

```cpp
#include <expected>
#include <string>
#include <charconv>

enum class ParseError { not_a_number, out_of_range, empty };

std::expected<int, ParseError> parse_int(std::string_view s) {
    if (s.empty()) return std::unexpected(ParseError::empty);
    int v;
    auto [p, ec] = std::from_chars(s.data(), s.data() + s.size(), v);
    if (ec == std::errc::result_out_of_range) return std::unexpected(ParseError::out_of_range);
    if (ec != std::errc{} || p != s.data() + s.size()) return std::unexpected(ParseError::not_a_number);
    return v;
}

std::expected<int, ParseError> double_positive(std::string_view s) {
    return parse_int(s)
        .and_then([](int n) -> std::expected<int, ParseError> {
            if (n < 0) return std::unexpected(ParseError::out_of_range);
            return n * 2;
        })
        .transform([](int n) { return n + 1; });
}
```

- `transform(f)` — apply `f` to the value if present (E unchanged).
- `and_then(f)` — apply `f` returning another `expected` (flat-map / bind).
- `or_else(f)` — recover from an error; `f` takes `E`, returns an `expected`.
- `transform_error(f)` — map the error type.

### Error type design

Pick one of these patterns:

| Pattern | When |
|---------|------|
| `enum class` | Small fixed set, no payload |
| `std::error_code` | Cross-library interop, integrates with `<system_error>` |
| Polymorphic `std::unique_ptr<IError>` | Rich diagnostic info (rare) |
| Tagged struct `{ Code code; std::string message; }` | API boundaries needing context |

Avoid `std::expected<T, std::exception_ptr>` — defeats the purpose.

### std::format specifications

```cpp
#include <format>

std::format("{:>10}", "hi");           // "        hi"
std::format("{:*^10}", "hi");          // "****hi****"
std::format("{:+.3f}", 3.14159);       // "+3.142"
std::format("{:#06x}", 255);           // "0x00ff"
std::format("{:%Y-%m-%d}", today);     // chrono types: ISO date
std::format("{1} {0} {1}", "a", "b");  // positional: "b a b"
```

The mini-language: `[fill][align][sign][#][0][width][.precision][type]`.

### Custom formatter

```cpp
struct Money { long long cents; std::string currency; };

template <>
struct std::formatter<Money> {
    bool with_symbol = false;
    constexpr auto parse(std::format_parse_context& ctx) {
        auto it = ctx.begin();
        if (it != ctx.end() && *it == '$') { with_symbol = true; ++it; }
        return it;
    }
    auto format(Money const& m, std::format_context& ctx) const {
        return with_symbol
            ? std::format_to(ctx.out(), "{} {}.{:02}", m.currency, m.cents / 100, m.cents % 100)
            : std::format_to(ctx.out(), "{}.{:02}",   m.cents / 100, m.cents % 100);
    }
};

std::print("Total: {:$}\n", Money{1599, "USD"});  // "Total: USD 15.99"
```

### std::print vs std::cout

`std::print` (C++23) writes directly to the underlying stream FILE*, bypassing iostream sync overhead. It is locale-independent by default — explicit `std::print(std::locale{}, ...)` opts in. Output is buffered and flushed on `\n` only when the stream is line-buffered (i.e. typically only on terminals).

```cpp
#include <print>
std::println("user={} latency={:.2f}ms", name, ms);   // adds \n, flushes if needed
std::print(stderr, "ERROR: {}\n", err);
```

Compile-time format checking: `std::format("{:d}", "hello")` is a compile error; `std::vformat` is the runtime-string escape hatch.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `*expected` without checking `has_value()` | UB if error | `if (e) ... else ...` or `value_or` |
| `std::expected<void, E>` forgotten | Common pattern, easy to miss the `<void, ...>` form | Use it; `e->void_member()` works as expected |
| `std::format` with runtime string literal | Compile error | Use `std::vformat(fmt_string, std::make_format_args(...))` |
| Custom formatter without `parse()` | Inherits from default — easy mistake to skip | Always implement both `parse` and `format` |
| Calling `value()` and treating exception as control flow | Defeats the value-based design | Use `and_then`/`or_else` chains |
| Logging `std::expected<T, E>` directly | Not formattable by default | Add a custom formatter or unwrap before logging |

## See also

- https://en.cppreference.com/w/cpp/utility/format — full format specification
- https://en.cppreference.com/w/cpp/io/print — std::print/std::println
- https://en.cppreference.com/w/cpp/utility/expected/expected — full expected API
