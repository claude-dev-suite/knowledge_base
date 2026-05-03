# Windows Drivers - KMDF - SAL Annotations

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/using-sal-annotations-to-reduce-c-cpp-code-defects
> **Skill**: dev-suite skill `windows/wdf-kmdf` — see SKILL.md for the always-loaded quick reference.

## What this covers

The annotation catalog you'll actually use — IRQL, lock, buffer, in/out, success — and how each one is consumed by the MSVC code analyzer (`/analyze`), Static Driver Verifier, and CodeQL. Read when adding annotations to a new function or when SDV emits a confusing diagnostic about an annotation it cannot prove.

## Deep dive

### Why SAL is non-negotiable in driver code

The C language doesn't carry buffer sizes, lock states, IRQL contracts, or success/failure semantics across function boundaries. SAL adds those as machine-checkable annotations. Three tools consume them:

- **`/analyze`** in MSVC: free, runs at every build; treats most SAL contracts as warnings (`C6XXX`).
- **Static Driver Verifier (SDV)**: runs at WHQL/HCK time; uses a deeper symbolic engine and SAL is the contract language for its rule packs.
- **CodeQL** (Microsoft's `Windows-Driver-Developer-Supplemental-Tools`): runs queries against the AST + dataflow.

A function without SAL is invisible to these tools — it's a hole in the proof.

### IRQL annotations

```c
_IRQL_requires_(PASSIVE_LEVEL)            // caller MUST be at PASSIVE
_IRQL_requires_max_(DISPATCH_LEVEL)       // caller MUST be at <= DISPATCH
_IRQL_requires_min_(APC_LEVEL)            // caller MUST be >= APC
_IRQL_raises_(DISPATCH_LEVEL)             // function raises to DISPATCH on return
_IRQL_saves_                              // _Out_ slot stores old IRQL
_IRQL_restores_                           // _In_ slot restores IRQL
_IRQL_always_function_min_(PASSIVE_LEVEL) // entire function body at >= PASSIVE
```

Example:

```c
_IRQL_requires_max_(DISPATCH_LEVEL)
_When_(return == STATUS_SUCCESS, _IRQL_saves_global_(QueueLock, Irql))
NTSTATUS LockQueue(_In_ WDFSPINLOCK lock, _Out_ PKIRQL Irql);
```

### Buffer annotations

```c
_In_                  // pointer to const, must be non-NULL
_In_opt_              // pointer to const, may be NULL
_Out_                 // function writes through pointer
_Out_opt_
_Inout_
_In_reads_(n)         // _In_ array of n elements
_In_reads_bytes_(n)   // _In_ buffer of n bytes
_Out_writes_(n)
_Out_writes_bytes_to_(cap, written)   // wrote `written` of `cap` capacity
_Field_size_(n)       // struct field is array of n
_Field_size_bytes_(n)
```

Example for an IOCTL helper:

```c
_IRQL_requires_max_(DISPATCH_LEVEL)
_Must_inspect_result_
NTSTATUS CopyToCaller(_Out_writes_bytes_(OutLen) PVOID Out,
                      _In_ size_t OutLen,
                      _In_reads_bytes_(InLen) const VOID* In,
                      _In_ size_t InLen);
```

### Lock annotations

```c
_Acquires_lock_(lock)
_Releases_lock_(lock)
_Requires_lock_held_(lock)
_Requires_lock_not_held_(lock)
_When_(cond, _Acquires_lock_(lock))
_Acquires_exclusive_lock_(lock)
_Acquires_shared_lock_(lock)
```

For functions that conditionally acquire:

```c
_When_(return == STATUS_SUCCESS, _Acquires_lock_(*Lock))
NTSTATUS TryAcquire(_Out_ WDFSPINLOCK* Lock);
```

### Result and success annotations

```c
_Must_inspect_result_                 // caller must use return value
_Success_(return >= 0)                // success when NT_SUCCESS
_When_(return == 0, _Out_)            // OUT param valid only on success
_Check_return_                        // narrower than _Must_inspect_result_
_Result_nullonfailure_
```

### Combining for a typical KMDF helper

```c
_IRQL_requires_max_(DISPATCH_LEVEL)
_Must_inspect_result_
_Success_(return == STATUS_SUCCESS)
NTSTATUS
MyReadDeviceState(
    _In_  WDFDEVICE Device,
    _Out_ PMY_STATE State);
```

`/analyze` will warn if a caller ignores the return, calls at HIGH_LEVEL, or reads `State` on the failure branch.

### Typedef-level annotations

WDF callback types are pre-annotated. Defining a new callback type:

```c
typedef
_IRQL_requires_(PASSIVE_LEVEL)
_Must_inspect_result_
NTSTATUS
EVT_MY_CALLBACK(
    _In_ WDFDEVICE Device,
    _Out_ PULONG Value);

typedef EVT_MY_CALLBACK *PFN_MY_CALLBACK;
```

Implementations should `EVT_MY_CALLBACK MyCallback;` (forward decl) so `/analyze` matches the typedef contract.

### `_Analysis_assume_` and suppressions

When you genuinely know better than the analyzer:

```c
_Analysis_assume_(p != NULL);            // tell /analyze a fact
#pragma warning(suppress: 28167)         // suppress one diagnostic on next line
```

Use sparingly — every assume is a hole. Prefer fixing the annotation.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| `_Out_` parameter not always written | Caller may read uninitialized | Initialize at top of function or use `_When_` |
| `_In_reads_(n)` where `n` is in elements but param is `bytes` | Tool computes wrong overflow | Match the unit suffix (`_bytes_`) |
| Annotating only some helpers | Holes propagate | Annotate every non-trivial function |
| Acquiring lock under `_When_(cond, ...)` but not releasing under same condition | Lock leak diagnostic | Mirror the `_When_` on release |
| Suppressing `28167` (IRQL) instead of fixing | Real bugs hide | Fix the IRQL contract; `_IRQL_raises_` exists |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/sal-2-annotations-for-windows-drivers
- https://learn.microsoft.com/en-us/cpp/code-quality/code-analysis-for-c-cpp-overview
- https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/26165-possible-failure-to-release-lock
