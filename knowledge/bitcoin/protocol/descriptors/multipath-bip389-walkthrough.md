# Multipath Descriptors (BIP389) Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/descriptors`.
> Canonical source: BIP389
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/descriptors/SKILL.md

## Concept

BIP389 introduces the `<...;...>` syntax to express multiple parallel
derivation paths in a single descriptor string. Before BIP389, a
wallet exporting receive AND change chains needed two separate
descriptors. BIP389 collapses them: `<0;1>/*` declares two ranged
chains, `0/*` (receive) and `1/*` (change), as siblings.

The compactness matters because hardware wallets and PSBT exchange
benefit from a single descriptor source-of-truth, and because some
script types (Taproot, miniscript) are awkward to express twice with
identical structure save for one component.

## Walkthrough / mechanics

Grammar:

```
MULTIPATH = "<" PATH_ELEM (";" PATH_ELEM)+ ">"
PATH_ELEM = NUMBER ("h" | "H" | "'")?
```

- Exactly ONE `<...>` block per descriptor; multiple multipath
  elements would create a 2D matrix.
- Each `PATH_ELEM` produces a parallel derivation chain.
- Hardened/unhardened mixing is permitted but rare.

When expanded by a wallet, `wpkh([fp/84'/0'/0']xpub.../<0;1>/*)`
produces TWO descriptor lineages:

```
wpkh([fp/84'/0'/0']xpub.../0/*)   # receive
wpkh([fp/84'/0'/0']xpub.../1/*)   # change
```

Each lineage has its OWN ranged scan with `next_index` tracked
independently. Bitcoin Core's `importdescriptors` accepts the
multipath form and creates two scriptPubKeyManagers under the hood.

Address generation: for index `n` on chain `c`:

```
xpub.../c/n  -> derive child key -> compute scriptPubKey via outer function
```

## Worked example

Wallet exports a Taproot multipath descriptor:

```
tr([d34db33f/86'/0'/0']xpub.../<0;1>/*)#abcdefgh
```

Expansion in Bitcoin Core (`listdescriptors`):

```json
[
  {
    "desc": "tr([d34db33f/86'/0'/0']xpub.../0/*)#aaaa1111",
    "active": true,
    "internal": false,
    "range": [0, 999]
  },
  {
    "desc": "tr([d34db33f/86'/0'/0']xpub.../1/*)#bbbb2222",
    "active": true,
    "internal": true,
    "range": [0, 999]
  }
]
```

`internal: true` marks the change chain - the wallet UI hides those
addresses from "next address" generation but still scans them for
incoming-from-self.

For 2-of-3 sortedmulti P2WSH:

```
wsh(sortedmulti(2,
  [d34db33f/48'/0'/0'/2']xpubA.../<0;1>/*,
  [b15ec0de/48'/0'/0'/2']xpubB.../<0;1>/*,
  [c0ffee00/48'/0'/0'/2']xpubC.../<0;1>/*))#chk
```

Each cosigner derives:

```
chain 0 (receive):
  pkA = xpubA/0/n  pkB = xpubB/0/n  pkC = xpubC/0/n
chain 1 (change):
  pkA' = xpubA/1/n  pkB' = xpubB/1/n  pkC' = xpubC/1/n
```

The sortedmulti reordering happens AFTER expansion at each `n`, so the
witness script can differ between receive and change at the same
index (rare, only when public keys cross a sort boundary).

Multipath with three or more chains (e.g., BIP329 label categories):

```
wpkh([fp/84'/0'/0']xpub.../<0;1;2>/*)
```

Chain 2 might be reserved for "labelled-only" or "swap" addresses.
Bitcoin Core supports any number of multipath elements.

## Common bugs / pitfalls

1. **Trying to use `*` inside the multipath.** `<0;*>` is INVALID;
   `*` is only allowed at the end. Multipath is for sibling-chain
   selection, not range expansion within a chain.
2. **Multiple multipath blocks.** `<0;1>/<0;1>/*` would imply 4 chains
   but BIP389 forbids this; wallets reject as syntax error.
3. **Hardened multipath.** Possible (`<0';1'>`) but only meaningful
   if you have an xprv. xpub-only wallets cannot derive hardened
   children, so multipath chains MUST be unhardened.
4. **Inconsistent chain count across cosigners.** All three
   cosigners' descriptors must agree on the multipath shape. If one
   wallet exported `<0;1>` and another `<0;1;2>`, Bitcoin Core treats
   them as different descriptors (different checksums) and rejects.
5. **Missing checksum after editing multipath.** Adding chain 2 to
   `<0;1>` -> `<0;1;2>` changes the checksum. Re-run
   `getdescriptorinfo` before importing.
6. **Watch-only re-import.** When re-importing a multipath descriptor
   into a fresh wallet, `next_index` resets to 0 for both chains.
   Specify `next_index` per chain via array: `[100, 50]` matches
   `<0;1>` order.

## References

- BIP389: https://github.com/bitcoin/bips/blob/master/bip-0389.mediawiki
- BIP380: https://github.com/bitcoin/bips/blob/master/bip-0380.mediawiki
- Bitcoin Core descriptors doc: https://github.com/bitcoin/bitcoin/blob/master/doc/descriptors.md
- bdk multipath support: https://docs.rs/bdk/latest/bdk/descriptor/
