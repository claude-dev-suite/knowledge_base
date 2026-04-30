# SeedSigner Stateless Flow Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/seedsigner`.
> Canonical source: https://github.com/SeedSigner/seedsigner
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/seedsigner/SKILL.md

## Concept

SeedSigner is **stateless**: it boots from SD, has no persistent storage
of seeds, and wipes RAM on power-off. Each session, the user re-enters
the seed via dice rolls, manual word entry, or a SeedQR (a single QR
encoding the BIP-39 mnemonic). After signing, the seed is gone.

The threat model: even if a thief steals the device, there's nothing on
it. Even if the boot SD is imaged, no seed material is there. The seed
exists only in user memory + their backup paper / metal.

The hardware is a Raspberry Pi Zero with a Waveshare 1.3" screen +
camera + four buttons. No SE; security comes from the stateless model
and minimal trusted code base (open-source, audited).

## Walkthrough

### Boot and load seed

1. Insert SeedSigner microSD with the prepared image.
2. Power on Pi Zero (USB cable).
3. Boot ~10s -> main menu.
4. "Seeds" -> "Load a Seed":
   - **From dice rolls**: 99 rolls of d6 -> ~256 bits entropy.
   - **From 24 BIP-39 words**: enter via on-screen keyboard.
   - **From SeedQR**: scan paper/metal QR encoding mnemonic.
5. Optional: add BIP-39 passphrase ("25th word").
6. Seed now in RAM only.

### Verify xpub on coordinator

1. SeedSigner: Seeds -> [my seed] -> "Export Xpub".
2. Pick policy: Single Sig P2WPKH BIP-84, etc.
3. Coordinator (Sparrow) scans animated QR (UR `crypto-account`).
4. Coordinator imports xpub + path + fingerprint.

### Sign PSBT

```
Coordinator                                        SeedSigner
1. Build PSBT
2. Display animated QR (crypto-psbt)
                                                   3. Menu: Sign PSBT
                                                   4. Camera scans
                                                   5. Display tx summary:
                                                      - inputs
                                                      - outputs
                                                      - fee
                                                   6. User confirms each output
                                                   7. Sign in RAM
                                                   8. Display animated QR
                                                      (signed PSBT)
9. Scan back, finalize, broadcast
                                                   10. Power off ->
                                                       seed wiped
```

## Worked example

SeedQR format (Compact):

```
3 14 18 1170 1771 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1
```

Each number is a BIP-39 wordlist index (0-2047), space-separated. A
24-word seed encodes to a CBOR-style QR readable by the camera in one
shot. Compact SeedQR uses 11 bits per word and packs to ~33 bytes.

Generating a SeedQR offline:

```python
from PIL import Image
import qrcode

words = "abandon abandon abandon ... about".split()
indices = [BIP39_WORDS.index(w) for w in words]
data = " ".join(f"{i:04d}" for i in indices)

img = qrcode.make(data, error_correction=qrcode.constants.ERROR_CORRECT_M)
img.save("seedqr.png")   # print + laminate, or etch on metal
```

Use a high-error-correction QR (M or Q) so wear on the printout
doesn't break it.

## Common pitfalls

- **Camera quality**: Pi Zero V1.3 cam is OK; V2.1 is better. Bad lens
  scratches make QR scanning slow.
- **Dice entropy mistakes**: rolling exactly 99 times of d6 gives
  256.46 bits but mishandled rolls (re-rolls "for luck") bias entropy.
  Trust the device's deterministic mapping.
- **Passphrase per session**: if you add a passphrase, you must enter
  it the same way every time, or you're signing for a different
  wallet. Test with small amount first.
- **Lost seed**: stateless = no recovery from device. Backup is
  EVERYTHING.
- **Compromised SD image**: download from official GitHub releases,
  verify SHA256 + GPG signature before flashing. A modified image
  could exfiltrate via WiFi (Pi Zero W has WiFi - SeedSigner image
  disables it; verify firewall rules if paranoid).
- **Multisig setup**: descriptor must be re-imported each session via
  QR before SeedSigner can recognize a multisig PSBT. Add this to
  your routine.

## References

- SeedSigner project: https://github.com/SeedSigner/seedsigner
- SeedQR spec: https://github.com/SeedSigner/seedsigner/blob/dev/docs/seed_qr/README.md
- Sparrow + SeedSigner workflow: https://www.sparrowwallet.com/docs/quick-start.html
