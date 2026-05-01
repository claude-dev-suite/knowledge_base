# embit on MicroPython - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/embit`.
> Canonical source: https://github.com/diybitcoinhardware/embit
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/embit/SKILL.md

## Concept

`embit` is a Python Bitcoin library written from the ground up for
embedded use. It runs unmodified on CPython, and -- crucially -- on
MicroPython, where it powers DIY hardware signers like SeedSigner,
Krux, and Specter DIY. Compared to `python-bitcoinlib` it ships
without `bitcoin.rpc` (no JSON), uses byte-oriented APIs that minimise
allocations, and lays out modules so that unused code (Liquid, Schnorr,
etc.) can be excluded from a frozen-bytecode firmware build.

Standard wallet flow on a MicroPython HW signer: receive a PSBT (over
QR / SD / USB serial), parse with `embit.psbt.PSBT`, derive keys with
`embit.bip32.HDKey`, sign each input, write the signed PSBT back out.

## API walkthrough

```python
from embit import bip32, bip39, networks
from embit.descriptor import Descriptor
from embit.psbt import PSBT
from embit.script import p2wpkh

NET = networks.NETWORKS["main"]

mnemonic = "abandon abandon abandon abandon abandon abandon " \
           "abandon abandon abandon abandon abandon about"
seed = bip39.mnemonic_to_seed(mnemonic)
root = bip32.HDKey.from_seed(seed, version=NET["xprv"])

acc = root.derive("m/84h/0h/0h")
xpub = acc.to_public()
print("fingerprint:", root.child(0).fingerprint.hex())
print("xpub:", xpub.to_base58())

# Address at /0/5
node = acc.derive("0/5")
print("address:", p2wpkh(node).address(NET))
```

## Worked example: sign a PSBT on a MicroPython device

```python
from embit.psbt import PSBT
from embit import bip32, bip39, networks

NET = networks.NETWORKS["main"]

def sign_psbt_b64(psbt_b64: str, mnemonic: str) -> str:
    seed = bip39.mnemonic_to_seed(mnemonic)
    root = bip32.HDKey.from_seed(seed, version=NET["xprv"])
    fp = root.child(0).fingerprint

    psbt = PSBT.from_string(psbt_b64)

    for inp in psbt.inputs:
        for pub, (master_fp, path) in inp.bip32_derivations.items():
            if master_fp == fp:
                child = root.derive(path)
                # embit signs in-place; adds entries to inp.partial_sigs
                psbt.sign_with(child, hash_type=None)
                break

    return psbt.to_string()
```

`sign_with` walks all inputs and signs whatever it can derive from the
provided HD key. For Tapscript inputs, embit also looks up
`tap_bip32_derivations` and `tap_internal_key`.

## On a SeedSigner / Krux device

The same API runs verbatim on the device; the surrounding code is just
camera/QR I/O:

```python
import sys
sys.path.append("/sd/embit")           # frozen modules path

from embit.psbt import PSBT
psbt_b64 = read_qr_animated()           # device-specific helper
signed = sign_psbt_b64(psbt_b64, ...)
display_qr_animated(signed)             # device-specific helper
```

Frozen-bytecode builds shrink RAM use significantly; on ESP32-class
devices, you typically freeze `embit.bip32`, `embit.bip39`, `embit.psbt`,
`embit.script`, `embit.networks` and exclude the rest.

## Common pitfalls

- MicroPython's `int` is unbounded but slow for 256-bit math; embit
  avoids unnecessary big-int ops, but compute bounds (e.g. ECDSA via
  the device's accelerator if available) live in C extension stubs you
  may need to provide.
- `embit.psbt.PSBT.to_string()` returns the base64 form; for binary use
  `psbt.serialize()`.
- Stack depth: deeply nested derivations (`root.derive("m/x/y/z/...")`
  with long paths) can blow MicroPython's default 4 KB stack. Bump
  `MICROPY_TASK_STACK_SIZE` or derive incrementally.
- API differences vs `python-bitcoinlib`: not a drop-in replacement.
  `from bitcoin import ...` (python-bitcoinlib) has different module
  layout from `from embit import ...`.
- Liquid extensions (`embit.liquid`) require additional curve ops; only
  freeze them if the device targets Liquid.

## References

- Repo: https://github.com/diybitcoinhardware/embit
- Frozen MicroPython firmware (Specter DIY): https://github.com/cryptoadvance/specter-diy
- SeedSigner: https://github.com/SeedSigner/seedsigner
- Companion: [../../hardware/seedsigner/SKILL.md](../../hardware/seedsigner/SKILL.md), [../../hardware/krux/SKILL.md](../../hardware/krux/SKILL.md)
