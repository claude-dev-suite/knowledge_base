# C++ - Coroutines (C++20) and std::generator (C++23)

> **Source**: https://en.cppreference.com/w/cpp/language/coroutines
> **Skill**: dev-suite skill `cpp` — see SKILL.md for the always-loaded quick reference.

## What this covers

The mental model behind C++20 coroutines (a function becomes a state machine), the promise type / awaiter protocol you must understand to debug them, when to reach for `std::generator` vs a third-party `task<T>`, and the pitfalls around dangling references in the coroutine frame.

## Deep dive

### The coroutine machinery

A function that contains `co_await`, `co_yield`, or `co_return` is a coroutine. The compiler:

1. Allocates a *coroutine frame* (heap by default) holding parameters, locals, and a `promise_type`.
2. Returns whatever `promise_type::get_return_object()` constructs to the caller.
3. Runs the body up to the first suspension point.
4. Resumes via `coroutine_handle::resume()`.

The return type's `promise_type` defines the protocol. This is why every coroutine library defines a `task<T>`, `lazy<T>`, or `generator<T>` — they each provide a `promise_type` with different policies.

### std::generator (C++23) for synchronous sequences

```cpp
#include <generator>
#include <ranges>

std::generator<std::pair<std::filesystem::path, std::uintmax_t>>
walk(std::filesystem::path root) {
    for (auto const& e : std::filesystem::recursive_directory_iterator{root}) {
        if (e.is_regular_file())
            co_yield {e.path(), e.file_size()};
    }
}

for (auto const& [p, sz] : walk("/tmp") | std::views::take(100)) {
    std::print("{:>10} {}\n", sz, p.string());
}
```

`std::generator<T>` is a *range*; it composes with views. It is single-pass (input range).

### Awaiters: the three-method protocol

`co_await expr` requires `expr` to model the *Awaitable* concept. After transformation, the awaiter must provide:

```cpp
struct Awaiter {
    bool await_ready() noexcept;                            // true => skip suspend
    void await_suspend(std::coroutine_handle<> h) noexcept; // schedule resumption
    T    await_resume();                                    // result of co_await
};
```

`await_suspend` can return `void`, `bool`, or another `coroutine_handle<>` — the last form enables *symmetric transfer*, eliminating stack growth in await chains.

### A minimal task<T> sketch (what libraries provide)

```cpp
template <typename T>
struct task {
    struct promise_type {
        T value_;
        std::coroutine_handle<> continuation_{};
        task get_return_object() {
            return task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }
        auto final_suspend() noexcept {
            struct Final {
                bool await_ready() noexcept { return false; }
                std::coroutine_handle<> await_suspend(
                    std::coroutine_handle<promise_type> h) noexcept {
                    return h.promise().continuation_;       // symmetric transfer
                }
                void await_resume() noexcept {}
            };
            return Final{};
        }
        void return_value(T v) { value_ = std::move(v); }
        void unhandled_exception() { std::terminate(); }
    };
    /* operator co_await, destructor that destroys the handle, ... */
};
```

In practice use `cppcoro`, `folly::coro`, `unifex`, or `stdexec` — writing this yourself is a tarpit.

### Lifetime: the frame owns nothing it didn't allocate

```cpp
std::generator<int> bad(std::vector<int> const& v) {  // reference parameter
    for (int x : v) co_yield x;                       // dangling if caller passes a temporary
}
auto g = bad(std::vector{1,2,3});                     // UB on first iteration
```

Pass by value into coroutines (the parameter lives in the frame), or document the lifetime contract loudly.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Passing references / views into a coroutine | Frame outlives caller's temporaries | Pass by value, or guarantee lifetime |
| Throwing in `unhandled_exception` body | Calls `std::terminate` if you re-throw incorrectly | Store as `std::exception_ptr`, rethrow on resumption |
| Forgetting `co_return` in non-void coroutine | Falls off end → UB | Always end with explicit `co_return` |
| `await_suspend` doing slow work | Blocks the resuming thread | Only schedule, never run |
| Recursive `co_await` chains without symmetric transfer | Stack overflow | Return `coroutine_handle` from `await_suspend` |
| Using `std::generator` across threads | Single-pass + non-thread-safe handle | Use a queue/channel between threads |

## See also

- https://en.cppreference.com/w/cpp/coroutine — coroutine library facilities
- https://en.cppreference.com/w/cpp/utility/generator — std::generator reference
- https://lewissbaker.github.io/ — Lewis Baker's coroutine theory series (canonical learning resource)
