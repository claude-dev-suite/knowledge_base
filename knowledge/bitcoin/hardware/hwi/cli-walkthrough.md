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

> **Lifecycle note (September 2026).** HWI is winding down. In issue
> [#850](https://github.com/bitcoin-core/HWI/issues/850) (18 August 2026)
> Ava Chow announced that HWI will finish MuSig2, cut a likely final
> release, then hold in minimal maintenance mode until a replacement is
> ready and be archived. No new features and no new device support are
> being accepted. Wizardsardine's Rust
> [BHWI](https://github.com/wizardsardine/bhwi) (WIP as of September 2026)
> is named as the successor. Everything below describes HWI 3.2.0
> (10 February 2026), the current release.

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
# Download from GitHub releases; 3.2.0 is the current release (Feb 2026)
wget https://github.com/bitcoin-core/HWI/releases/download/3.2.0/hwi-3.2.0-linux-x86_64.tar.gz
# Also published: linux-aarch64, mac-arm64, mac-x86_64 (.tar.gz) and
# windows-x86_64 (.zip), plus SHA256SUMS.txt.asc for GPG verification.
```

### Linux udev rules

```bash
# Simplest: let HWI do it
sudo hwi installudevrules

# Or by hand, from a source checkout
sudo cp hwilib/udev/*.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo usermod -aG plugdev $USER
# Logout / login
```

### Detect devices

```bash
hwi enumerate
# Since 3.0.0 (April 2024) emulators and simulators are ignored by
# default; pass --emulators to have them enumerated too.
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
- **Wallet policy registration for Ledger multisig**: released HWI
  (through 3.2.0) has *no* registration subcommand and no
  `--wallet-name` / `--wallet-hmac` flags. The Ledger driver rebuilds
  the wallet policy from the PSBT and registers it inline on each
  `signtx` / `displayaddress`, so the user re-confirms the policy on the
  device every run. An explicit `hwi registerdescriptor` plus
  `--registration` on `signtx` / `displayaddress` is merged on master
  (PRs #841, #792, August 2026) but unreleased as of September 2026.
- **Stale HWI version**: Bitcoin Core's bundled HWI may lag. Install a
  newer HWI and point Core's `external_signer` at the new path.
- **Bridge daemons**: Trezor Bridge or BitBoxBridge running on host
  conflict with HWI's direct access. Stop bridges when using HWI.
- **Windows USB driver**: Trezor + Ledger require Zadig-installed
  WinUSB drivers to bypass HID class issues.

## References

- HWI repo: https://github.com/bitcoin-core/HWI
- HWI releases: https://github.com/bitcoin-core/HWI/releases
  (3.2.0, 10 Feb 2026 - Jade Plus, BitBox02 Nova, Testnet4, native Jade
  PSBT signing, PSBT MuSig2 fields; 3.1.0, 17 Sep 2024 - Trezor Safe 5,
  new Ledger udev rules/model IDs, `tr()` single-leaf parsing fix;
  3.0.0, 6 Apr 2024 - `--emulators`, emulators ignored by default)
- HWI wind-down announcement: https://github.com/bitcoin-core/HWI/issues/850
- BHWI (Rust successor, WIP): https://github.com/wizardsardine/bhwi
- HWI docs: https://hwi.readthedocs.io
- Bitcoin Core external signer doc: `doc/external-signer.md`
