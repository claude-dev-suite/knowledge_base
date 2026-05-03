# Windows Drivers - UMDF v2 - Sample Tour

> **Source**: https://github.com/microsoft/Windows-driver-samples/tree/main/general/echo/umdf2
> **Skill**: dev-suite skill `windows/wdf-umdf` — see SKILL.md for the always-loaded quick reference.

## What this covers

A guided tour of the most useful UMDF v2 samples in `microsoft/Windows-driver-samples` and the inbox class extension samples (IDD, sensors, HID). Read when starting a new UMDF driver and looking for a known-good template, or when researching a specific class extension.

## Deep dive

### echo / umdf2 — the canonical starter

`general/echo/umdf2/` is the smallest non-trivial UMDF v2 sample.

| File | Purpose |
|------|---------|
| `Driver.cpp` | `DriverEntry`, `EvtDeviceAdd`, WPP init/cleanup |
| `Device.cpp` | Device context, `EvtDevicePrepareHardware`, queue creation |
| `Queue.cpp` | `EvtIoRead`, `EvtIoWrite`, `EvtIoDeviceControl` echoing through a buffer |
| `Public.h` | IOCTL codes shared with the user-mode test app |
| `echoapp/echoapp.cpp` | Win32 console test that opens the device interface |

What it teaches: the minimum surface — DriverEntry, device add, default queue, device interface. C++ is allowed (look for `std::unique_ptr` for context state). Read this first.

### echo / umdf2autosync

`general/echo/umdf2autosync/` is the same as above with `WdfSynchronizationScopeQueue` to demonstrate automatic serialization of queue callbacks. Useful to compare `Sequential + Auto Sync` vs `Parallel + Manual Sync`.

### fx2 (USB) — Cypress FX2 over UMDF

`usb/fx2_driver/` ports the kernel-mode FX2 sample to UMDF over WinUSB. Notable for:

- `UmdfDispatcher = NativeUSB` in the INF
- `WdfUsbTargetDeviceCreateWithParameters` to obtain the WDFUSBDEVICE handle
- Selective suspend handled by the framework via `WDF_DEVICE_POWER_POLICY_IDLE_SETTINGS`
- `WdfUsbTargetPipeFormatRequestForRead` for continuous reader patterns

If you're building a USB peripheral driver in UMDF, copy this layout. Strip the FX2-specific endpoints; keep the request-issuing patterns.

### sensors

`sensors/` contains a family of UMDF drivers using `SensorsCx`:

- `simulator/` — pure software sensor (accelerometer, gyro, light)
- `accelerometer/` — driver for an actual I2C device
- `combo/` — combo sensor reporting multiple types from one device

Each registers `SENSOR_CONTROLLER_CONFIG` with `SensorsCxDeviceInitConfig`. The class extension does most of the heavy lifting — your driver fills in `EvtSensorStart`, `EvtSensorStop`, `EvtSensorGetDataInterval`, etc.

### IDD (Indirect Display Driver)

`general/indirect-display/` is the canonical IDD sample. It demonstrates:

- `IddCxDeviceInitConfig` with adapter capabilities
- `IDDCX_ADAPTER_INIT` registering adapter callbacks (`EvtIddCxAdapterInitFinished`, `EvtIddCxAdapterCommitModes`)
- `IDDCX_MONITOR` swap chains and `IddCxSwapChainSetDevice`
- A separate test client that creates a virtual display

If you're shipping a virtual display driver (USB display, screen mirroring, GPU virtualization client), this sample is the only correct starting point — IDD is UMDF-only.

### HID minidrivers

`hid/` directory:

- `vhidmini2/` — virtual HID minidriver in UMDF (replaces the old `vhidmini` sample)
- `firefly/` — UMDF driver controlling LEDs on a Logitech Firefly mouse via HID feature reports

`vhidmini2` shows the standard `HidCx*` callback registration via `HidCxDeviceInitConfig` and is the right base for any new HID minidriver.

### NetAdapter UMDF

`network/wifi/` contains a WiFi miniport written against the NetAdapterCx user-mode framework. NetAdapterCx supports both UMDF and KMDF; UMDF is fine for over-the-bus NICs (USB WiFi).

### Bluetooth profile drivers

`bluetooth/bthecho/` is a UMDF profile driver demonstrating `BthCx*` callbacks for an L2CAP-based custom protocol. Profile drivers are always UMDF.

### How to build

The samples assume Visual Studio + WDK. From an EWDK or VS Developer Command Prompt:

```cmd
git clone --depth 1 https://github.com/microsoft/Windows-driver-samples
cd Windows-driver-samples\general\echo\umdf2
msbuild kmdfecho.sln /p:Configuration=Debug /p:Platform=x64
```

Output: `x64\Debug\WUDFEcho\WUDFEcho.dll` + `WUDFEcho.inf` + `WUDFEcho.cat`.

To install on a test VM (test signing enabled):

```cmd
pnputil /add-driver WUDFEcho.inf /install
```

Verify in Device Manager that the device shows up under "Sample Device" with no error code. Run `echoapp.exe` to exercise.

### License

MIT-licensed. Copy individual sample folders into your driver repo. Do not vendor the whole repository — it's huge and unrelated samples slow analysis tools.

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Building UMDF samples without the WDK | `WUDFx02000.lib not found` | Install WDK matching VS version |
| Trying to install UMDF v2 driver without test signing | "Driver is not digitally signed" | `bcdedit /set testsigning on` and reboot |
| Using vhidmini (old) sample | UMDF v1 / dead | Use `vhidmini2` |
| Following blog posts predating UMDF v2 | COM-based v1 patterns appear | Cross-check against the docs.microsoft.com timestamp |
| Linking the IDD sample without IddCx headers | IddCx is in WDK 1903+ | Verify WDK version >= 10.0.18362 |

## See also

- https://github.com/microsoft/Windows-driver-samples
- https://learn.microsoft.com/en-us/windows-hardware/drivers/develop/sample-umdf-drivers
- https://learn.microsoft.com/en-us/windows-hardware/drivers/display/indirect-display-driver-model-overview
