# Windows Drivers - HID Input Filter - Filter Drivers

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/filter-drivers
> **Skill**: dev-suite skill `windows/hid-input-filter` — see SKILL.md for the always-loaded quick reference.

## What this covers

Filter-driver mechanics in KMDF: what a "filter FDO" actually is, how the device
stack is built, where upper vs lower filters land relative to function drivers, and
how class-wide filters differ from device-specific filters in INF land. The SKILL
shows the simplest INF stanza; this file explains the layering and the lesser-used
options.

## Deep dive

### Anatomy of a device stack

When PnP enumerates a device, it builds a stack of device objects:

```
   Top of stack ─►  Filter 1 (upper class filter)        attached after FDO
                    Filter 2 (upper device filter)
                    Function FDO  ◄─ owns the device     created by function driver
                    Filter 3 (lower device filter)
                    Filter 4 (lower class filter)
   Bottom of stack ► Bus PDO                             owned by parent bus driver
```

Order from registry:

1. PnP loads service from `LowerFilters` (class) bottom-up
2. Then `LowerFilters` (device) bottom-up
3. Then the `Service` (function driver) - creates the FDO
4. Then `UpperFilters` (device) top-down
5. Then `UpperFilters` (class) top-down

Each filter's `EvtDeviceAdd` calls `WdfFdoInitSetFilter(DeviceInit)` *before*
`WdfDeviceCreate`. That single call rewires KMDF so:

- All IRPs default-pass-through to the next-lower object
- The driver does not own power policy
- PnP IDs are inherited from the function driver's FDO

Without `WdfFdoInitSetFilter`, KMDF assumes you are the function driver and will
fail unhandled IRPs — a classic bug.

### Class filter vs device filter

| | Class filter | Device filter |
|---|--------------|---------------|
| INF section | `[ClassInstall32].AddReg` HKR | Per-device hardware key |
| Scope | All devices in the class GUID | Only the matching VID/PID |
| Install | Once per class via co-installer | With each device's INF |
| Update | Re-install or `pnputil -i -a` | Driver update on the device |
| Risk | Affects every keyboard/touch/mouse on the box | Bounded blast radius |

For HID: the class GUID `{745A17A0-74D3-11D0-B6FE-00A0C90F57DA}` is the HID class.
Mouse class is `{4D36E96F-...}`. Choose the narrowest GUID that still sees the
data you care about.

### Surprise removal and orphaned IRPs

Filters must not cancel IRPs they didn't initiate. On `IRP_MN_SURPRISE_REMOVAL`,
KMDF tears down the queue and any pending request you forwarded with
`SEND_AND_FORGET` is owned by the lower stack — you don't need to do anything. But
if you took a reference (e.g., kept the WDFREQUEST in your own context for later
completion) you must release it on the EvtDeviceRemove edge.

```c
// Cancel an inverted-call request held in driver context
VOID MyEvtDeviceSelfManagedIoSuspend(WDFDEVICE Device) {
    PDEVICE_CONTEXT ctx = GetContext(Device);
    WdfSpinLockAcquire(ctx->Lock);
    WDFREQUEST pending = ctx->PendingNotify;
    ctx->PendingNotify = NULL;
    WdfSpinLockRelease(ctx->Lock);
    if (pending) WdfRequestComplete(pending, STATUS_DEVICE_REMOVED);
}
```

### When you need the WDM (PIRP) view

Filters that intercept things KMDF doesn't model directly (e.g. raw
`IRP_MN_QUERY_INTERFACE` to grab a function pointer from a function driver) drop to
WDM with `WdfDeviceWdmDispatchPreprocessedIrp`. This is rare for HID filters but
common for PCI/USB shim drivers.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Forgot `WdfFdoInitSetFilter` | KMDF treats you as function driver, IRPs error out | Always call it in `EvtDriverDeviceAdd` before `WdfDeviceCreate` |
| Filter loads but `EvtIoInternalDeviceControl` never fires | You created a queue without setting it as the default for that IO type | `WDF_IO_QUEUE_CONFIG_INIT_DEFAULT_QUEUE` and bind the right `EvtIo*` |
| Class filter ships globally for one device | Hits every device in the class | Use device-scoped INF or filter inside the callback by `HardwareId` |
| Holding a request across power transitions | Driver hangs on suspend | Complete or dispatch holds on `EvtDeviceSelfManagedIoSuspend` |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/installing-a-kmdf-filter-driver
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/wdffdoinitsetfilter
- https://learn.microsoft.com/en-us/windows-hardware/drivers/install/inf-addreg-directive
