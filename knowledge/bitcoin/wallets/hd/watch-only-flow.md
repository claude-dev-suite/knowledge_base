# Watch-Only Wallet Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/hd`.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/hd/SKILL.md

## Concept

A watch-only wallet holds public keys, descriptors, and metadata but no
signing material. It can monitor balances, build PSBTs, and forward them to
an offline signer. The architectural split mirrors PSBT roles (BIP174):
**Updater** (knows xpub, builds tx) is online; **Signer** (holds xprv) is
offline. The watch-only wallet is the Updater.

Three reasons to deploy this pattern:

1. **Hot/cold separation** — production server (Updater) never touches
   private keys.
2. **Multi-device sign** — laptop builds, hardware wallet signs.
3. **Compliance** — auditors get visibility into balances without spending
   power.

## Walkthrough / mechanics

The minimum data the Updater needs:

- Account-level extended public key (e.g., from `m/84'/0'/0'`).
- Master fingerprint (4 bytes) and the path used (`84h/0h/0h`).
- Script type indicator — usually expressed as the descriptor wrapper
  (`wpkh()`, `tr()`, `wsh(sortedmulti(2,...))`).
- Range of indices to scan (gap limit).

For multisig the Updater needs the same data for each cosigner xpub, plus
quorum (`sortedmulti(M, ...)`).

Once the descriptor is loaded, the Updater can:

- Derive arbitrary receive addresses (`deriveaddresses`).
- Scan UTXO set / chain (`scantxoutset`, `importdescriptors timestamp:0`).
- Build PSBTs (`walletcreatefundedpsbt`) where each input includes
  `bip32_derivs` so the Signer knows which key path to use.
- Verify signed PSBTs and broadcast.

The Updater never receives private keys back from the Signer. Final tx is
the only thing that crosses the cold/hot boundary in the broadcast
direction.

## Worked example

Spinning up a watch-only Bitcoin Core wallet from a hardware wallet xpub:

```bash
# 1. Create a private-keys-disabled wallet
bitcoin-cli createwallet "watch_hw" \
  disable_private_keys=true blank=true descriptors=true load_on_startup=true

# 2. Import the descriptor (account xpub from Coldcard or similar)
ACCT_XPUB="xpub6CUGRUonZSQ4TWtTMmzXdrXDtypWKiKrhko4egpiMZbpiaQL2jkwSB1icqYh2cfDfVxdx4df189oLKnC5fSwqPfgyP3hooxujYzAu3fDVmz"
FP="73c5da0a"
DESC="wpkh([${FP}/84h/0h/0h]${ACCT_XPUB}/<0;1>/*)"
DESC_CK=$(bitcoin-cli getdescriptorinfo "$DESC" | jq -r .descriptor)

bitcoin-cli -rpcwallet=watch_hw importdescriptors "[
  {\"desc\":\"${DESC_CK}\",\"active\":true,\"range\":[0,999],\"timestamp\":\"now\"}
]"

# 3. Get a receive address
bitcoin-cli -rpcwallet=watch_hw getnewaddress

# 4. After funds arrive, build a PSBT
bitcoin-cli -rpcwallet=watch_hw walletcreatefundedpsbt \
  '[]' '[{"bc1qrecipient...":0.0005}]' 0 '{"feeRate":0.00002}'
# returns base64 PSBT

# 5. Save psbt.txt, transfer to airgapped signer (QR / SD card)
# 6. Signer returns signed PSBT
# 7. Finalize and broadcast
psbt_signed=$(cat signed.psbt)
hex=$(bitcoin-cli finalizepsbt "$psbt_signed" | jq -r .hex)
bitcoin-cli sendrawtransaction "$hex"
```

For Sparrow / Specter the same data flows via the QR-code-driven UR2.0
PSBT exchange standard (BCR-2020-006).

## Multisig watch-only

The descriptor template for a 2-of-3 wsh sortedmulti with three xpubs from
different devices:

```
wsh(sortedmulti(2,
   [aaaaaaaa/48h/0h/0h/2h]xpub1.../<0;1>/*,
   [bbbbbbbb/48h/0h/0h/2h]xpub2.../<0;1>/*,
   [cccccccc/48h/0h/0h/2h]xpub3.../<0;1>/*))#checksum
```

`sortedmulti` (BIP67) sorts pubkeys lexicographically per leaf so the
address is deterministic regardless of import order. `multi` does not sort
and breaks if cosigners load xpubs in different order.

## Common pitfalls

- Forgetting `range`: `importdescriptors` defaults to `[0,1000]` only when
  active=true; out-of-range deposits go invisible.
- `timestamp:0` triggers a full chain rescan — a fresh node may take 12+
  hours. Use `now` if you know no funds existed before, or supply a
  realistic Unix epoch matching the seed creation date.
- Importing the wallet xpub instead of the *account* xpub: the account
  xpub corresponds to the third hardened depth (`m/84'/0'/0'`), giving the
  Updater visibility into one account only. Importing the master xpub
  exposes too much and breaks tooling that expects depth-3 xpubs.
- Inserting an xprv into a `disable_private_keys=true` wallet — Core
  rejects with an error; this is intentional. Use a separate signing wallet.
- `walletprocesspsbt` on a watch-only wallet fills in `bip32_derivs` and
  HD path metadata but cannot sign. The output PSBT is still useful as
  a "Updater finalization step" before forwarding to the Signer.

## References

- BIP174 PSBT: https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki
- BIP67 deterministic pubkey ordering: https://github.com/bitcoin/bips/blob/master/bip-0067.mediawiki
- UR2.0 BCR-2020-006: https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2020-006-urtypes.md
- See also: [bip44-49-84-86-comparison.md](bip44-49-84-86-comparison.md), [../../core/descriptors-wallet/import-flow-walkthrough.md](../../core/descriptors-wallet/import-flow-walkthrough.md)
