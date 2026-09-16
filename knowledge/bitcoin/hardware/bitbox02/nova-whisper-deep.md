# BitBox02 Nova and the Whisper Bluetooth Architecture - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/bitbox02`.
> Canonical source: https://blog.bitbox.swiss/en/whisper-how-the-secure-bluetooth-integration-of-the-bitbox02-nova-works/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/bitbox02/SKILL.md

## Concept

Shift Crypto announced the **BitBox02 Nova** on 20 June 2025 and began
shipping it in summer 2025. It is a second model alongside the original
BitBox02, not a replacement: as of September 2026 both are sold and both
receive firmware updates. The Nova keeps the Multi / Bitcoin-only
firmware split, microSD backup, optional passphrase and anti-klepto.

Three things changed that matter to an integrator:

1. A different secure chip (**Infineon OPTIGA Trust M V3**, EAL6+
   certified) in place of the original's **ATECC608B**.
2. A glass-topped OLED and new case colors.
3. **Bluetooth Low Energy**, added solely so the BitBoxApp can talk to
   the device on iPhone and iPad, where USB access is restricted. The
   BLE integration is branded **Whisper**.

There is no NFC on either model. Connectivity is USB-C on the original,
USB-C plus BLE on the Nova.

## Hardware comparison

Per the bitbox.swiss specification pages as of September 2026:

| Aspect | BitBox02 | BitBox02 Nova |
|--------|----------|---------------|
| Microcontroller | ATSAMD51J20A (120 MHz Cortex-M4F, TRNG) | ATSAMD51J20A (120 MHz Cortex-M4F, TRNG) |
| Secure chip | ATECC608B (TRNG, NIST SP 800-90A/B/C) | OPTIGA Trust M V3, EAL6+, NDA-free |
| Display | 128x64 px white OLED | 128x64 px white OLED with glass top |
| Connectivity | USB-C | USB-C + Bluetooth LE |
| Compatibility | Windows 10+, macOS 12+, Linux, Android | Windows 10+, macOS 10.15+, iOS 16+, iPadOS 16+, Android 9+, Linux (x86_64) |
| Colors | Black | Midnight Black, Polar White, Bitcoin Orange |
| Input | Capacitive touch sensors | Capacitive touch sensors |
| Editions | Multi, Bitcoin-only | Multi, Bitcoin-only |

Both spec pages list the same microcontroller part, so the "increased
storage capacity" the 20 June 2025 announcement cites (more room for
future firmware features) is stated of the device, not of that part.

The dual-chip principle is unchanged in both: the open-source firmware
runs on the MCU and the secure chip is never trusted blindly. The Nova's
chip is **NDA-free**: its documentation can be read without signing a
non-disclosure agreement, which the 20 June 2025 announcement contrasts
with "many other alternatives".

One behavioural change: the original BitBox02 wipes to factory settings
after 10 failed device-password attempts, enforced by the firmware. The
Nova's secure chip enforces that same limit with a **hardware counter**,
so stripping the firmware does not lift it.

## Whisper walkthrough

Whisper's premise is that the Bluetooth stack is not trusted, so every
layer assumes it may be hostile.

### Hardware isolation

BLE runs on a **dedicated DA14531 MCU**, physically separate from the
main MCU. It has no access to the main MCU's flash storage and never
learns wallet secrets. This mirrors the dual-chip argument: the main
firmware stays independent of code it cannot audit at runtime.

### No mutable state

The Bluetooth firmware is open source and reproducible (given an SDK
file from the manufacturer). It is loaded directly into the Bluetooth
chip's **working memory (RAM) on each boot** and is cryptographically
verified by the main MCU, which Shift describes as giving a consistent,
verifiable runtime environment with no mutable state.

### Two encryption layers

1. BLE link layer: **LE Secure Connections** with authenticated
   pairing, the highest setting in the Bluetooth standard. Pairing
   shows a code in an iOS system dialog and on the Nova's screen; the
   user confirms they match.
2. Application layer: the same BitBox end-to-end encryption the
   BitBoxApp already used over USB runs on top. The Bluetooth chip does
   not hold the decryption key, so it cannot read transaction data even
   if the BLE link is broken.

No private keys are transmitted in either direction in any case, so the
link-layer encryption is primarily a **privacy** control.

### Metadata minimisation

- **Reduced signal strength**: while advertising, the antenna is driven
  at the lowest viable power, so discovery and pairing only work in
  close vicinity. Range is extended slightly only after the connection
  is established, for reliability.
- **Random private addresses (RPA)**: the standard BLE RPA mechanism
  obfuscates the MAC address, so unpaired observers see rotating
  addresses and encrypted blobs rather than a trackable identifier.
- **Device names**: an unconfigured Nova advertises a random name each
  power-on (`BitBox A1D6`, `BitBox C2A4`, ...), shown both in the app
  and on the device so the user pairs with the right one. After setup
  the user-chosen device name is advertised, and the user can pick
  something deliberately unremarkable.

### USB first

BLE is only a fallback for Apple mobile devices:

- As soon as the Nova receives data from the BitBoxApp over USB, the
  Bluetooth chip is turned off and any connection terminated.
- If the Nova is plugged in over USB before the BitBoxApp starts, the
  Bluetooth firmware is never started at all.
- Bluetooth can be disabled **persistently** with a firmware setting.
  Re-enabling requires a USB host, so do not disable it if the Nova is
  your only iPhone/iPad signing device.

## Worked example

Handling both secure chips when surfacing device errors. The firmware's
`src/securechip/securechip.h` (BitBoxSwiss/bitbox02-firmware, master as
of September 2026) keeps one error enum with a common range plus one
chip-specific range per backend:

```c
typedef enum {
    // Errors common to any securechip implementation
    SC_ERR_IFS = -1,
    ...
    SC_ERR_MEMORY = -8,

    // Errors specific to the ATECC
    SC_ATECC_ERR_ZONE_UNLOCKED_CONFIG = -100,
    ...
    SC_ATECC_ERR_RESET_KEYS = -106,

    // Errors specific to the Optiga
    SC_OPTIGA_ERR_CREATE = -201,
    ...
    SC_OPTIGA_ERR_UNEXPECTED_LEN = -206,
} securechip_error_t;
```

So a host-side error map keyed only on the `-100..-106` block silently
drops every Nova-specific failure, which lands in `-201..-206`. Map the
common range first, then branch on the chip:

```python
COMMON = {-1: "interface", -2: "invalid args", -3: "config mismatch",
          -6: "incorrect password", -8: "memory"}

def describe(code):
    if code in COMMON:
        return COMMON[code]
    if -106 <= code <= -100:
        return f"ATECC608B secure chip error {code}"   # original BitBox02
    if -206 <= code <= -201:
        return f"OPTIGA Trust M V3 secure chip error {code}"  # Nova
    return f"unknown securechip error {code}"
```

The same header carries `securechip_password_stretch_algo_t`, whose
`SECURECHIP_PASSWORD_STRETCH_ALGO_V0` is commented as the
"Legacy/initial value for BitBox02 and BitBox02 Nova" and whose `V1` is
"Currently used only by Optiga" - the algorithm is a per-device
property, not a per-model constant.

## Host and library support

- **HWI** added BitBox02 Nova support in **3.2.0**, released
  10 February 2026. Earlier HWI releases (3.1.0 and older) enumerate
  only the original BitBox02.
- The firmware repository carries two secure-chip backends side by
  side. `src/securechip/securechip.h` defines a shared error space plus
  `SC_ATECC_ERR_*` and `SC_OPTIGA_ERR_*` ranges, and a password-stretch
  algorithm enum (`SECURECHIP_PASSWORD_STRETCH_ALGO_V0` for both
  devices' initial algorithm, `V1` used only by Optiga). Code that
  surfaces device errors to users should handle both ranges rather than
  assuming ATECC.
- **BitBoxApp** reached the Apple App Store with the 07.2025 "Lucerne"
  release, which brought the app to iOS.

## Common pitfalls

- **Assuming Nova replaced the BitBox02.** Both are current products as
  of September 2026; device enumeration must handle both.
- **Assuming the secure chip part number.** The original is ATECC608B;
  the Nova is OPTIGA Trust M V3. Neither is an ATECC508A, and neither
  device has NFC.
- **Pinning an old HWI.** A toolchain on HWI 3.1.0 or earlier will not
  see a Nova at all.
- **Disabling Bluetooth without a USB host.** The setting is persistent
  and cannot be reversed from an iPhone or iPad.
- **Expecting Bluetooth on the original BitBox02.** It has no BLE
  hardware, so no firmware update can add iPhone/iPad support to it.

## References

- Introducing BitBox02 Nova (20 Jun 2025): https://blog.bitbox.swiss/en/introducing-bitbox02-nova/
- Whisper technical article (3 Jul 2025): https://blog.bitbox.swiss/en/whisper-how-the-secure-bluetooth-integration-of-the-bitbox02-nova-works/
- BitBox02 Nova specifications: https://bitbox.swiss/en/bitbox02/nova/
- BitBox02 specifications: https://bitbox.swiss/en/bitbox02/
- BitBox02 firmware repo: https://github.com/BitBoxSwiss/bitbox02-firmware
- HWI 3.2.0 release notes: https://github.com/bitcoin-core/HWI/releases/tag/3.2.0
