# BitBox02 microSD Backup Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/bitbox02`.
> Canonical source: https://shiftcrypto.support/help/en/4-bitbox02/40-where-can-i-get-help-with-the-bitbox02-microsd-card-backup
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/bitbox02/SKILL.md

## Concept

BitBox02 (Shift Crypto) makes seed backup unusual: instead of asking the
user to write down 24 BIP-39 words on paper, the device writes an
**encrypted** copy of the wallet to a bundled microSD card. The
encryption key is derived from the user's device password
(`PBKDF2(password, salt)`), so possession of the SD alone is useless
without knowing the password. Users CAN also display the 24 BIP-39 words
for paper backup, but the SD path is the canonical Shift workflow.

This trades a paper backup that's vulnerable to "rubber-hose" coercion
for a digital backup that requires both the SD and the password to
restore. Loss of one of the two is recoverable from the other.

## Walkthrough

### Initial setup

1. Connect BitBox02, open BitBoxApp.
2. Choose "Create new wallet".
3. Name the device.
4. Set **device password** (no length minimum but recommended >8 chars
   with mix; this becomes the SD encryption key).
5. Insert microSD.
6. App: "Create backup". Device generates 24-word seed in SE.
7. SE encrypts seed + metadata to SD as `backup_<timestamp>.json`:
   ```json
   {
     "version": 4,
     "ciphertext": "<base64>",
     "salt": "<hex>",
     "device_id": "<hex>",
     "name": "MyBitBox"
   }
   ```
8. App writes file to SD; device verifies by reading back.
9. Optional: app displays "Show recovery words" for paper backup of the
   24 BIP-39 mnemonic.

### Restore flow

1. Wipe BitBox02 (or use new device), set new password.
2. Insert SD with backup file.
3. App: "Restore from microSD".
4. Pick backup from list.
5. Enter device password.
6. SE decrypts ciphertext, loads seed.
7. Wallet is restored.

If the device password is forgotten, the BIP-39 paper backup (if you
created one) is the fallback. Without either, funds are lost.

## Worked example

What's actually inside the encrypted blob (per BitBox02 firmware spec):

```
backup_data = struct {
  seed:               [32]byte,    # raw secp256k1 seed
  language:           u8,          # BIP-39 wordlist index
  birthdate:          u32,          # creation timestamp
  name:               [64]byte,    # wallet name
  metadata:           ...
}

salt = random 32 bytes (per backup)
encryption_key = HKDF(password, salt, "BB02-backup")
ciphertext = AES-256-GCM(encryption_key, backup_data, nonce=salt[:12])
```

The whole blob is signed by the device's attestation key so an attacker
can't substitute a different ciphertext on the SD.

```python
# Inspecting an exported backup file (offline, password required)
import json, base64
from cryptography.hazmat.primitives.kdf.hkdf import HKDF
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

with open("backup_xxx.json") as f:
    blob = json.load(f)
salt = bytes.fromhex(blob["salt"])
ct = base64.b64decode(blob["ciphertext"])
key = HKDF(...).derive(password.encode())
plaintext = AESGCM(key).decrypt(salt[:12], ct, None)
```

(Such offline decryption is for forensic / education; normal use goes
through BitBoxApp + the device.)

## Common pitfalls

- **Lost SD AND lost password**: irrecoverable. Always also write the
  24 BIP-39 words on paper.
- **Backup file modified by another OS**: some macOS versions add
  `.DS_Store` and Spotlight indexes to the SD; harmless, but if the
  filesystem becomes corrupted, the device may fail to read backups.
  Keep the SD clean.
- **Reusing one SD across devices**: each BitBox02 has its own
  attestation key; a backup file is bound to the device that wrote it.
  You can restore on a *different* BitBox by importing as "from
  backup", but the `device_id` field will not match (firmware allows
  this with a warning).
- **Filesystem format**: must be **FAT32**. BitBox02 ships pre-formatted;
  if reformatted to exFAT, device refuses to write.
- **Old firmware backups**: backup format version evolved (v1 -> v4).
  Newer firmware reads older formats but not vice versa.

## References

- Shift Crypto microSD backup help: https://shiftcrypto.support/help/en/4-bitbox02
- BitBox02 firmware repo: https://github.com/digitalbitbox/bitbox02-firmware
- Backup file format spec: search "BackupBackup" in firmware src/
