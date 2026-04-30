# Runestone Encoding - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/metaprotocols/runes`.
> Canonical source: https://docs.ordinals.com/runes/specification.html
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/metaprotocols/runes/SKILL.md

## Concept

A runestone is a Bitcoin OP_RETURN output whose payload encodes Rune
protocol messages. Unlike BRC-20 (free-form JSON in inscriptions),
the runestone is a tightly packed binary structure of varints and
tag-value pairs. Validation rules are baked into the protocol itself:
indexers all parse the same bytes the same way, eliminating the
indexer-disagreement risk of BRC-20. The OP_RETURN sits on the
witnessless side of the tx so it occupies full vBytes - the encoding
is therefore as compact as possible.

## Walkthrough / mechanics

Marker prefix:

```
OP_RETURN OP_PUSHNUM_13 <payload>
       0x6a       0x5d  ...
```

`OP_PUSHNUM_13` (0x5d) is the Rune protocol marker. Any other prefix
is non-Rune. Payload is one or more pushdata pushes concatenated.

Body is a stream of integers in LEB128 varint form. Tags are even
integers (so unknown odd tags can be ignored = forward-compatible).
Each tag is followed by its value. Multiple values for one tag (e.g.
edicts) repeat the tag.

Defined tags:

```
0   body          terminates header; subsequent ints are edicts (4-int tuples)
2   flags         bitfield (etching, terms, turbo, ...)
4   rune          rune name as integer
6   premine       amount minted to etcher
8   cap           total max mints (number of mint events, not amount)
10  amount        per-mint amount
12  height_start  height range begin
14  height_end    height range end
16  offset_start  relative-time begin
18  offset_end    relative-time end
20  mint          rune ID (block.tx) of the rune being minted
22  pointer       output index for default rune balance
24  divisibility  decimals for display
26  spacers       bitmask of dot positions for visual spacing
28  symbol        unicode codepoint
```

After the header (everything before tag 0), any further integers
form **edicts**. Each edict is exactly 4 ints:

```
(rune_id_block, rune_id_tx, amount, output_index)
```

Special output index 0 = "all unallocated", N = output index N
(0-indexed where output 0 is usually OP_RETURN itself).

## Worked example  (encoded data, real txs, configs)

Etch a rune named "DOG-GO-TO-THE-MOON" with 100B supply, divisibility
5, premine 100B, no further mints. The name encodes as
base-26-ish integer; the spec's encoding maps A=0, B=1, ..., Z=25
with leading-A handling.

Header tags:

```
flags         = 1  (etching)
rune          = <encoded "DOGGOTOTHEMOON">
divisibility  = 5
premine       = 100_000_000_000 * 10^5 = 1e16
spacers       = 0b1010101 (dots between syllables)
symbol        = 0x1F415 (dog emoji)
```

Bytes (illustrative):

```
6a 5d <push>
   02 01                        ; tag 2 (flags)  = 1
   04 <varint encoded name>     ; tag 4 (rune name)
   18 05                        ; tag 24 (divisibility) = 5
   06 <varint 1e16>             ; tag 6 (premine)
   1a 55                        ; tag 26 (spacers) = 0b1010101 = 85 (0x55)
   1c 95 e8 07                  ; tag 28 (symbol) varint of 0x1F415
   00                           ; tag 0 = body terminator (no edicts here)
```

Mint of an existing rune `840000:1`:

```
6a 5d <push>
   14 <varint 840000>           ; tag 20 hi part
   14 <varint 1>                ; tag 20 lo part (mint takes 2 ints)
   00                           ; body, no edicts
```

Transfer 1000 of rune 840000:1 to output 1, rest to pointer:

```
6a 5d <push>
   16 01                        ; tag 22 (pointer) = output 1
   00                           ; body
   <varint 840000> <varint 1> <varint 1000> <varint 1>   ; one edict
```

A real example: rune `UNCOMMON.GOODS` etching at block 840_000 tx 1
has 32-byte payload, fee ~5_000 sats at 50 sat/vB.

## Trade-offs / pitfalls

- **Cenotaph**: any malformed runestone (unknown even tag, truncated
  varint, unallocated edicts) is treated as a "cenotaph". Cenotaphs
  burn all rune balances flowing through that tx. Encoder bugs are
  catastrophic - test against the official test vectors.
- **Pointer default**: if pointer is omitted and edicts do not
  account for all balance, leftover balance flows to the first non-
  OP_RETURN output. If your tx has only one non-OP_RETURN output it
  silently absorbs everything.
- **Output index of OP_RETURN**: output indices are zero-based and
  include the OP_RETURN itself. An edict to output 0 sends to the
  OP_RETURN, which burns the balance.
- **Multiple OP_RETURNs**: only the first OP_RETURN that starts with
  `OP_PUSHNUM_13` is interpreted as a runestone. Others are ignored.
- **Reserved name protection**: until block 840_000 + 17_500 *
  10_080 (~year 2099), names below a certain integer threshold are
  reserved and any etch that uses one creates a cenotaph.
- **Divisibility max**: 38. Larger values are cenotaph.
- **Amount overflow**: amounts are u128. Sums during processing can
  saturate; spec requires saturating semantics, not wrapping.

## References

- Runes spec: https://docs.ordinals.com/runes/specification.html
- ord runestone parser: https://github.com/ordinals/ord/blob/master/crates/ordinals/src/runestone.rs
- LEB128 varint: https://docs.ordinals.com/runes/varint.html
