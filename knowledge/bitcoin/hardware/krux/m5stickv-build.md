# Krux M5StickV Build - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/krux`.
> Canonical source: https://selfcustody.github.io/krux/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/krux/SKILL.md

## Concept

Krux runs on Sipeed K210 RISC-V boards: M5StickV (small, ~$50) and
Maix Amigo (larger screen, ~$80). The K210 has no Bluetooth, no WiFi
in M5StickV (Amigo has optional WiFi but Krux disables it), and a
built-in camera. Krux is bitcoin-only, stateless (like SeedSigner), and
flashes its own firmware over the stock factory image.

The flash process replaces vendor firmware (which has WiFi, weather
demos, etc.) with Krux's signed image. After flashing, the device is
purpose-built for airgapped Bitcoin signing.

## Walkthrough

### Flash Krux firmware

```bash
# Download official release
wget https://github.com/selfcustody/krux/releases/download/vX.Y.Z/krux-v.X.Y.Z.zip
unzip krux-v.X.Y.Z.zip

# Verify signature (Krux uses GPG signatures on releases)
gpg --verify krux.kfpkg.sig krux.kfpkg

# Install ktool flasher
pip install krux-installer
# Or use the GUI:
# https://github.com/selfcustody/krux-installer

# Flash via USB
ktool -B m5stickv -b 1500000 krux.kfpkg
# (Or for Maix Amigo: -B amigo)
```

Press the button on M5StickV during boot to enter flash mode if needed.
Flashing takes ~2-3 minutes; do not disconnect.

### Boot and setup

1. Power on M5StickV: hold side button.
2. Krux logo appears -> main menu (small screen, ~135x240 px).
3. "Load Mnemonic" options:
   - **From snapshot**: device generates from camera entropy
     (point at random scene; SHA256 of frame).
   - **From dice rolls**: enter via on-screen.
   - **From 24 words**: type each.
   - **From SeedQR / Compact SeedQR**: scan.
   - **From Tinyseed**: scan a Tinyseed metal plate.
4. Optional BIP-39 passphrase.

### Sign PSBT (with Sparrow as coordinator)

```
Sparrow                                            Krux M5StickV
Right-click tx -> "Show QR"
[animated UR crypto-psbt]
                                                   Menu -> Sign -> PSBT -> QR
                                                   Camera scans
                                                   Shows summary on tiny screen
                                                   Outputs (scrollable)
                                                   Fee
                                                   Approve
                                                   Display animated QR back
"Scan QR" in Sparrow tx dialog
[reads signed PSBT]
Finalize + broadcast
```

## Worked example

Tinyseed integration: Krux can scan a Tinyseed metal plate (small
punched-out word indices) via camera:

```
Menu -> Load Mnemonic -> From Tinyseed -> Scan
Camera reads 24-word indices punched on plate
Krux reconstructs BIP-39 mnemonic in RAM
```

The Tinyseed format encodes BIP-39 words as binary numbers (11 bits per
word) on a 24-row metal plate. Krux's image processing thresholds the
camera frame, finds the punched holes, decodes per-row.

Air-gapped multisig setup with two Krux devices and a Coldcard:

1. Each device boots, loads its seed (from independent backups).
2. Each exports xpub at `m/48'/0'/0'/2'` via QR.
3. Sparrow combines into 2-of-3 wsh descriptor.
4. Sparrow distributes descriptor back to each device.
5. Spend: build PSBT in Sparrow -> QR -> sign on Krux 1 -> QR back ->
   QR -> sign on Coldcard -> SD -> QR back -> finalize.

## Common pitfalls

- **Tiny screen**: M5StickV has 135x240 LCD. Long bech32 addresses wrap
  awkwardly; user must scroll carefully. Verify the **complete** address
  before approving.
- **Camera focus**: M5StickV cam is fixed-focus; scanning at 5-15 cm
  works best. Closer or farther fails.
- **K210 endianness oddities**: some early Krux versions had bugs in
  CBOR/UR decoders; keep firmware up to date.
- **Battery drain**: M5StickV has small ~120 mAh battery. Charge
  fully before long signing sessions.
- **Lost or modified flash**: factory reset is via flashing
  vendor image, then re-flashing Krux. Without official image,
  device is bricked - keep release zip safe.
- **Non-genuine boards**: clones may have different display/cam wiring;
  Krux supports official Sipeed M5StickV and Maix Amigo only.

## References

- Krux docs: https://selfcustody.github.io/krux/
- Krux repo: https://github.com/selfcustody/krux
- M5StickV product: https://docs.m5stack.com/en/core/m5stickv
