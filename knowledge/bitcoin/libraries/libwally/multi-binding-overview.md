# libwally-core Multi-Binding Overview - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/libwally`.
> Canonical source: https://github.com/ElementsProject/libwally-core
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/libwally/SKILL.md

## Concept

`libwally-core` is Blockstream's wallet primitives library: BIP32/39/85,
PSBT (BIP174), Bitcoin tx serialization, BIP143/341 sighash, BIP340
Schnorr, and -- when built with `--enable-elements` -- Liquid
Confidential Transactions. It is intentionally **smaller than a node**:
no consensus rules, no chain validation, no P2P. The goal is one
audited C core that every binding consumes identically.

Bindings ship for:

- **Python**: `wallycore` (PyPI).
- **JavaScript / TS**: `libwally-core-js` via WebAssembly.
- **Java / Kotlin**: `wally-jvm` and `wally-android`.
- **Swift**: native bindings used by Blockstream Green iOS.
- **C**: native API.

The same function names appear in every binding (e.g.
`wally_bip39_mnemonic_to_seed`, `wally_bip32_key_from_seed`,
`wally_psbt_from_base64`).

## Build the C core

```bash
git clone https://github.com/ElementsProject/libwally-core
cd libwally-core
./tools/autogen.sh
./configure --enable-elements --enable-shared
make -j
sudo make install
```

`--enable-elements` adds Liquid CT helpers; omit if you only need
Bitcoin.

## Python binding walkthrough

```python
import wallycore as wally

# BIP39 mnemonic -> 64-byte seed
seed = bytearray(64)
wally.bip39_mnemonic_to_seed(
    "abandon abandon abandon abandon abandon abandon "
    "abandon abandon abandon abandon abandon about",
    "",                                      # passphrase
    seed)

# Master xprv
master = wally.bip32_key_from_seed(
    bytes(seed), wally.BIP32_VER_MAIN_PRIVATE,
    wally.BIP32_FLAG_KEY_PRIVATE)

# Derive m/84'/0'/0'
account = wally.bip32_key_from_parent_path(
    master,
    [
        84 + wally.BIP32_INITIAL_HARDENED_CHILD,
        0  + wally.BIP32_INITIAL_HARDENED_CHILD,
        0  + wally.BIP32_INITIAL_HARDENED_CHILD,
    ],
    wally.BIP32_FLAG_KEY_PRIVATE)

xpub_str = wally.bip32_key_to_base58(account, wally.BIP32_FLAG_KEY_PUBLIC)
print("xpub:", xpub_str)
```

## JavaScript (WASM) binding

```js
import { initWally, wally } from "libwally-core";   // package wraps WASM

await initWally();

const seed = new Uint8Array(64);
await wally.bip39_mnemonic_to_seed(
    "abandon abandon ... about", "", seed);

const master = await wally.bip32_key_from_seed(
    seed, wally.BIP32_VER_MAIN_PRIVATE, wally.BIP32_FLAG_KEY_PRIVATE);

const xpubB58 = await wally.bip32_key_to_base58(
    master, wally.BIP32_FLAG_KEY_PUBLIC);
console.log(xpubB58);
```

## Worked example: PSBT round-trip in Python

```python
import wallycore as wally

# Parse base64 PSBT
psbt_b64 = "cHNidP8BAH..."
psbt = wally.psbt_from_base64(psbt_b64)

# Inspect input count
n = wally.psbt_get_num_inputs(psbt)
print("inputs:", n)

# Sign each input (assumes BIP32 derivations are populated)
seckey = bytes.fromhex("01" * 32)
for i in range(n):
    wally.psbt_sign_input(psbt, i, seckey, 0)

# Finalize + extract tx
wally.psbt_finalize(psbt, 0)
tx = wally.psbt_extract(psbt, 0)
raw_hex = wally.tx_to_hex(tx, 0)
print(raw_hex)

wally.tx_free(tx)
wally.psbt_free(psbt)
```

The Python binding follows the C API closely: caller-allocated buffers,
explicit `*_free` for handles, integer flags for behaviour. The JS
binding hides allocation but mirrors the same function signatures.

## Common pitfalls

- **Manual memory management**: every `wally_*_alloc` or handle
  returned via `psbt_from_base64` must be paired with a `*_free`. The
  Python and JS bindings do not GC these for you in older versions.
- **Buffer sizing**: many functions accept a caller-provided buffer
  plus its length. Passing an undersized buffer truncates silently in
  some bindings and asserts in others -- check `wally_*_get_length`
  helpers for the right size.
- **Elements build flag**: code using `wally_asset_*` or
  `wally_confidential_addr_*` only links when libwally was built with
  `--enable-elements`.
- **Threading**: contexts are thread-safe but heavy. For high-volume
  Liquid services, batch operations on a single thread.
- **Binding version skew**: the JS WASM binding sometimes lags the C
  library by a release. Pin the C library and the binding together.

## References

- Repo: https://github.com/ElementsProject/libwally-core
- API docs: https://wally.readthedocs.io
- PyPI: https://pypi.org/project/wallycore/
- Companion: [secp256k1-c/SKILL.md](../secp256k1-c/SKILL.md), [../../l2/liquid/SKILL.md](../../l2/liquid/SKILL.md)
