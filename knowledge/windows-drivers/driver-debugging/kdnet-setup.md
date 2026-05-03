# Windows Drivers - Driver Debugging - KDNET Setup

> **Source**: https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-a-network-debugging-connection-automatically
> **Skill**: dev-suite skill `windows/driver-debugging` — see SKILL.md for the always-loaded quick reference.

## What this covers

End-to-end KDNET configuration using `kdnet.exe` on the target, the explicit `bcdedit /dbgsettings net` flow when the wizard fails, key generation rules, and the supported NIC list. Use this when SKILL.md's quick `bcdedit` snippet is not enough — usually because the NIC isn't recognised, the key is rejected, or the target boots but never connects.

## Deep dive

### kdnet.exe — the supported wizard

Ship in `C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\kdnet.exe`. Copy this file plus `VerifiedNICList.xml` to the **TARGET** machine, then from an elevated cmd:

```cmd
:: Discover supported NICs and their bus addresses
kdnet.exe

:: Output (truncated):
::   Network debugging is supported on the following NICs:
::   busparams=0.25.0, Intel(R) Ethernet Connection (7) I219-V

:: Configure with HOST IP and a port (>=49152, <=65535)
kdnet.exe 10.0.0.42 50000

:: Output ends with the auto-generated key:
::   Key=2steg.1g6id.3pj4d.3ajke
::   To debug this machine, run the following command on the host:
::     windbg -k net:port=50000,key=2steg.1g6id.3pj4d.3ajke
```

Reboot the target after the wizard prints "OK". The settings live in BCD and persist across reinstalls of WinDbg.

### Manual fallback with bcdedit

If `kdnet.exe` rejects the NIC (no `busparams` row), KDNET still works on most Intel/Realtek/Broadcom adapters via raw `bcdedit`:

```cmd
:: Find the NIC busparams: Device Manager -> Properties -> Details -> "Location information"
:: e.g. "PCI bus 0, device 25, function 0"  ->  busparams=0.25.0

bcdedit /dbgsettings net hostip:10.0.0.42 port:50000 busparams:0.25.0
:: bcdedit prints the key — write it down

bcdedit /debug on
shutdown /r /t 0
```

### Key generation rules

The KDNET key is *not* a password. It is a 256-bit value encoded as four 5-character base32-like groups separated by dots. You can generate a deterministic key with:

```cmd
bcdedit /dbgsettings net hostip:10.0.0.42 port:50000 key:my.own.key.value
```

…but each group **must** match `^[a-z0-9]{1,13}$` and the four groups together must be unique across your debugged machines if you ever attach more than one to the same host port.

### HOST side connection

```cmd
windbg -k net:port=50000,key=2steg.1g6id.3pj4d.3ajke
:: or
kd    -k net:port=50000,key=2steg.1g6id.3pj4d.3ajke
```

WinDbg-Preview: File → Start debugging → Attach to kernel → Net tab → paste port + key.

The host listens on UDP — open inbound port 50000 in Windows Defender Firewall (`netsh advfirewall firewall add rule name="KDNET" dir=in action=allow protocol=udp localport=50000`).

### Verifying the link before reboot

```cmd
:: TARGET
bcdedit /dbgsettings
:: Should print key=, port=, hostip=, dhcp=No, busparams=...

bcdedit /enum
:: The {current} entry should show debug=Yes
```

## Common pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| NIC not in `VerifiedNICList.xml` | KDNET only works on Microsoft-validated MAC controllers | Add a known-good USB-Ethernet (StarTech USB31000NDS works) or use a different machine |
| Wi-Fi as the debug interface | KDNET is wired-only | Use Ethernet, USB-Eth, or a USB 3 debug cable |
| HOST behind NAT / different subnet | KDNET needs UDP reachability both ways | Put both machines on the same L2 segment, or set up a routed VPN |
| Stale key in BCD | After reinstall, target uses old key | Re-run `kdnet.exe` to regenerate |
| Two targets sharing a HOST port | Keys collide on the host listener | Use a unique port per target |
| Hyper-V Gen2 VM target | KDNET needs a synthetic NIC with the right bus path | Use `kdnet.exe` inside the VM — it auto-detects the synthetic NIC |

## See also

- https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/supported-ethernet-nics-for-network-kernel-debugging-in-windows-10
- https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-kernel-mode-debugging-over-a-network-cable-manually
- https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-network-debugging-of-a-virtual-machine-host
