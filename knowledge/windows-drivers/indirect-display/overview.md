# Windows Drivers - Indirect Display - Overview

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/display/indirect-display-driver-model-overview
> **Skill**: dev-suite skill `windows/indirect-display` — see SKILL.md for the always-loaded quick reference.

## What this covers

The IDD model in context: where it sits relative to WDDM and DXGK, what runs in
kernel vs user mode, and the OS components that talk to it. The SKILL gives a
practical "what fits" matrix; this file explains the architectural reasons.

## Deep dive

### IDD vs WDDM

| Aspect | WDDM (real GPU) | IDD (virtual / streaming) |
|--------|------------------|----------------------------|
| Kernel surface | DXGK miniport | None — IddCx KMDF shim only |
| User-mode component | UMD (D3D usermode driver, e.g. nvwgf2um.dll) | UMDF v2 IDD driver in `WUDFHost.exe` |
| Hosts processes | DWM, all D3D apps | DWM only (apps stay on their primary GPU) |
| Memory | VRAM owned by GPU | System RAM staging + DXGI shared handles |
| Display path | Real DDI → CRTC → physical port | Virtual swap-chain delivered to driver |
| Hot-plug latency | µs | ms (PnP + IddCx monitor arrival) |
| Supports adapter as primary | YES | NO (additive only) |
| Code model | C++ kernel-mode, IRQL-aware | C++ user-mode, COM-style RAII |

The IDD framework is essentially a "virtual GPU" driver that doesn't render — it
just receives the composited frame DWM produces for the virtual monitor. The
*real* GPU on the system still does the rendering and composition.

### Render path

```
   App (WPF/UWP/Win32)
     ↓ D3D
   Real GPU (composes app windows into per-window surfaces)
     ↓ DXGI present
   DWM (composites windows into per-monitor swap-chains)
     ↓ For each monitor:
       - Real monitor: WDDM scanout
       - Virtual monitor: framework hands swap-chain to IDD driver
         ↓
       Your IDD's processing thread: encode/transmit
```

DWM does *not* know your driver is virtual — it sees a display adapter advertising
monitors. The framework abstracts the swap-chain plumbing so your driver only
deals with `IddCxSwapChain*` calls.

### Where IddCx fits

`IddCx.lib` is a UMDF v2 helper library (statically linked into your driver) plus
`IddCx.dll` runtime that lives in `WUDFHost.exe`. Your DLL implements callbacks;
IddCx implements the WDDM emulation that DXGK requires. The framework also brokers
the GPU-to-GPU surface transfer when DWM renders on GPU A but the swap-chain must
be consumed by your driver on GPU B (or no GPU at all) — this is sometimes called
*cross-adapter resource sharing*.

### Process and thread model

Your IDD DLL runs in `WUDFHost.exe`. Each call from the framework arrives on a
framework thread; long work (frame encoding, network I/O) **must not** run inline
on these. Pattern: per monitor, one consumer thread that owns:

- A D3D11 device on the same adapter as the swap-chain surfaces
- An NVENC / QSV / AMF encoder session
- A staging texture pool
- A network sender (separate thread or async I/O)

### Privilege and signing

IDD drivers must be signed with a WHQL-attested or EV-signed cert. The
`UmdfDispatcher = NativeIdd` line in the INF is privileged — Microsoft requires
attestation signing at minimum to load it on retail Windows.

### Versioning

| OS | IddCx version | Notable additions |
|----|--------------|-------------------|
| Win10 1709 | 1.0 | Initial IDD support |
| Win10 1903 | 1.4 | Hardware cursor, monitor query mode |
| Win10 2004 | 1.5 | HDR / wide color gamut |
| Win11 21H2 | 1.8 | Adapter-mode change without recreate |
| Win11 22H2 | 1.9 | More flexible cursor formats |
| Win11 24H2 | 1.10 | VRR support |

Declare the minimum with `IDDCX_VERSION_MAJOR` / `_MINOR` on
`IddCxDeviceInitializeAsync`. Fall back gracefully on older OSes.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Trying to make IDD the primary display | Framework explicitly forbids it | Add as additional display; user can drag windows there |
| Running encoder on the framework callback thread | Stalls IddCx, DWM throttles down | Spawn a dedicated processor thread per monitor |
| Sharing one D3D11 device across all monitors | Submission queue contention, poor multi-monitor scaling | One device + immediate context per monitor thread |
| Installing on Win10 < 1709 | IddCx not present | Set INF `MinNTVersion` to 10.0.16299 or newer |
| Skipping signing during dev | Driver fails to load with code 52 | Use test-mode signing + `bcdedit /set testsigning on` for dev VMs |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/display/indirect-display-driver-model-architecture
- https://learn.microsoft.com/en-us/windows-hardware/drivers/develop/installation-and-deployment-of-umdf
- https://github.com/microsoft/Windows-driver-samples/tree/main/general/IndirectDisplay
