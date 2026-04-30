# Ledger BOLOS App Architecture - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/ledger`.
> Canonical source: https://developers.ledger.com/docs/device-app/introduction
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/ledger/SKILL.md

## Concept

BOLOS (Blockchain Open Ledger Operating System) is the OS that runs
inside the Secure Element (SE) on every Ledger device. Each cryptocurrency
is a **separate signed app** loaded as ELF into the SE; only one app is
active at a time. BOLOS isolates each app in its own sandbox: an app
cannot read another app's seed material, and a Bitcoin app cannot leak
keys to a malicious altcoin app loaded later.

Apps are written in C using the **Ledger SDK** and signed by Ledger's
hardware-management key before being side-loaded by the user via Ledger
Live or `ledgerctl`.

## Walkthrough

Bitcoin app source tree (`app-bitcoin-new`):

```
src/
  main.c                # entry point, APDU dispatcher
  apdu_dispatcher.c     # CLA/INS routing
  handler/              # per-command handlers
    get_extended_pubkey.c
    sign_psbt.c
    get_wallet_address.c
    register_wallet.c
  swap/                 # external "Exchange" app integration
  ui/                   # UX flows (NBGL on Stax/Flex, BAGL on Nano)
```

Communication is **APDU** (ISO 7816-style) over USB-HID or Bluetooth.
Each command is a `CLA INS P1 P2 LC <data>` frame. The Bitcoin app
defines:

| INS | Command | Notes |
|-----|---------|-------|
| 0x00 | GET_EXTENDED_PUBKEY | xpub at path |
| 0x02 | REGISTER_WALLET | bind a wallet policy + HMAC |
| 0x03 | GET_WALLET_ADDRESS | display address for verification |
| 0x04 | SIGN_PSBT | sign a PSBT (chunked) |
| 0x10 | GET_MASTER_FINGERPRINT | quick fp query |

The host streams large data (PSBT, descriptors) in chunks because the
SE has limited RAM (~30 KB on Nano S Plus). The protocol uses a
**client-state**-driven flow: device asks for specific chunks back,
host serves them.

## Worked example

Querying a Ledger Bitcoin app from Python:

```python
from ledger_bitcoin import createClient, Chain, WalletPolicy
from ledger_bitcoin.client import TransportClient

transport = TransportClient(interface="hid")
client = createClient(transport, chain=Chain.MAIN)

# Query master fingerprint
fp = client.get_master_fingerprint()
# Get xpub at a path
xpub = client.get_extended_pubkey("m/84'/0'/0'", display=False)

# Register a single-sig policy (auto, no HMAC needed for default policies)
policy = WalletPolicy(
    name="",
    descriptor_template="wpkh(@0/**)",
    keys_info=[f"[{fp.hex()}/84'/0'/0']{xpub}"]
)
hmac = None  # default registered = None

# Sign PSBT
result = client.sign_psbt(psbt, policy, hmac)
```

App lifecycle on the device:

1. Bitcoin app icon -> user selects -> BOLOS jumps to app entry.
2. App initializes UI buffers, derives master fp.
3. Loop: read APDU, dispatch handler, write response.
4. On Bitcoin app exit -> BOLOS wipes app RAM, returns to dashboard.

Wallet policies for multisig must be **registered first** (`register_wallet`
INS); device returns 32-byte HMAC. Subsequent `sign_psbt` for that policy
must include the HMAC or device rejects.

## Common pitfalls

- **Wrong app open**: e.g., Ethereum app open while host sends Bitcoin
  CLAs. Returns `0x6E00` (CLA not supported).
- **Path length limits**: Ledger SDK truncates paths > 10 levels. Stay
  under that for derivation.
- **HMAC drift**: registering a wallet policy on one device and trying
  to sign on another (different SE) - HMAC will not validate. Each
  device-policy pair has its own HMAC.
- **Bluetooth + heavy PSBT**: Nano X BLE drops chunks under load. USB
  is more reliable for large multisig PSBTs.
- **App version mismatch**: legacy `0x40` APDUs from old `hw-app-btc`
  vs modern `0x00` APDUs from `ledger-bitcoin`. Ensure the host
  library matches your app version (v2.1+ uses the new API).

## References

- BOLOS docs: https://developers.ledger.com/docs/device-app/introduction
- app-bitcoin-new: https://github.com/LedgerHQ/app-bitcoin-new
- ledger-bitcoin Python client: https://github.com/LedgerHQ/app-bitcoin-new/tree/master/bitcoin_client
