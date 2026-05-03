# C++ - Concepts and Constraints (C++20)

> **Source**: https://en.cppreference.com/w/cpp/language/constraints
> **Skill**: dev-suite skill `cpp` — see SKILL.md for the always-loaded quick reference.

## What this covers

Concept design beyond the one-liner examples: subsumption rules, the four ways to constrain a template, building composite concepts, when to prefer `requires`-clause vs trailing `requires`, and standard library concepts in `<concepts>` and `<iterator>`/`<ranges>`.

## Deep dive

### Four ways to attach constraints

```cpp
// 1. Concept used as a type-parameter
template <std::integral T>
T gcd(T a, T b);

// 2. requires-clause after the template parameter list
template <typename T>
    requires std::integral<T>
T gcd(T a, T b);

// 3. requires-clause trailing the function (sees the parameters)
template <typename T>
T gcd(T a, T b) requires std::integral<T>;

// 4. Abbreviated-function-template (parameter-list form)
auto gcd(std::integral auto a, std::integral auto b);   // each `auto` is independent
```

Form 4 is concise but each `auto` is an *independent* template parameter — `auto a, auto b` does not require `a` and `b` to share a type. Use form 1 or 2 when types must match.

### Building requires-expressions

A `requires`-expression is a compile-time predicate over expressions. It has four kinds of requirements:

```cpp
template <typename T>
concept Container = requires(T c, const T cc, typename T::value_type v) {
    typename T::iterator;                                 // type requirement
    { c.size() } -> std::convertible_to<std::size_t>;     // compound requirement with return type
    c.push_back(v);                                       // simple requirement (well-formed)
    requires std::default_initializable<T>;               // nested requirement (boolean)
};
```

`{ expr } -> Concept` checks that `decltype((expr))` satisfies `Concept`. Use it instead of `decltype()` SFINAE.

### Subsumption: which overload wins

When two overloads both match, the *more constrained* one is preferred — but only if its constraints *subsume* the other's. Subsumption is computed on the normalized atomic constraints, not the source text:

```cpp
template <typename T> concept Animal = requires(T a) { a.eat(); };
template <typename T> concept Dog = Animal<T> && requires(T d) { d.bark(); };

void handle(Animal auto&& x);     // less constrained
void handle(Dog auto&& x);        // more constrained — chosen for dogs

handle(Labrador{});  // calls the Dog overload
```

If you write the same conjunction in two places without going through a named concept, subsumption may *not* see them as the same atom. Always factor into named concepts.

### Standard library concepts worth knowing

| Concept | Header | Use case |
|---------|--------|----------|
| `std::integral`, `std::floating_point` | `<concepts>` | Numeric overloads |
| `std::same_as<T, U>` | `<concepts>` | Exact type equality (symmetric) |
| `std::derived_from<Base>` | `<concepts>` | Inheritance constraint |
| `std::convertible_to<To>` | `<concepts>` | Implicit conversion check |
| `std::invocable<F, Args...>` | `<concepts>` | Callable validation |
| `std::ranges::range`, `std::ranges::sized_range` | `<ranges>` | Range-based algorithms |
| `std::input_iterator`, `std::random_access_iterator` | `<iterator>` | Iterator categories |

### Realistic example: a typed event bus

```cpp
template <typename E>
concept Event = requires {
    typename E::id_type;
    requires std::is_trivially_copyable_v<E>;
} && sizeof(E) <= 64;

template <typename H, typename E>
concept Handler = Event<E> && std::invocable<H, const E&>;

class Bus {
    template <Event E, Handler<E> H>
    void subscribe(H handler);
};
```

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Using `std::is_*_v<T>` traits inside `requires` | Works but loses subsumption | Wrap in a named concept built from standard concepts |
| `auto x, auto y` thinking they share a type | Each `auto` is independent | Use named template parameter `<T> ... T x, T y` |
| Concept calls a function that throws on bad types | Concepts must be SFINAE-friendly | Use `requires` and `decltype`, never call code |
| Compound requirement `{ x } -> int` | Wrong syntax — need a concept | `{ x } -> std::same_as<int>` |
| Constraint dispatches don't pick the "more specific" overload | Constraints not in subsumption relation | Refactor to share a named base concept |
| `concept C = T::value` (raw bool member access) | Hard error if missing | Wrap in `requires { requires T::value; }` |

## See also

- https://en.cppreference.com/w/cpp/concepts — standard concepts library reference
- https://en.cppreference.com/w/cpp/language/requires — requires-expression details
- https://en.cppreference.com/w/cpp/iterator — iterator concepts
