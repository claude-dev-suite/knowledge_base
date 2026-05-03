# Windows Drivers - UMDF v2 - UMDF vs KMDF API Differences

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/comparing-umdf-2-0-functionality-to-kmdf
> **Skill**: dev-suite skill `windows/wdf-umdf` — see SKILL.md for the always-loaded quick reference.

## What this covers

The function-by-function differences between UMDF v2 and KMDF: which `Wdf*` APIs are present in both, which are kernel-only, which are UMDF-only, and the user-mode equivalents for kernel facilities you can't use. Read when porting a KMDF driver to UMDF or when the linker complains about a missing symbol.

## Deep dive

### The shared API surface

Roughly 80% of the `Wdf*` API works identically in both:

```cpp
WdfDriverCreate          WdfDeviceCreate          WdfIoQueueCreate
WdfRequestComplete       WdfRequestRetrieveInputBuffer
WdfRequestRetrieveOutputBuffer                    WdfMemoryCreate
WdfTimerCreate           WdfWorkItemCreate
WdfSpinLockCreate        WdfWaitLockCreate        WdfDpcCreate
WdfFileObjectGetFileObject                        WdfDeviceCreateDeviceInterface
WdfPdoInitAllocate       WdfFdoInitSetFilter
```

A KMDF driver written purely against the shared surface can often be recompiled as UMDF by changing the project type — but the moment you call `ExAllocatePool2`, `KeWaitForSingleObject`, `MmMapIoSpaceEx`, or use SAL `_IRQL_*` annotations, you cross into kernel-only territory.

### Kernel-only APIs (won't link in UMDF)

| Family | Examples | UMDF replacement |
|--------|----------|------------------|
| Pool | `ExAllocatePool2`, `ExFreePoolWithTag` | `new`/`delete`, `malloc`/`free`, `HeapAlloc` |
| Wait | `KeWaitForSingleObject`, `KeSetEvent` | Win32 `WaitForSingleObject`, `SetEvent` |
| MMIO | `MmMapIoSpaceEx`, `READ_REGISTER_*` | Not available; need KMDF companion |
| DMA | `WdfDmaEnablerCreate`, `WdfDmaTransactionInitialize` | Not available |
| Interrupts | `WdfInterruptCreate` (any variant) | Not available; reflector handles inbox stacks |
| Files | `ZwCreateFile`, `ZwReadFile` | Win32 `CreateFileW`, `ReadFile` |
| Registry | `ZwOpenKey`, `RtlQueryRegistryValuesEx` | Win32 registry APIs (`RegOpenKeyEx`...) or `WdfRegistry*` (works in both) |
| Threads | `PsCreateSystemThread` | `std::thread`, `CreateThread` |
| Timers (high-res) | `KeSetTimerEx` with `1ms` resolution | Win32 `CreateWaitableTimerEx(... HIGH_RESOLUTION)` |

### UMDF-only / different in UMDF

| Concept | UMDF specifics |
|---------|----------------|
| `WdfRequestImpersonate` | Lets you impersonate the calling process's token for ACL checks |
| `WdfDeviceWdmDispatchIrp` | Not available in UMDF; kernel-only request manipulation |
| File object policies | `WdfFileObjectWdfCannotUseFsContexts` differs because no kernel `FILE_OBJECT` exists in your address space |
| `IddCx` (Indirect Display Class Extension) | UMDF-only |
| `SensorsCx`, `HidCx` | UMDF-only class extensions |

### Memory model

KMDF: `ExAllocatePool2(POOL_FLAG_NON_PAGED, n, tag)` returns a kernel pointer. The IRQL determines paged vs non-paged. SDV verifies your usage.

UMDF: standard process heap. C++ exceptions, RTTI, smart pointers all work. Use any modern C++ memory tooling:

```cpp
auto buffer = std::make_unique<uint8_t[]>(size);
// or
std::vector<uint8_t> buffer(size);
```

You can still call `WdfMemoryCreate` to tie a buffer to a WDF object lifetime, useful for "free when this device dies".

### Concurrency model

KMDF: IRQL discipline (PASSIVE/APC/DISPATCH/DIRQL). Spinlocks raise IRQL.

UMDF: every callback runs at PASSIVE-equivalent. There's no DPC, no ISR, no spinlock IRQL effect. `WdfSpinLock` exists but is implemented as a SRWLock — semantically a wait lock with no IRQL change. Don't expect it to be cheaper than `WdfWaitLock`.

### Synchronization scope still works

```cpp
WDF_OBJECT_ATTRIBUTES attrs;
WDF_OBJECT_ATTRIBUTES_INIT_CONTEXT_TYPE(&attrs, MY_DEVICE_CONTEXT);
attrs.ExecutionLevel       = WdfExecutionLevelPassive;
attrs.SynchronizationScope = WdfSynchronizationScopeDevice;
WdfDeviceCreate(&DeviceInit, &attrs, &device);
```

This serializes WDF callback dispatch the same as KMDF — useful for rate-limiting concurrent IOCTL handlers.

### File and registry I/O — choose one style

You can mix `WdfRegistry*` (cross-mode) with Win32 `RegOpenKeyEx`. The WDF wrappers give you object-tied lifetime; Win32 gives you the full surface (`RegNotifyChangeKeyValue`, `RegLoadAppKey`...). For new UMDF code, prefer Win32 unless you specifically want WDF cleanup.

For named pipes, sockets, files, web requests — use Win32 directly. There's no equivalent in WDF and there shouldn't be.

### SAL changes

Drop `_IRQL_*` annotations in UMDF code; they're meaningless. Keep buffer/lock annotations (`_In_`, `_Out_writes_(n)`, `_Acquires_lock_(...)`); `/analyze` still uses them.

### Build differences

In Visual Studio, the project type for UMDF v2 is "User Mode Driver, Empty (UMDF V2)". Resulting binary is a `.dll`, not `.sys`. You still ship an `.inf`, `.cat`, and the matching `.pdb`. The `.dll` lives under `%SystemRoot%\System32\drivers\UMDF\` after install — note the UMDF subdirectory.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Linking `ntoskrnl.lib` from a UMDF project | Imports kernel symbols that don't exist in WUDFHost | Use UMDF project template; link against `WUDFx02000.lib` and Win32 libs |
| Using `_IRQL_requires_max_(DISPATCH_LEVEL)` in UMDF code | Ignored, cargo-culted from KMDF | Remove for UMDF code |
| Treating `WDFSPINLOCK` as cheap | It's a SRWLock under the hood | Profile; use `std::mutex` if you don't need WDF lifetime |
| Memory-mapping device registers | No `MmMapIoSpaceEx` in user mode | Use a KMDF companion or an inbox kernel client |
| Calling `WdfRequestImpersonate` then a kernel-only operation | Impersonation is for Win32 secured-resource access | Use it before `CreateFileW` against a secured target |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/wdf-functions-by-version
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/wdf-functions-not-supported-in-umdf
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/wdf-functions-only-supported-in-umdf
