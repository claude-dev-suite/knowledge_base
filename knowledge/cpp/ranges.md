# C++ - Ranges, Views, and Algorithms (C++20/23)

> **Source**: https://en.cppreference.com/w/cpp/ranges
> **Skill**: dev-suite skill `cpp` — see SKILL.md for the always-loaded quick reference.

## What this covers

How range views compose lazily, the difference between view adaptors and range algorithms, projections, the `std::ranges::to` materializer in C++23, custom views via `view_interface`, and the cost model that determines whether a pipeline allocates or stays zero-overhead.

## Deep dive

### Views are lazy and cheap to copy

A `view` is a range with O(1) copy/move and reference semantics over its underlying data. Pipelines build a stateless adaptor object that evaluates lazily during iteration:

```cpp
#include <ranges>
#include <vector>

auto pipeline =
    std::views::iota(1)
    | std::views::transform([](int n){ return n * n; })
    | std::views::take_while([](int n){ return n < 1000; })
    | std::views::filter([](int n){ return n % 2 == 0; });

for (int x : pipeline) std::print("{} ", x);
```

No allocation, no intermediate vector. Each element flows through the chain on demand.

### Projections: algorithms that index by member

`std::ranges` algorithms accept a *projection* — a callable applied to each element before the predicate runs. This replaces the `[](auto& a, auto& b){ return a.id < b.id; }` boilerplate:

```cpp
struct Order { int id; double total; std::string customer; };
std::vector<Order> orders = load();

std::ranges::sort(orders, std::less{}, &Order::total);
auto big = std::ranges::find_if(orders, [](double t){ return t > 1000.0; }, &Order::total);
```

Member-pointers and member-function-pointers are valid projections via `std::invoke` semantics.

### Materializing with std::ranges::to (C++23)

```cpp
auto top_customers = orders
    | std::views::filter([](const Order& o){ return o.total > 100.0; })
    | std::views::transform(&Order::customer)
    | std::ranges::to<std::vector<std::string>>();

// Works for any container that can be constructed from a range
auto unique_ids = orders
    | std::views::transform(&Order::id)
    | std::ranges::to<std::set<int>>();
```

Pre-C++23 substitute: `std::vector<T> v(rng.begin(), rng.end());` — but only works for common ranges (where `begin()` and `end()` have the same type). Many views are not common; wrap with `std::views::common` if needed.

### Useful view adaptors (selected)

| View | Effect |
|------|--------|
| `views::zip(a, b)` (C++23) | Pairwise iteration |
| `views::enumerate(rng)` (C++23) | `(index, value)` pairs |
| `views::chunk(rng, n)` (C++23) | Non-overlapping windows of size n |
| `views::adjacent<N>(rng)` (C++23) | Sliding windows of fixed size |
| `views::join` | Flatten range-of-ranges |
| `views::split(rng, delim)` | Split by delimiter |
| `views::reverse` | O(1) reverse view |
| `views::stride(rng, n)` (C++23) | Every nth element |

### Iterator-invalidation rules still apply

Views hold references to the underlying range. Modifying the source mid-iteration is the same UB as with iterators:

```cpp
std::vector<int> v{1,2,3,4,5};
for (int x : v | std::views::filter([](int n){ return n > 2; })) {
    if (x == 4) v.push_back(99);   // UB — invalidates the filter's cached iterator
}
```

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Passing a view by value, expecting deep copy | Views are reference-semantic | Materialize with `ranges::to` first |
| Storing a view of a temporary range | Dangling reference | Move/copy the source into a local first |
| `views::filter` evaluating predicate twice per element | Required by the standard for input iterators | Cache work inside transform if expensive |
| Range-based `for` over `views::filter | views::transform` recomputing | Combine into one transform when work is shared |
| Calling `std::sort` on a view | Most views aren't random-access or aren't writable | Materialize, then sort |
| `auto v = vec | views::filter(...)` outliving `vec` | View dangles | Use `vec` inside the loop or extend lifetime |

## See also

- https://en.cppreference.com/w/cpp/ranges/view — view concept and `view_interface` for custom views
- https://en.cppreference.com/w/cpp/algorithm/ranges — full ranges algorithms list
- https://en.cppreference.com/w/cpp/ranges/to — std::ranges::to reference
