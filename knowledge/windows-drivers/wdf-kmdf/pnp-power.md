# Windows Drivers - KMDF - PnP & Power

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/supporting-pnp-and-power-management-in-your-driver
> **Skill**: dev-suite skill `windows/wdf-kmdf` — see SKILL.md for the always-loaded quick reference.

## What this covers

The PnP/Power state machine KMDF runs on your behalf, the exact ordering of `EvtDevicePrepareHardware` -> `EvtDeviceD0Entry` -> queues-running -> `EvtDeviceD0Exit` -> `EvtDeviceReleaseHardware`, idle/wake settings, surprise-removal handling, and the relationship between system power states (S0..S5) and device power states (D0..D3). Read when designing power policy, when devices fail to enter S3, or when surprise-remove leaks resources.

## Deep dive

### The lifecycle, in order

```
EvtDriverDeviceAdd
  WdfDeviceCreate
EvtDevicePrepareHardware       ; PASSIVE; map BARs, allocate DMA enabler
  EvtDeviceD0Entry             ; PASSIVE; bring device online, enable interrupts
    queues start dispatching
    [ runtime ]
    queues stop on power transition
  EvtDeviceD0Exit              ; PASSIVE; quiesce hardware
EvtDeviceReleaseHardware       ; PASSIVE; unmap BARs, free DMA
  WdfObjectDelete(device)      ; framework, on remove
```

Sleep cycles re-enter `EvtDeviceD0Exit` -> `EvtDeviceD0Entry`. `EvtDevicePrepareHardware` runs once per resource set — typically once per PnP start; you can also see it again after a docking change.

### Resource translation in PrepareHardware

```c
NTSTATUS MyEvtDevicePrepareHardware(_In_ WDFDEVICE Device,
                                    _In_ WDFCMRESLIST RawList,
                                    _In_ WDFCMRESLIST TranslatedList)
{
    PMY_DEVICE_CONTEXT ctx = GetDeviceContext(Device);
    ULONG count = WdfCmResourceListGetCount(TranslatedList);

    for (ULONG i = 0; i < count; i++) {
        PCM_PARTIAL_RESOURCE_DESCRIPTOR d =
            WdfCmResourceListGetDescriptor(TranslatedList, i);
        switch (d->Type) {
        case CmResourceTypeMemory:
            ctx->RegsBase = MmMapIoSpaceEx(d->u.Memory.Start,
                                           d->u.Memory.Length,
                                           PAGE_READWRITE | PAGE_NOCACHE);
            ctx->RegsLen = d->u.Memory.Length;
            break;
        case CmResourceTypeInterrupt:
            // captured by WdfInterruptCreate using same translated list
            break;
        }
    }
    return ctx->RegsBase ? STATUS_SUCCESS : STATUS_DEVICE_CONFIGURATION_ERROR;
}
```

Pair with `MmUnmapIoSpace` in `EvtDeviceReleaseHardware`.

### D0Entry / D0Exit — the runtime contract

```c
NTSTATUS MyEvtDeviceD0Entry(_In_ WDFDEVICE Device, _In_ WDF_POWER_DEVICE_STATE PrevState)
{
    PMY_DEVICE_CONTEXT ctx = GetDeviceContext(Device);

    // Restore device state if coming from D3
    if (PrevState >= WdfPowerDeviceD3) {
        WriteHwInit(ctx);
    }
    // Enable interrupts last
    EnableHwInterrupts(ctx);
    return STATUS_SUCCESS;
}

NTSTATUS MyEvtDeviceD0Exit(_In_ WDFDEVICE Device, _In_ WDF_POWER_DEVICE_STATE TargetState)
{
    PMY_DEVICE_CONTEXT ctx = GetDeviceContext(Device);

    // Disable interrupts first to avoid spurious DPCs
    DisableHwInterrupts(ctx);

    if (TargetState >= WdfPowerDeviceD3) {
        SaveHwContext(ctx);     // for restore in D0Entry
    }
    return STATUS_SUCCESS;
}
```

Power-managed queues are stopped by KMDF between `D0Exit` and the next `D0Entry`. Any in-flight request in that interval flows through `EvtIoStop`.

### Idle and wake policy

Set runtime idle so the device drops to D2/D3 when unused, and wake so it can pull the system back to S0:

```c
WDF_DEVICE_POWER_POLICY_IDLE_SETTINGS idle;
WDF_DEVICE_POWER_POLICY_IDLE_SETTINGS_INIT(&idle, IdleCannotWakeFromS0);
idle.IdleTimeout      = 5000;             // ms
idle.UserControlOfIdleSettings = IdleAllowUserControl;
idle.Enabled          = WdfTrue;
WdfDeviceAssignS0IdleSettings(device, &idle);

WDF_DEVICE_POWER_POLICY_WAKE_SETTINGS wake;
WDF_DEVICE_POWER_POLICY_WAKE_SETTINGS_INIT(&wake);
wake.Enabled = WdfTrue;
WdfDeviceAssignSxWakeSettings(device, &wake);
```

`IdleCanWakeFromS0` requires hardware that can self-trigger an interrupt while in low power.

### Surprise removal

```c
NTSTATUS MyEvtDeviceSurpriseRemoval(_In_ WDFDEVICE Device)
{
    // Hardware is GONE. Do not touch registers; reads return all-1s.
    PMY_DEVICE_CONTEXT ctx = GetDeviceContext(Device);
    InterlockedExchange(&ctx->Removed, 1);
    return STATUS_SUCCESS;
}
```

After surprise removal, KMDF still calls `EvtDeviceReleaseHardware`, but you must NOT do MMIO. Unmap and free only.

### S0..S5 vs D0..D3hot/D3cold

System states are kernel-wide; device states are per-device. KMDF translates between them via the bus driver's S->D mapping. You only react to D-states; the framework handles the S-state plumbing. The `_PRX` / `_PR0` / `_PR3` ACPI methods (on ACPI devices) and `D3cold` capability declared in INF determine whether your device can lose Vcc during S3.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Touching MMIO in `EvtDeviceSurpriseRemoval` | Reads return `0xFFFFFFFF`; writes silently dropped or trigger MCE | Mark device as gone; skip register access from then on |
| Enabling interrupts in `PrepareHardware` | Race vs. `D0Entry` setup | Enable last in `D0Entry`, disable first in `D0Exit` |
| Forgetting to save state on D3 | Device restores garbage after sleep | `SaveHwContext` in `D0Exit` when `TargetState >= WdfPowerDeviceD3` |
| Idle timeout too aggressive | Endless D0/D3 cycling thrashes power | Start at several seconds, profile with `powercfg /energy` |
| Holding requests across power transition without `EvtIoStop` | Bugcheck `0x9F` (DRIVER_POWER_STATE_FAILURE) | Implement `EvtIoStop`; complete or `WdfRequestStopAcknowledge` |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/device-power-states
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/supporting-idle-power-down
- https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/handling-pnp-state-changes
