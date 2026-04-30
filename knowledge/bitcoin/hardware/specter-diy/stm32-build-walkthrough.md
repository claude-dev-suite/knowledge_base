# Specter DIY STM32 Build Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/specter-diy`.
> Canonical source: https://github.com/cryptoadvance/specter-diy
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/specter-diy/SKILL.md

## Concept

Specter DIY is built on the STM32F469-Discovery board (STMicroelectronics
ARM Cortex-M4, 4.3" capacitive touchscreen, MicroSD slot, USB OTG). The
firmware is MicroPython + embit for Bitcoin primitives, flashed via
ST-Link. There's no SE; security comes from physical isolation, the user's
seed handling, and (optionally) a USB-only or QR-only mode.

It targets developer / tinkerer users who want a real STM32 dev
experience and the ability to audit + modify everything. It pairs with
Specter Desktop for the multisig coordinator UX.

## Walkthrough

### Hardware needed

- STM32F469I-DISCO board (~$80 from ST or Mouser).
- USB cable (mini-B for the discovery board's USB-OTG, plus the
  ST-Link CN1).
- Linux/macOS host with `stm32flash` or STM32CubeProgrammer.

### Build firmware

```bash
git clone https://github.com/cryptoadvance/specter-diy
cd specter-diy

# Install build deps
docker build -t specter-diy-builder -f docker/Dockerfile.build .

# Build with Docker (reproducible)
docker run --rm -v "$PWD":/build specter-diy-builder \
    bash -c "cd /build && make -C f469disco -j$(nproc)"

# Output:
ls f469disco/build/
# specter-diy.bin
# specter-diy.dfu
```

### Flash via DFU

```bash
# Put board in DFU mode: hold BOOT0 while pressing RESET
sudo dfu-util -a 0 -i 0 -s 0x08000000 -D specter-diy.bin

# Or via ST-Link (CN1 USB)
st-flash write specter-diy.bin 0x08000000
```

Reset board; touchscreen shows Specter logo. ~30-second boot.

### Initial setup on device

1. Generate or restore 24-word BIP-39 seed via touchscreen.
2. Enter optional passphrase.
3. Pair with Specter Desktop:
   - Connect device USB.
   - Specter Desktop autodetects.
   - Or generate xpub for QR-airgap mode.

### Signing PSBT

USB mode (default):

```
Specter Desktop                                    Specter DIY
Build PSBT
Send via USB to device
                                                   Touchscreen shows tx
                                                   User confirms each output
                                                   Sign internally
                                                   Send signed PSBT back
Finalize + broadcast
```

QR airgap mode (toggle in device settings):

```
Specter Desktop                                    Specter DIY
Display animated QR (UR crypto-psbt)
                                                   "Scan QR" via USB-attached
                                                   webcam OR external scanner
                                                   (the discovery board has no camera)
                                                   ...
```

Note: the F469-Disco has NO camera, so true QR-only airgap requires
either:

- Adding an external camera via expansion header (advanced).
- Using a webcam attached to a separate "QR bridge" computer.
- Using the touchscreen to enter PSBT manually (impractical).

In practice, Specter DIY is mostly used in USB mode; SD-card transfer
is the airgap option.

## Worked example

Reproducible build verification:

```bash
# Compute SHA256 of the official release binary
sha256sum specter-diy.bin

# Compare with the SHA256 in the GitHub release notes
# (or signed by Crypto Advance's GPG key)

# Verify your local build matches:
docker run --rm -v "$PWD":/build specter-diy-builder \
    sha256sum /build/f469disco/build/specter-diy.bin
```

A reproducible build means the SHA from your build matches the public
release - confidence the firmware on the device matches the audited
source.

Multisig with Specter DIY + Coldcard + Trezor (2-of-3):

1. Each device generates xpub at `m/48'/0'/0'/2'`.
2. Specter Desktop combines into wsh sortedmulti descriptor.
3. Specter DIY: import descriptor via QR (or USB if not airgap).
4. Spend: PSBT goes through each in turn; each touchscreen confirms.

## Common pitfalls

- **No SE**: same caveat as SeedSigner / Krux. STM32 flash is encrypted
  but not certified; assume physical attacks possible.
- **Discovery board availability**: STMicroelectronics has had
  shortages on F469-Disco. Specter team has discussed alternative
  boards (F746-Disco, custom hardware).
- **MicroPython memory limits**: very large multisig PSBTs (15+
  cosigners) can OOM. Stick to reasonable thresholds.
- **USB driver issues on Linux**: udev rules needed for `/dev/ttyACM*`
  permissions. See Specter docs.
- **No camera = limited airgap**: if you really want QR-only, prefer
  Passport / SeedSigner / Krux.

## References

- Specter DIY repo: https://github.com/cryptoadvance/specter-diy
- F469-Disco: https://www.st.com/en/evaluation-tools/32f469idiscovery.html
- Specter Desktop: https://github.com/cryptoadvance/specter-desktop
