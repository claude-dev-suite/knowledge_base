# Descriptor Import Flow with Ranges - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/descriptors`.
> Canonical source: BIP380, Bitcoin Core descriptor documentation
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/descriptors/SKILL.md

## Concept

Importing a descriptor into Bitcoin Core (or any descriptor wallet)
requires choosing scan parameters: `range`, `timestamp`, `active`,
`internal`, `label`, `next_index`. These choices determine:

- How many addresses are scanned at import time (range).
- How far back the rescan goes (timestamp).
- Whether the descriptor backs the user-visible "new address" feature
  (active, internal).

Getting these wrong wastes hours on a rescan or fails to find existing
funds. The skill covers descriptor syntax; this article focuses on
the import RPC payload, gap-limit semantics, and the rescan timeline.

## Walkthrough / mechanics

The `importdescriptors` RPC takes an array of import requests:

```json
[
  {
    "desc": "wpkh([fp/84'/0'/0']xpub.../0/*)#chk",
    "active": true,
    "internal": false,
    "range": [0, 999],
    "next_index": 0,
    "timestamp": 1577836800,
    "label": "main wallet"
  },
  {
    "desc": "wpkh([fp/84'/0'/0']xpub.../1/*)#chk",
    "active": true,
    "internal": true,
    "range": [0, 999],
    "next_index": 0,
    "timestamp": 1577836800
  }
]
```

Field semantics:

| Field | Meaning |
|-------|---------|
| `desc` | The descriptor string with checksum. |
| `active` | Wallet-active. Only one active descriptor per (output type, internal) pair. Inactive imports are still scanned but not used for new-address generation. |
| `internal` | True = change addresses; not shown to user. |
| `range` | `[lo, hi]` inclusive index range to derive and scan. |
| `next_index` | Where the wallet starts allocating new addresses. |
| `timestamp` | Unix seconds; rescan from here forward. `"now"` skips rescan. |
| `label` | Optional label for the active receive descriptor. |

The wallet derives all addresses in `range`, registers them in the
scriptPubKey index, then scans the chain from `timestamp` onward.

## Worked example

A user is restoring a wallet from a 12-word seed. The wallet was
created on 2023-06-01 and has 47 receive addresses used.

**Step 1: derive xpubs locally** (don't ship seed to Core).

```bash
# Using a local tool (Coldcard, Sparrow, bdk-cli):
descriptor=wpkh([d34db33f/84h/0h/0h]xpub.../<0;1>/*)
descriptor_with_checksum=$(bitcoin-cli getdescriptorinfo "$descriptor" | jq -r .descriptor)
```

**Step 2: choose range generously.** Default gap limit is 20; if the
user has 47 used + a gap of up to 20, set range to `[0, 100]` to be
safe.

```bash
bitcoin-cli createwallet "restored" false true "" false true
bitcoin-cli -rpcwallet=restored importdescriptors '[
  {
    "desc": "'$descriptor_with_checksum'",
    "active": true,
    "range": [0, 100],
    "timestamp": 1685577600
  }
]'
```

`timestamp: 1685577600` is 2023-06-01 UTC, the wallet creation date.
Bitcoin Core rescans from the block containing that timestamp forward
(`getblockchaininfo` to find the height).

**Step 3: monitor rescan.**

```bash
bitcoin-cli -rpcwallet=restored getwalletinfo | jq .scanning
# { "duration": 0, "progress": 0.42 }
```

Rescan time depends on chain length from `timestamp`. From mid-2023
to today is ~150_000 blocks; expect ~30-60 minutes on consumer SSD.

**Step 4: verify funds.**

```bash
bitcoin-cli -rpcwallet=restored getbalances
bitcoin-cli -rpcwallet=restored listunspent
```

If a UTXO is missing, expand range:

```bash
bitcoin-cli -rpcwallet=restored importdescriptors '[
  {"desc":"'$descriptor_with_checksum'","range":[0,500],"timestamp":"now"}
]'
```

The new range covers the missing index. `timestamp: "now"` skips
rescan since UTXOs already in the chain at higher indices weren't
added before.

## Common bugs / pitfalls

1. **Range too small.** Default gap limit of 20 means the wallet stops
   scanning after 20 unused addresses in a row. If the original wallet
   used non-sequential indices (some software does for privacy),
   coins can be hidden beyond the range.
2. **Importing only the receive chain.** Forgetting the change
   descriptor (chain 1) means change UTXOs go undetected. Always
   import receive AND change with `<0;1>` multipath OR two separate
   imports.
3. **timestamp later than first usage.** Rescan misses early UTXOs.
   Use the wallet creation date or "0" for genesis-onward (very slow
   - days).
4. **timestamp = "now" on first import.** Skips rescan; all existing
   UTXOs are invisible. Only use "now" when extending range on a
   wallet you know was already fully scanned at that time.
5. **`active: true` conflict.** Two active descriptors for the same
   output type (e.g., two wpkh for receive) cause "already an active
   descriptor". Set the older one inactive first via
   `importdescriptors` with `active: false`.
6. **Re-importing a different descriptor with the same xpub.** Bitcoin
   Core allows it but creates separate scriptPubKey managers; address
   generation may flip-flop between them on `getnewaddress`. Avoid by
   always passing the exact descriptor string you want active.
7. **Hardened post-xpub derivation.** `xpub.../0'/*` fails because
   xpub cannot derive hardened. The hardened components must be in
   the key-origin path `[fp/...]`.

## References

- BIP380: https://github.com/bitcoin/bips/blob/master/bip-0380.mediawiki
- Bitcoin Core RPC docs: https://github.com/bitcoin/bitcoin/blob/master/doc/descriptors.md
- bdk wallet sync semantics: https://docs.rs/bdk/latest/bdk/wallet/struct.Wallet.html
