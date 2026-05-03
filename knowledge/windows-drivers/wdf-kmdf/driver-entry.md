# Windows Drivers - KMDF - DriverEntry & EvtDriverDeviceAdd

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/driverentry-for-kmdf-drivers
> **Skill**: dev-suite skill `windows/wdf-kmdf` — see SKILL.md for the always-loaded quick reference.

## What this covers

The exact contract between the kernel loader and your `DriverEntry`, every field on `WDF_DRIVER_CONFIG`, registration flags such as `WdfDriverInitNoDispatchOverride`, the unload path, and the differences between class drivers and per-instance drivers. Use this when standing up a new driver project or when `DriverEntry` returns success but your callbacks never fire.

## Deep dive

### What the I/O Manager guarantees on entry

`DriverEntry` is invoked at `PASSIVE_LEVEL` in the `System` process. `DriverObject->DriverExtension` is already allocated. `RegistryPath` points at `HKLM\SYSTEM\CurrentControlSet\Services\<YourService>` and is **only valid for the duration of the call** — copy the `UNICODE_STRING` if you need it later (KMDF stashes a copy for you when you call `WdfDriverCreate`).

You may NOT touch `DriverObject->MajorFunction` if `WdfDriverInitNoDispatchOverride` is unset (the default), because KMDF will overwrite the table. If you need to coexist with a hand-written WDM dispatch slot, set the flag and call `WdfDriverMiniportUnload` from your manual unload routine.

### Filling out WDF_DRIVER_CONFIG

```c
NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject,
                     _In_ PUNICODE_STRING RegistryPath)
{
    WDF_DRIVER_CONFIG    config;
    WDF_OBJECT_ATTRIBUTES attrs;
    NTSTATUS              status;

    // Per-driver context (rare but useful for global state)
    WDF_OBJECT_ATTRIBUTES_INIT_CONTEXT_TYPE(&attrs, MY_DRIVER_CONTEXT);
    attrs.EvtCleanupCallback = MyDriverCleanup;   // PASSIVE; before unload

    WDF_DRIVER_CONFIG_INIT(&config, MyEvtDeviceAdd);
    config.DriverPoolTag      = 'rvDM';            // shows up in !poolused
    config.EvtDriverUnload    = MyEvtDriverUnload; // optional, defaults are fine
    config.DriverInitFlags    = 0;                 // or WdfDriverInitNonPnpDriver

    // Tell WPP to initialize before the framework starts emitting traces
    WPP_INIT_TRACING(DriverObject, RegistryPath);

    status = WdfDriverCreate(DriverObject,
                             RegistryPath,
                             &attrs,
                             &config,
                             WDF_NO_HANDLE);

    if (!NT_SUCCESS(status)) {
        WPP_CLEANUP(DriverObject);
        return status;
    }
    return STATUS_SUCCESS;
}
```

### EvtDriverDeviceAdd — one call per device instance

The PnP manager invokes this every time a matching hardware ID enumerates. You receive a `PWDFDEVICE_INIT` you can mutate before `WdfDeviceCreate`. After `WdfDeviceCreate` succeeds, the `PWDFDEVICE_INIT` is consumed — do not touch it.

```c
NTSTATUS MyEvtDeviceAdd(_In_ WDFDRIVER Driver, _Inout_ PWDFDEVICE_INIT DeviceInit)
{
    UNREFERENCED_PARAMETER(Driver);

    // Tell PnP to use buffered I/O for read/write by default
    WdfDeviceInitSetIoType(DeviceInit, WdfDeviceIoBuffered);

    // File create/close/cleanup callbacks
    WDF_FILEOBJECT_CONFIG fc;
    WDF_FILEOBJECT_CONFIG_INIT(&fc, MyEvtFileCreate, MyEvtFileClose, WDF_NO_EVENT_CALLBACK);
    WdfDeviceInitSetFileObjectConfig(DeviceInit, &fc, WDF_NO_OBJECT_ATTRIBUTES);

    // PnP/Power callbacks
    WDF_PNPPOWER_EVENT_CALLBACKS pp;
    WDF_PNPPOWER_EVENT_CALLBACKS_INIT(&pp);
    pp.EvtDevicePrepareHardware = MyEvtDevicePrepareHardware;
    pp.EvtDeviceReleaseHardware = MyEvtDeviceReleaseHardware;
    pp.EvtDeviceD0Entry         = MyEvtDeviceD0Entry;
    pp.EvtDeviceD0Exit          = MyEvtDeviceD0Exit;
    WdfDeviceInitSetPnpPowerEventCallbacks(DeviceInit, &pp);

    WDF_OBJECT_ATTRIBUTES attrs;
    WDF_OBJECT_ATTRIBUTES_INIT_CONTEXT_TYPE(&attrs, MY_DEVICE_CONTEXT);

    WDFDEVICE device;
    NTSTATUS status = WdfDeviceCreate(&DeviceInit, &attrs, &device);
    if (!NT_SUCCESS(status)) return status;

    // Create a symbolic link so user mode can CreateFile("\\\\.\\MyDev")
    DECLARE_CONST_UNICODE_STRING(symlink, L"\\DosDevices\\MyDev");
    status = WdfDeviceCreateSymbolicLink(device, &symlink);
    if (!NT_SUCCESS(status)) return status;

    // Default queue
    WDF_IO_QUEUE_CONFIG q;
    WDF_IO_QUEUE_CONFIG_INIT_DEFAULT_QUEUE(&q, WdfIoQueueDispatchParallel);
    q.EvtIoDeviceControl = MyEvtIoDeviceControl;
    return WdfIoQueueCreate(device, &q, WDF_NO_OBJECT_ATTRIBUTES, NULL);
}
```

### Non-PnP drivers (NT-style services)

Set `WdfDriverInitNonPnpDriver` in `config.DriverInitFlags` and provide `config.EvtDriverUnload`. There is no `EvtDriverDeviceAdd`; you call `WdfControlDeviceInitAllocate` from `DriverEntry` and create a control device manually.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Returning success but `WdfDriverCreate` failed | Loader thinks driver is loaded; service stays running but does nothing | Always propagate the `NTSTATUS` and `WPP_CLEANUP` |
| Capturing `RegistryPath` directly | Buffer is reused after return | `RtlDuplicateUnicodeString` or rely on the framework's copy via `WdfDriverGetRegistryPath` |
| Forgetting `WPP_INIT_TRACING` before first trace | First `TraceEvents` is dropped | Init WPP as the very first action |
| Setting both `WdfFdoInitSetFilter` and creating queues with `EvtIoRead` | Filter swallows reads | Filters should forward unhandled requests; only intercept what you care about |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/creating-a-framework-driver-object
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/writing-an-evtdriverdeviceadd-event-callback-function
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/wdf-driver-config-init
