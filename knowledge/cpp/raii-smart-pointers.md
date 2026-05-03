# C++ - RAII and Smart Pointers

> **Source**: https://en.cppreference.com/w/cpp/memory
> **Skill**: dev-suite skill `cpp` — see SKILL.md for the always-loaded quick reference.

## What this covers

Deep dive into RAII as the foundation of C++ resource management, the three smart-pointer flavors and their costs, custom deleters with stateful types, and the subtle ownership patterns (factories, polymorphic returns, observer relationships) that come up in real codebases.

## Deep dive

### RAII beyond memory

RAII binds *any* resource lifetime to an object: file descriptors, locks, GL contexts, transactions, signal masks. The pattern is "constructor acquires, destructor releases", and it composes — a `class Connection` holding a `std::unique_ptr<Socket>` and a `std::lock_guard` is exception-safe automatically.

```cpp
class Transaction {
    Database& db_;
    bool committed_{false};
public:
    explicit Transaction(Database& db) : db_(db) { db_.begin(); }
    ~Transaction() { if (!committed_) db_.rollback(); }
    Transaction(const Transaction&) = delete;
    Transaction& operator=(const Transaction&) = delete;
    void commit() { db_.commit(); committed_ = true; }
};
```

### unique_ptr internals

Zero-overhead vs raw pointer when the deleter is empty (EBO). Stateful deleters add storage. For factory functions returning polymorphic types:

```cpp
struct IRenderer { virtual ~IRenderer() = default; virtual void draw() = 0; };
class VulkanRenderer : public IRenderer { /* ... */ };

std::unique_ptr<IRenderer> make_renderer(Backend b) {
    switch (b) {
        case Backend::Vulkan: return std::make_unique<VulkanRenderer>();
        case Backend::Metal:  return std::make_unique<MetalRenderer>();
    }
    return nullptr;
}
```

Always declare a `virtual ~IRenderer() = default;` — otherwise `delete` through the base pointer is UB.

### shared_ptr control block

`make_shared` performs one allocation for object + control block; `shared_ptr<T>(new T)` performs two and prevents the optimization. Atomic refcount means cache-line contention under heavy multi-thread sharing — measure before defaulting to it.

```cpp
class Cache {
    std::unordered_map<Key, std::weak_ptr<Resource>> entries_;
public:
    std::shared_ptr<Resource> get(Key k) {
        if (auto it = entries_.find(k); it != entries_.end())
            if (auto sp = it->second.lock()) return sp;
        auto sp = std::make_shared<Resource>(load(k));
        entries_[k] = sp;
        return sp;
    }
};
```

`weak_ptr::lock()` is the only safe way to use a weak_ptr; `expired()` is racy.

### Custom deleters and C APIs

```cpp
struct FileCloser { void operator()(std::FILE* f) const noexcept { if (f) std::fclose(f); } };
using FilePtr = std::unique_ptr<std::FILE, FileCloser>;

FilePtr open_log(const char* path) {
    return FilePtr{std::fopen(path, "a")};
}
```

Prefer a functor type over a function pointer — function pointers add 8 bytes; functors with no state cost zero.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `shared_ptr` cycles (parent <-> child) | Refcount never reaches 0 | Make the back-edge `weak_ptr` |
| `enable_shared_from_this` called from constructor | No control block exists yet | Use a static factory that creates via `make_shared` then calls init |
| Passing `shared_ptr<T>` by value everywhere | Atomic inc/dec on every call | Pass `T&` or `const T&`; share only at ownership boundaries |
| `unique_ptr<T[]>` with `make_unique<T>(n)` | Wrong overload — single object | Use `std::make_unique<T[]>(n)` |
| Returning `unique_ptr<Derived>` to a slot expecting `unique_ptr<Base>` with a custom deleter | Deleter type mismatch | Use the default deleter or explicitly convert |
| Storing `weak_ptr` then calling `.lock()` in a hot loop | Atomic ops per call | Hoist the locked `shared_ptr` outside the loop |

## See also

- https://en.cppreference.com/w/cpp/memory/unique_ptr — full unique_ptr API including array specialization
- https://en.cppreference.com/w/cpp/memory/shared_ptr/atomic — atomic operations on shared_ptr (C++20)
- https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#R-smart — Core Guidelines on resource management
