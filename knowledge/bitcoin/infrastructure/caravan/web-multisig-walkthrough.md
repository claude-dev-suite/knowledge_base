# Caravan Web Multisig Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/caravan`.
> Canonical source: https://unchained-capital.github.io/caravan/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/caravan/SKILL.md

## Concept

Caravan is a browser-only multisig coordinator from Unchained Capital. It
runs entirely as a static React app, never sees private keys, and stores
no wallet state on a server - all wallet config lives in a JSON file the
user must download and back up. Hardware wallet communication happens via
a local "HWI Bridge" (an HTTP proxy to the bitcoin-core/HWI library) so
keys never leave the device. This article walks the full 2-of-3 setup on
mainnet, address verification, and a PSBT-based spend, all without
installing the Caravan app itself - just a browser and a local HWI bridge.

## Walkthrough / mechanics

Prerequisites:

- A modern browser (Chrome/Brave/Firefox).
- HWI 3.x installed locally:
  ```bash
  pip install hwi
  ```
- Optional: a self-hosted Caravan static build (for offline / air-gapped
  use), `git clone https://github.com/caravan-bitcoin/caravan && npm i &&
  npm run build` then serve `build/` with any static file server.
- Bitcoin Core RPC (optional - for broadcasting; otherwise use any
  block explorer's POST endpoint).

Step 1 - start HWI bridge. Caravan talks to HWI over HTTP via the
`hermit` or `caravan-hwi-bridge` adapter:

```bash
git clone https://github.com/caravan-bitcoin/caravan-hwi-bridge
cd caravan-hwi-bridge
npm install && npm start
# bridge listening on http://127.0.0.1:8080
```

Step 2 - open Caravan, click "Create Wallet". Configuration form:

```
Network: mainnet
Address Type: P2WSH (Native SegWit)
Quorum: 2 of 3
Client: Mainnet -> Public (Block Explorer) | Private (RPC)
Key 1: Method = "Hardware Wallet"  -> Coldcard
Key 2: Method = "Hardware Wallet"  -> Trezor
Key 3: Method = "Hardware Wallet"  -> BitBox02
```

For each key Caravan calls the bridge to enumerate the device, fetches the
xpub at `m/48'/0'/0'/2'`, displays the master fingerprint, and lets you
verify the xpub on the device's screen.

Step 3 - save the wallet config JSON. Caravan generates:

```json
{
  "name": "Family Vault",
  "addressType": "P2WSH",
  "network": "mainnet",
  "client": { "type": "public" },
  "quorum": { "requiredSigners": 2, "totalSigners": 3 },
  "extendedPublicKeys": [
    { "name": "Coldcard", "bip32Path": "m/48'/0'/0'/2'",
      "xpub": "xpub6E...", "method": "xpub",
      "xfp": "d34db33f" },
    { "name": "Trezor",   "bip32Path": "m/48'/0'/0'/2'",
      "xpub": "xpub6F...", "xfp": "a1b2c3d4" },
    { "name": "BitBox02", "bip32Path": "m/48'/0'/0'/2'",
      "xpub": "xpub6G...", "xfp": "99887766" }
  ],
  "startingAddressIndex": 0
}
```

Click "Download Configuration" - **this file is required for recovery**.
Save offline copies; the seeds alone do not restore the wallet because
the threshold and key ordering live only here.

Step 4 - generate addresses. Wallet view -> "Receive" -> Caravan derives
addresses from the descriptor. For each address you intend to use, click
"Verify on Device" - the bridge sends the address to each HW wallet and
the device displays it for comparison. **All three must match.**

Step 5 - spending. Wallet -> "Send":

```
1. Enter recipient + amount + fee.
2. Caravan builds an unsigned PSBT.
3. For each signer slot: click "Sign with <device>".
4. HWI bridge ferries the PSBT to the device, user reviews, signs.
5. Caravan combines partials.
6. Once threshold reached, click "Broadcast" (uses configured client).
```

## Worked example

Assemble and inspect a Caravan-built PSBT manually:

```bash
# Caravan exports unsigned.psbt (base64) - decode locally
PSBT=$(cat unsigned.psbt)
bitcoin-cli decodepsbt "$PSBT" \
  | jq '{
      version: .tx.version,
      vin_count: (.inputs | length),
      vout_count: (.outputs | length),
      first_input_derivs: .inputs[0].bip32_derivs
    }'
```

Sample output:

```json
{
  "version": 2,
  "vin_count": 1,
  "vout_count": 2,
  "first_input_derivs": [
    { "master_fingerprint": "d34db33f", "path": "m/48h/0h/0h/2h/0/4" },
    { "master_fingerprint": "a1b2c3d4", "path": "m/48h/0h/0h/2h/0/4" },
    { "master_fingerprint": "99887766", "path": "m/48h/0h/0h/2h/0/4" }
  ]
}
```

Sign one of the partials directly with HWI (handy for offline machines
where Caravan's bridge cannot reach the device):

```bash
hwi --device-type coldcard signtx "$PSBT"
# returns the partial-signed PSBT
```

Then drop the result back into Caravan's "Add Signature" textbox.

| Step | Action | Time |
|------|--------|------|
| Setup | Devices + bridge online | 5 min |
| Wallet creation | Caravan UI, save config | 5 min |
| Address verification | All three devices | 3 min/address |
| Receive small test | 0.0001 BTC | 1 confirmation |
| Spend round trip | Sign w/ 2 devices, broadcast | 10-15 min |

## Common pitfalls

- Losing the wallet config JSON - this is the single most common Caravan
  recovery failure; print it, encrypt-backup it, give a copy to a
  co-signer.
- Using the public Caravan instance with high-value vaults - even though
  the keys never leave devices, the page is loaded from the internet;
  for cold storage prefer the self-hosted static build, served from a
  dedicated machine.
- Trusting xpubs the bridge reports without on-device verification - a
  compromised bridge could substitute its own xpub. Always verify each
  xpub on each device's screen during onboarding.
- BIP32 path mismatch - Caravan defaults to `m/48'/0'/0'/2'` (native
  multisig); some users (or older Coldcard firmware) export at `m/45'`
  and the addresses won't match.
- Bridge-vs-device USB conflicts - only one process can hold the HID
  handle at a time; quit Specter / Sparrow before using Caravan's bridge.
- Public-block-explorer privacy - in "Public" client mode Caravan queries
  blockstream.info or mempool.space for balance/tx data, which leaks
  the addresses to that operator. Use "Private RPC" with your own node
  for anonymous use.

## References

- Caravan: https://github.com/caravan-bitcoin/caravan
- Caravan docs: https://unchained-capital.github.io/caravan/
- HWI: https://github.com/bitcoin-core/HWI
- BIP380 descriptors: https://github.com/bitcoin/bips/blob/master/bip-0380.mediawiki
