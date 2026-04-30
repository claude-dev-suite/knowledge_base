# HWI CLI Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/hwi`.
> Canonical source: https://github.com/bitcoin-core/HWI
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/hwi/SKILL.md

## Concept

HWI (Hardware Wallet Interface) is a Python CLI + library that provides
a unified API across Trezor, Ledger, Coldcard, BitBox02, Jade, and
KeepKey. It abstracts the vendor-specific transports (Trezor's gRPC-like
protocol, Ledger's APDU, Coldcard's CKCC, BitBox's noise protocol, Jade's
serial CBOR) behind one set of commands: `enumerate`, `getxpub`,
`displayaddress`, `signtx`, `signmessage`, `setupdevice`, `wipe`, etc.

Bitcoin Core uses HWI as its **external signer**: Core RPC calls
shell out to `hwi.exe`/`hwi` for signing operations, letting
`bitcoin-cli walletprocesspsbt` work with hardware wallets seamlessly.

## Walkthrough

### Installation

```bash
# Pip
pip install hwi

# Or from source
git clone https://github.com/bitcoin-core/HWI
cd HWI
pip install -e .

# Standalone binary (no Python needed)
# Download from GitHub releases
wget https://github.com/bitcoin-core/HWI/releases/download/2.4.0/hwi-2.4.0-linux-amd64.tar.gz
```

### Linux udev rules

```bash
# Required for non-root USB access
sudo cp hwi/udev/*.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo usermod -aG plugdev $USER
# Logout / login
```

### Detect devices

```bash
hwi enumerate
[
    {
        "type": "trezor",
        "model": "trezor_t",
        "path": "webusb:001:6:1:2",
        "fingerprint": "d34db33f",
        "needs_pin_sent": false,
        "needs_passphrase_sent": false
    },
    {
        "type": "coldcard",
        "model": "coldcard",
        "path": "0001:0007:00",
        "fingerprint": "abcd1234"
    }
]
```

### Get xpub

```bash
# By fingerprint
hwi -f d34db33f getxpub "m/84'/0'/0'"

# By type + path
hwi --device-type trezor --device-path "webusb:001:6:1:2" \
    getxpub "m/84h/0h/0h"

# Output:
# {"xpub": "xpub6CUGRUonZSQ4..."}
```

### Display address (verify on device)

```bash
hwi -f d34db33f displayaddress \
    --path "m/84h/0h/0h/0/0" \
    --addr-type wit
# Device screen shows the address; user verifies.
```

### Sign a PSBT

```bash
PSBT=$(cat my.psbt | base64 -w0)
hwi -f d34db33f signtx "$PSBT" > signed.psbt
```

### Setup / wipe / restore

```bash
# Wipe a device (destroys all keys; requires user confirmation on device)
hwi --device-type trezor wipe

# Setup new device (interactive)
hwi --device-type trezor setup --label "MyTrezor"

# Restore from mnemonic (Trezor only; never enters mnemonic on host - uses dry_run)
hwi --device-type trezor restore
```

## Worked example

Bitcoin Core integration via external signer:

```bash
# 1. Tell Core where HWI is
bitcoin-cli -named -rpcwallet=signer createwallet \
    wallet_name="hw_signer" \
    disable_private_keys=true \
    blank=true \
    descriptors=true \
    external_signer=true

# 2. Set the external signer command
bitcoin-cli -rpcwallet=hw_signer setexternalsigner '"hwi --chain main"'

# 3. Import descriptor from device
bitcoin-cli -rpcwallet=hw_signer enumeratesigners
# Returns descriptors at standard paths

# 4. Generate addresses (verify on device with displayaddress)
bitcoin-cli -rpcwallet=hw_signer getnewaddress

# 5. Build + sign PSBT, Core calls HWI under the hood
bitcoin-cli -rpcwallet=hw_signer walletcreatefundedpsbt ...
bitcoin-cli -rpcwallet=hw_signer walletprocesspsbt <psbt>
# Above triggers HWI to sign on the connected device.
```

## Common pitfalls

- **Multiple devices same fingerprint**: rare but if you have two
  copies of the same seed (HW1 hot, HW2 vault), HWI sees them as
  identical. Use `--device-path` to disambiguate.
- **PIN/passphrase**: Trezor blocks until user types PIN on device or
  via host (Trezor One uses host matrix; Model T device-only). HWI
  surfaces a `needs_pin_sent` flag.
- **Wallet policy registration for Ledger multisig**: HWI 2.x supports
  this via `--wallet-name` and `--wallet-hmac` arguments. Without
  them, Ledger rejects multisig PSBTs.
- **Stale HWI version**: Bitcoin Core's bundled HWI may lag. Install a
  newer HWI and point Core's `external_signer` at the new path.
- **Bridge daemons**: Trezor Bridge or BitBoxBridge running on host
  conflict with HWI's direct access. Stop bridges when using HWI.
- **Windows USB driver**: Trezor + Ledger require Zadig-installed
  WinUSB drivers to bypass HID class issues.

## References

- HWI repo: https://github.com/bitcoin-core/HWI
- HWI docs: https://hwi.readthedocs.io
- Bitcoin Core external signer doc: `doc/external-signer.md`
