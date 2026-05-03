# Windows Drivers - Indirect Display - IddSampleDriver Tour

> **Source**: https://github.com/microsoft/Windows-driver-samples/tree/main/general/IndirectDisplay
> **Skill**: dev-suite skill `windows/indirect-display` — see SKILL.md for the always-loaded quick reference.

## What this covers

A walkthrough of Microsoft's `IndirectDisplay` C++ sample: file layout, monitor
arrival flow, the dummy frame consumer, and where production drivers diverge from
the sample. The SKILL points at the sample but doesn't tour it; this file is the
guided read.

## Deep dive

### File layout

```
IndirectDisplay/
├── Driver.cpp           Driver/EvtDeviceAdd, registers IddCx callbacks
├── Driver.h             Class definitions: IndirectDeviceContext, etc.
├── IndirectDisplay.inf  UMDF v2 install w/ NativeIdd dispatcher
├── SwapChainProcessor   Class that owns one consumer thread per monitor
├── DirectXDevice        Wraps a D3D11 device for the IDD's adapter LUID
└── EDID                 Static byte array used as the default monitor descriptor
```

Look for these key classes in `Driver.cpp`/`Driver.h`:

- `IndirectDeviceContext` — per-WDF-device state (adapter, monitor list)
- `IndirectMonitorContext` — per-monitor state (assigned swap chain, processor)
- `SwapChainProcessor` — runs the consumer thread, owns a `Microsoft::WRL::Wrappers::Event`
- `DirectXDevice` — wraps a D3D11 device pinned to a specific adapter

### EvtDeviceAdd flow

```cpp
NTSTATUS IndirectDriverEvtDeviceAdd(WDFDRIVER /*Driver*/,
                                    PWDFDEVICE_INIT DeviceInit) {
    WDF_OBJECT_ATTRIBUTES devAttr;
    WDF_OBJECT_ATTRIBUTES_INIT_CONTEXT_TYPE(&devAttr, IndirectDeviceContextWrapper);
    devAttr.EvtCleanupCallback = IndirectDeviceContextCleanup;

    IDDCX_CLIENT_CONFIG iddConfig;
    IDD_CX_CLIENT_CONFIG_INIT(&iddConfig);
    iddConfig.EvtIddCxAdapterInitFinished      = IndirectAdapterInitFinished;
    iddConfig.EvtIddCxAdapterCommitModes       = IndirectAdapterCommitModes;
    iddConfig.EvtIddCxParseMonitorDescription  = IndirectParseMonitorDescription;
    iddConfig.EvtIddCxMonitorGetDefaultModes   = IndirectMonitorGetDefaultModes;
    iddConfig.EvtIddCxMonitorQueryTargetModes  = IndirectMonitorQueryModes;
    iddConfig.EvtIddCxMonitorAssignSwapChain   = IndirectMonitorAssignSwapChain;
    iddConfig.EvtIddCxMonitorUnassignSwapChain = IndirectMonitorUnassignSwapChain;

    NTSTATUS s = IddCxDeviceInitConfig(DeviceInit, &iddConfig);
    if (!NT_SUCCESS(s)) return s;

    WDFDEVICE device;
    s = WdfDeviceCreate(&DeviceInit, &devAttr, &device);
    if (!NT_SUCCESS(s)) return s;

    s = IddCxDeviceInitialize(device);
    auto* ctx = WdfObjectGet_IndirectDeviceContextWrapper(device);
    ctx->pContext = new IndirectDeviceContext(device);
    return s;
}
```

### Adapter init

```cpp
void IndirectDeviceContext::InitAdapter() {
    IDDCX_ADAPTER_CAPS caps = {};
    caps.Size = sizeof(caps);
    caps.MaxMonitorsSupported = 1;          // bump for multi-monitor
    caps.EndPointDiagnostics.Size = sizeof(caps.EndPointDiagnostics);
    caps.EndPointDiagnostics.GammaSupport      = IDDCX_FEATURE_IMPLEMENTATION_NONE;
    caps.EndPointDiagnostics.TransmissionType  = IDDCX_TRANSMISSION_TYPE_WIRED_OTHER;
    caps.EndPointDiagnostics.pEndPointFriendlyName     = L"IddSampleDriver Endpoint";
    caps.EndPointDiagnostics.pEndPointManufacturerName = L"Microsoft";
    caps.EndPointDiagnostics.pEndPointModelName        = L"IddSample";
    // Fill ID strings (firmwareDate, etc.)

    IDARG_IN_ADAPTER_INIT init = {};
    init.WdfDevice = m_WdfDevice;
    init.pCaps     = &caps;
    init.ObjectAttributes = WDF_NO_OBJECT_ATTRIBUTES;
    IDARG_OUT_ADAPTER_INIT out;
    HRESULT hr = IddCxAdapterInitAsync(&init, &out);
    m_Adapter = out.AdapterObject;
}
```

### Monitor arrival in EvtIddCxAdapterInitFinished

```cpp
NTSTATUS IndirectAdapterInitFinished(IDDCX_ADAPTER Adapter,
                                     const IDARG_IN_ADAPTERINITFINISHED* pIn) {
    auto* ctx = WdfObjectGet_IndirectDeviceContextWrapper(
                    IddCxAdapterGetWdfDevice(Adapter))->pContext;
    if (NT_SUCCESS(pIn->AdapterInitStatus)) {
        ctx->FinishInit(0);     // 0 == "primary" virtual monitor index
    }
    return STATUS_SUCCESS;
}

void IndirectDeviceContext::FinishInit(UINT idx) {
    IDDCX_MONITOR_INFO info = {};
    info.Size = sizeof(info);
    info.MonitorType = DISPLAYCONFIG_OUTPUT_TECHNOLOGY_DISPLAYPORT_EXTERNAL;
    info.ConnectorIndex = idx;
    info.MonitorDescription.Size = sizeof(info.MonitorDescription);
    info.MonitorDescription.Type = IDDCX_MONITOR_DESCRIPTION_TYPE_EDID;
    info.MonitorDescription.DataSize = ARRAYSIZE(g_EdidBlock);
    info.MonitorDescription.pData    = g_EdidBlock;
    GUID containerId = MakeContainerId(idx);
    info.MonitorContainerId = containerId;

    IDARG_IN_MONITORCREATE create = { m_Adapter, &info };
    IDARG_OUT_MONITORCREATE created;
    IddCxMonitorCreate(&create, &created);

    IDARG_IN_MONITORARRIVAL arrival = { created.MonitorObject };
    IddCxMonitorArrival(created.MonitorObject, &arrival);
}
```

### Dummy frame consumer

`SwapChainProcessor::Run()` does the minimum needed to satisfy the framework — it
acquires, immediately finishes, releases. No encoding, no transmission.
Production drivers replace the body with NVENC submission and a network sender:

```cpp
void SwapChainProcessor::Run() {
    while (true) {
        WaitForSingleObject(m_NewFrameEvent.Get(), 100);
        if (m_TerminateEvent.is_signaled()) break;
        IDARG_OUT_RELEASEANDACQUIREBUFFER buf{};
        HRESULT hr = IddCxSwapChainReleaseAndAcquireBuffer(m_hSwapChain.Get(), &buf);
        if (hr == E_PENDING) continue;
        if (FAILED(hr)) break;
        // **Production replacement begins here**
        ConsumeFrame(buf);
        // **End production replacement**
        IDARG_IN_FINISHEDPROCESSINGFRAME fin{};
        IddCxSwapChainFinishedProcessingFrame(m_hSwapChain.Get(), &fin);
    }
}
```

### Where to extend the sample

For a real virtual-display product:

1. **Replace the EDID** with one carrying CEA HDR/audio blocks.
2. **Add multi-monitor** by calling `IndirectDeviceContext::FinishInit(N)` in a
   loop, capping at `MaxMonitorsSupported`.
3. **Plug a real consumer** — `ConsumeFrame` opens the shared handle, calls
   `OpenSharedResource1`, hands the texture to NVENC.
4. **Add a control surface** — IOCTL or named pipe so user-mode apps can hot-plug
   monitors (`Indirect*MonitorArrival/Departure`).
5. **Hook up cursor** via `IddCxMonitorSetupHardwareCursor`.
6. **HDR detection** — when DWM gives you `R16G16B16A16_FLOAT`, switch encoder
   to HEVC Main10.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Sample's hard-coded EDID rejected by OS | Manufacturer ID conflicts with real vendor | Generate a unique 3-letter ID (avoid known IDs like SAM/LGD) |
| Sample uses 1 D3D11 device for everything | Doesn't matter at 1 monitor; broken at N | One device per monitor processor |
| Sample doesn't handle re-assign | Mode change crashes | Drain on unassign, fresh state on next assign |
| Copy-paste keeps `MaxMonitorsSupported = 1` for multi-mon | Framework rejects extra arrivals | Bump cap before AdapterInitAsync |

## See also

- https://github.com/microsoft/Windows-driver-samples/blob/main/general/IndirectDisplay/Driver.cpp
- https://learn.microsoft.com/en-us/windows-hardware/drivers/display/install-an-indirect-display-driver
- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/iddcx/nf-iddcx-iddcxmonitorarrival
