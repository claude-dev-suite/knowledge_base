# Rune Etching Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/metaprotocols/runes`.
> Canonical source: https://docs.ordinals.com/runes/specification.html#etching
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/metaprotocols/runes/SKILL.md

## Concept

Etching is the one-time issuance of a new Rune. It assigns the rune
a globally unique ID `(block, tx_index)`, registers a human-readable
name, and optionally configures a public mint with caps and time
windows. Like inscriptions, etching uses a commit-reveal pattern -
not for technical reasons but for **anti-front-running**: the chosen
name is committed inside a Tapscript pre-image whose hash appears in
a commit tx, then revealed in the etching tx after a 6-block delay.
Without this delay, miners could see your unconfirmed etching, etch
the same name first, and steal it.

## Walkthrough / mechanics

```
1. Choose name N. Compute commit script:
      <name_bytes>           (push of the encoded name)
      OP_DROP
      <pubkey> OP_CHECKSIG
2. Build a Taproot output T1 committing to that script.
3. Broadcast commit tx; wait 6 confirmations.
4. Build the etching tx:
      input  : T1 spent via script-path (witness reveals the name)
      output0: OP_RETURN runestone with etching tag + name
      output1: P2TR or other = where the premine balance lands
5. Broadcast etching tx. Indexer sees:
      - input revealed name N at script position
      - runestone declares etching of name N
      - they match
      - rune ID assigned: (this_block, tx_index_in_block)
```

The 6-block delay rule: ord rejects an etching whose commit input
has fewer than 6 confirmations as a cenotaph. This makes name-
sniping require a 6-block reorg.

Mint window controls allow:

- absolute height range: `height_start..height_end`
- relative offset (counted from etching block): `offset_start..offset_end`
- mint cap: max number of mint events
- per-mint amount: how much each mint produces
- premine: amount minted to the etcher up front

A rune without `terms` (no public mint configured) is "etcher-only":
only the etcher's premine ever exists.

## Worked example  (encoded data, real txs, configs)

Etch `RSIC.GENESIS.RUNE` with 21_000_000 cap, 1_000 per mint,
divisibility 0, premine 1_000_000 to etcher, mintable for 1 year:

Commit script (pushed to Taproot leaf):

```
PUSH 17 bytes "RSIC.GENESIS.RUNE"  (encoded name)
OP_DROP
PUSH 32 bytes <etcher_xonly_pk>
OP_CHECKSIG
```

Etching runestone payload:

```
flags        = 0b011  (etching | terms)
rune         = <encoded RSICGENESISRUNE>
spacers      = 0b00010100  (dots after RSIC and GENESIS)
divisibility = 0
symbol       = 0x24 ($)
premine      = 1_000_000
amount       = 1_000
cap          = 21_000  (premine + 21_000 * 1_000 = 22M; some reach 21M only via mints)
height_start = 840_000
height_end   = 893_000  (~ 1 year later)
pointer      = 1        (premine goes to output 1)
```

Encoded as bytes (snippet):

```
02 03                       ; flags
04 <encoded name varint>    ; rune name
1a 14                       ; spacers = 0b00010100
18 00                       ; divisibility 0
1c 24                       ; symbol '$'
06 c0 84 3d                 ; premine 1_000_000 (LEB128)
0a e8 07                    ; amount 1_000
08 a8 a4 01                 ; cap 21_000
0c c0 e4 33                 ; height_start 840_000
0e c8 b6 36                 ; height_end 893_000
16 01                       ; pointer 1
00                          ; body terminator, no edicts
```

Etching tx fee at 100 sat/vB (peak halving fees): ~25_000 sats for
the runestone + ~10_000 for the input/witness = 35_000 sats. The
real RSIC etching at block 840_000 paid considerably more due to
mempool congestion.

## Trade-offs / pitfalls

- **Name griefing**: an attacker who sees your commit can race a
  competing commit and try to confirm theirs first. The 6-block
  delay only protects after commit confirmation; before that a
  miner-attacker can include their own etch in the same block and
  win on tx-index ordering. Only commit on a low-fee block to lower
  the attack incentive, or use private relays.
- **Reserved names**: numerical thresholds reserve "valuable" short
  names until far in the future. An etch that uses a reserved name
  becomes a cenotaph - the bytes still appear on chain but the
  registration fails.
- **Forgetting pointer**: premine flows to the first non-OP_RETURN
  output by default. If your tx puts change first by accident, your
  premine lands at the change address and the intended recipient
  gets nothing.
- **Cap arithmetic**: `cap` is the number of *mint events*, not the
  total amount. Total mintable supply is `premine + cap * amount`.
- **Activation height**: any etching before block 840_000 is invalid
  (cenotaph). Sniping attempts before activation produced a long
  list of cenotaphs in early April 2024.
- **Name length**: 1..28 chars after spacer expansion. Encoded
  varint must fit; longer names cenotaph.
- **Recovery**: if you lose the commit-spending key after broadcast,
  the etching cannot be revealed. The name is then unetched and
  available again after the commit output ages out of mempool/UTXO
  set.

## References

- Runes etching: https://docs.ordinals.com/runes/specification.html#etching
- Reserved names: https://docs.ordinals.com/runes/specification.html#reserved
- ord runes test vectors: https://github.com/ordinals/ord/tree/master/tests
