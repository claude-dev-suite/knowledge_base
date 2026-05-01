# Descriptor Import Flow Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/descriptors-wallet`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/doc/descriptors.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/descriptors-wallet/SKILL.md

## Concept

`importdescriptors` is the single RPC that loads a descriptor (and optionally its keys) into a Bitcoin Core wallet. The mental model: you are not "importing addresses" but registering a *function* (the descriptor) plus a *domain* (the `range`) plus a *time origin* (the `timestamp`). Core then derives every address in that domain, watches the chain from the timestamp forward, and treats matches as belonging to this wallet. Getting any of those three wrong produces silent failures: no addresses appear, balances stay at zero, or the rescan hammers the disk for hours.

## Walkthrough / mechanics

A descriptor import is a JSON object with mandatory and optional fields:

- `desc`: the descriptor string with checksum (`#chk`). Required. Multipath descriptors using `<0;1>` import receive and change in one entry.
- `active`: marks this descriptor as the source of new addresses for `getnewaddress`/`getrawchangeaddress`. Only one active descriptor per (script type, internal/external) slot.
- `internal`: `true` for change. Default `false`.
- `range`: `[end]` or `[start, end]` integers. Defines which child indexes Core will derive. Without it, only index 0 is derived.
- `timestamp`: either `"now"` (no historical scan), an integer Unix time, or `0` (full chain rescan). Determines from when blocks are scanned for matches.
- `next_index`: where `getnewaddress` starts handing out addresses. Default `0`.
- `label`: optional address label.

`importdescriptors` always returns an array, even for one entry. Each entry has a `success` boolean and a `warnings` array. If `timestamp` triggers a rescan, the call blocks until the rescan completes; for "now" it returns instantly.

A descriptor wallet must be created with `descriptors=true` and ideally `blank=true` if you intend to import only your own. A non-blank wallet auto-generates four script types of descriptors and you'll then have a mix.

## Worked example

End-to-end import of a watch-only wallet from an external xpub at BIP84 path:

```bash
# 1. Create blank watch-only wallet
$ bitcoin-cli -named createwallet \
    wallet_name="cold-watch" \
    disable_private_keys=true \
    blank=true \
    descriptors=true \
    load_on_startup=true
{"name":"cold-watch","warning":""}

# 2. Compute checksum if you only have the bare descriptor
$ bitcoin-cli getdescriptorinfo "wpkh([d34db33f/84h/0h/0h]xpub6CV2.../<0;1>/*)"
{
  "descriptor": "wpkh([d34db33f/84h/0h/0h]xpub6CV2.../<0;1>/*)#5sm0xagy",
  "checksum": "5sm0xagy",
  "isrange": true,
  "issolvable": true,
  "hasprivatekeys": false
}

# 3. Import as active receiving descriptor (multipath handles change too)
$ bitcoin-cli -rpcwallet=cold-watch importdescriptors '[
  {
    "desc": "wpkh([d34db33f/84h/0h/0h]xpub6CV2.../<0;1>/*)#5sm0xagy",
    "active": true,
    "internal": false,
    "range": [0, 999],
    "next_index": 0,
    "timestamp": "now"
  }
]'
[{"success":true,"warnings":["Range not given, using default keypool size"]}]
```

A historical wallet (rescan from a known birth time):

```bash
# Owner remembers the wallet was first funded around 2023-06-15 (Unix 1686787200)
$ bitcoin-cli -rpcwallet=cold-watch importdescriptors '[
  {
    "desc": "wpkh([d34db33f/84h/0h/0h]xpub6CV2.../<0;1>/*)#5sm0xagy",
    "active": true,
    "range": [0, 4999],
    "timestamp": 1686787200
  }
]'
# Blocks for several minutes while Core scans every block since June 2023
[{"success":true,"warnings":[]}]
```

Verify and use:

```bash
$ bitcoin-cli -rpcwallet=cold-watch listdescriptors | jq '.descriptors[].desc'
"wpkh([d34db33f/84h/0h/0h]xpub6CV2.../0/*)#abc"
"wpkh([d34db33f/84h/0h/0h]xpub6CV2.../1/*)#def"

$ bitcoin-cli -rpcwallet=cold-watch deriveaddresses \
    "wpkh([d34db33f/84h/0h/0h]xpub6CV2.../0/*)#abc" '[0,4]'
["bc1q...", "bc1q...", "bc1q...", "bc1q...", "bc1q..."]

$ bitcoin-cli -rpcwallet=cold-watch getbalances
{"mine":{"trusted":0.21500000,"untrusted_pending":0,"immature":0}, ...}
```

## Common pitfalls

- Forgetting `range`. Default keypool is 1000 but Core only derives index 0 if `range` is absent on import. You see addresses for index 0 and nothing else.
- Setting `timestamp: 0` accidentally. This triggers a rescan from genesis and can take many hours.
- Importing a descriptor without `<0;1>` multipath and no `internal` companion entry. New change addresses come from the auto-generated descriptors, not yours, leading to surprising change addresses.
- Skipping the checksum (`#chk`). Returns `{"success": false, "error": {"code": -5, "message": "Missing checksum"}}`. Always run `getdescriptorinfo` first.
- `importdescriptors` against a legacy wallet. Returns "Only descriptor wallets support this RPC". Migrate first with `migratewallet` or recreate.
- Marking two descriptors as `active` for the same script type. The second silently replaces the first as the source of new addresses; the first remains usable but stops issuing new ones.
- Importing the same descriptor twice with different `next_index`. Core keeps the larger of the two values, which can cause an address gap.

## References

- `doc/descriptors.md` in bitcoin/bitcoin.
- BIP 380 (descriptor language), BIP 389 (multipath `<a;b>`).
- `src/wallet/rpc/backup.cpp` for `importdescriptors` implementation.
- `getdescriptorinfo` and `deriveaddresses` RPCs.
