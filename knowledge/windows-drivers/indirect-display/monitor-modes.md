# Windows Drivers - Indirect Display - Monitor Modes

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/iddcx/ns-iddcx-iddcx_monitor_mode
> **Skill**: dev-suite skill `windows/indirect-display` — see SKILL.md for the always-loaded quick reference.

## What this covers

The structure of `IDDCX_MONITOR_MODE`, the difference between *target* modes
(scan-out timing) and *monitor* modes (what the panel claims to support), and how
to declare HDR and wide-color signaling correctly. The SKILL gives a one-line
example; this file covers the field-by-field detail.

## Deep dive

### Anatomy of IDDCX_MONITOR_MODE

```c
typedef struct _IDDCX_MONITOR_MODE {
    UINT                       Size;
    IDDCX_MONITOR_MODE_ORIGIN  Origin;       // Driver / Monitor / Override
    DISPLAYCONFIG_VIDEO_SIGNAL_INFO  MonitorVideoSignalInfo;
} IDDCX_MONITOR_MODE;
```

`MonitorVideoSignalInfo` is the substance — a Windows display-config struct
defined in `wingdi.h`:

```c
typedef struct DISPLAYCONFIG_VIDEO_SIGNAL_INFO {
    UINT64                          pixelRate;
    DISPLAYCONFIG_RATIONAL          hSyncFreq;
    DISPLAYCONFIG_RATIONAL          vSyncFreq;
    DISPLAYCONFIG_2DREGION          activeSize;
    DISPLAYCONFIG_2DREGION          totalSize;
    DISPLAYCONFIG_VIDEO_SIGNAL_INFO_::AdditionalSignalInfo
                                    AdditionalSignalInfo;
    DISPLAYCONFIG_SCANLINE_ORDERING scanLineOrdering;
} DISPLAYCONFIG_VIDEO_SIGNAL_INFO;
```

### Building a 1080p60 mode

```c
inline IDDCX_MONITOR_MODE MakeMode(UINT w, UINT h, UINT vRefresh) {
    IDDCX_MONITOR_MODE m = {};
    m.Size  = sizeof(m);
    m.Origin = IDDCX_MONITOR_MODE_ORIGIN_DRIVER;

    auto& s = m.MonitorVideoSignalInfo;
    s.activeSize       = { w, h };
    s.totalSize        = { w, h };       // virtual: no blanking
    s.AdditionalSignalInfo.vSyncFreqDivider = 1;
    s.AdditionalSignalInfo.videoStandard    = D3DKMDT_VSS_OTHER;
    s.vSyncFreq        = { vRefresh, 1 };
    s.hSyncFreq        = { vRefresh * h, 1 };
    s.pixelRate        = (UINT64)vRefresh * w * h;
    s.scanLineOrdering = DISPLAYCONFIG_SCANLINE_ORDERING_PROGRESSIVE;
    return m;
}
```

Because there is no real panel, `totalSize` can equal `activeSize` (no blanking
porch). `pixelRate` is informational for virtual monitors but DWM uses it for
v-blank pacing — undershoot and frames will be requested faster than declared,
overshoot and frames space out.

### Target vs monitor modes

| Type | Producer | Consumer | API |
|------|----------|----------|-----|
| **Monitor mode** | EDID (parsed by your driver) | DWM uses to pick display resolution | `EvtIddCxParseMonitorDescription`, `MonitorGetDefaultModes` |
| **Target mode** | Your driver (what scan-out timings exist) | OS uses to know what HW timings the link can carry | `EvtIddCxMonitorQueryTargetModes` |

For physical displays these often differ (panel native is 4K60, signal can also
do 1080p120). For virtual displays it's normally fine to declare them identical.

### ColorInfo for HDR / WCG

The `MonitorVideoSignalInfo` does not directly carry color metadata — that lives
in the EDID's CEA HDR Static Metadata block (see `edid.md`). However, some IddCx
versions extend `MonitorVideoSignalInfo` via `AdditionalSignalInfo` with bit flags
to declare wide gamut. To declare 10-bit HDR support you must also handle
`IDDCX_FEATURE_IMPLEMENTATION_HARDWARE` in `IDDCX_ADAPTER_CAPS` and accept
`DXGI_FORMAT_R16G16B16A16_FLOAT` swap chains.

### Variable refresh (VRR)

Modern IddCx (1.10+) supports VRR via `IDDCX_FEATURE_IMPLEMENTATION_HARDWARE`.
Declare a refresh range in `IDDCX_MONITOR_MODE` rather than discrete points:
fill `vSyncFreq` with the *max* and let `IDDCX_MONITOR_DESCRIPTION_TYPE_DXGI`
pass the range through swap-chain metadata. Receivers (e.g. NVENC streaming a
game) benefit from VRR by avoiding tearing without v-sync.

### Default-mode list shape

Most virtual-display drivers ship a fixed list:

```c
static const struct { UINT w, h, hz; } kDefaults[] = {
    {1920, 1080, 60},
    {2560, 1440, 60},
    {3840, 2160, 60},
    {1280,  720, 60},
    {1920, 1080, 30},     // bandwidth-friendly fallback
};
```

Some drivers expose a registry-configurable list so users can add their own.
Keep `PreferredMonitorModeIdx` pointing at the most common (typically 1080p60)
to avoid surprising users with a 4K default that pegs the encoder.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Returning a single mode | Settings UI shows no alternatives | Provide a sensible 4-6 mode list |
| `pixelRate` zero | DWM division by zero in pacing | Always compute `vRefresh * w * h` |
| Mismatch between EDID's preferred timing and your declared modes | OS chooses a mode you can't deliver | Generate EDID from the same mode list |
| `Origin = MONITOR` for a synthesized mode | Confuses the display-mode arbitration | Use `IDDCX_MONITOR_MODE_ORIGIN_DRIVER` for synthesized |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/wingdi/ns-wingdi-displayconfig_video_signal_info
- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/iddcx/ns-iddcx-iddcx_target_mode
- https://learn.microsoft.com/en-us/windows-hardware/drivers/display/displayconfig-api
