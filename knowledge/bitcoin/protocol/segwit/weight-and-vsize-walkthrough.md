# Weight and Vsize - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/segwit`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/segwit/SKILL.md

## Concept

SegWit replaced the byte-based block size limit with a "weight" budget. The relationship between weight, virtual bytes (vsize), and on-the-wire bytes is the source of many off-by-one fee bugs and PSBT mis-estimations. This article gives the formula, the per-component breakdown, and the rounding rules - precisely.

## Walkthrough / mechanics

### Definitions

```
base_size      = serialized size of tx WITHOUT marker, flag, witness data
total_size     = serialized size of tx WITH marker, flag, witness data
witness_size   = total_size - base_size       (zero for non-segwit txs)

weight (WU)    = base_size * 3 + total_size
              == base_size * 4 + witness_size

vsize (vB)     = ceil(weight / 4)             (virtual bytes)
```

Equivalently: every non-witness byte costs 4 WU, every witness byte costs 1 WU. The 3:1 discount.

### Block budget

```
MAX_BLOCK_WEIGHT = 4_000_000 WU
```

Equivalent to 1 MB of non-witness bytes plus up to 3 MB of witness bytes.

### Component breakdown for typical inputs

| Spend | base bytes per input | witness bytes per input | weight contribution |
|-------|----------------------|--------------------------|---------------------|
| P2PKH | ~148 (script + ECDSA sig + pubkey) | 0 | 592 WU -> 148 vB |
| P2SH 2-of-3 | ~253 | 0 | 1012 WU -> 253 vB |
| P2WPKH | 41 (outpoint + 0-byte scriptSig + sequence) | ~108 (sig 72 + pubkey 33 + length bytes) | 41*4 + 108 = 272 WU -> 68 vB |
| P2WSH 2-of-3 | 41 | ~253 | 41*4 + 253 = 417 WU -> 105 vB |
| P2SH-P2WPKH | 64 (incl 23 bytes scriptSig push) | ~108 | 64*4 + 108 = 364 WU -> 91 vB |
| P2TR key-path (SIGHASH_DEFAULT) | 41 | 65 (varint+64 sig) | 41*4 + 65 = 229 WU -> 58 vB |
| P2TR script-path 2-of-2 (CHECKSIG/CHECKSIGADD) | 41 | ~155 (2 sigs + leaf + control) | ~319 WU -> 80 vB |

(The witness count includes the witness stack length varint and per-item length varints.)

### Why ceil for vsize

`weight / 4` may not be integer. Rounding up ensures a tx is never assigned less vsize than it occupies in the block budget. A 1-WU tx still costs 1 vB.

### Block weight accounting

For a block:
```
block_weight = sum( tx.weight for tx in block ) + 4 * len(serialized_block_header_and_tx_count)
```

Header is 80 bytes, tx count is 1-9 bytes; both go into the non-witness side and are multiplied by 4.

### Sigops weight (independent budget)

```
MAX_BLOCK_SIGOPS_COST = 80_000
```

Each CHECKSIG-class op in legacy/segwit-v0 counts 1; sigops cost is `count * 4` (matching the weight ratio). Tapscript does not count toward block sigops cost (it has its own per-input budget).

## Worked example

A 2-input, 2-output tx. Input 0 is P2WPKH, input 1 is P2TR key-path. Output 0 is P2WPKH, output 1 is P2TR.

Sizes:

```
nVersion           4
marker+flag        2 (witness only - counts toward total_size, not base_size)
vin_count          1
vin[0]:
  outpoint        36
  scriptSig len    1   (zero-length)
  scriptSig        0
  sequence         4
vin[1]:            same -> 41
vout_count         1
vout[0]: 8+1+22 = 31  (value + len + 22-byte P2WPKH spk)
vout[1]: 8+1+34 = 43  (value + len + 34-byte P2TR spk)
nLockTime          4
                   ----
base_size           = 4 + 1 + 41 + 41 + 1 + 31 + 43 + 4 = 166

witness:
  input 0 stack: 1 item count + (1 len + 72 sig) + (1 len + 33 pk) = 108
  input 1 stack: 1 item count + (1 len + 64 sig)                    =  66
  total witness                                                       = 174

total_size = 166 + 2 (marker+flag) + 174 = 342
weight     = 166 * 3 + 342 = 840
vsize      = ceil(840 / 4) = 210 vB
```

If the user pays 5 sat/vB, fee = 5 * 210 = 1050 sat.

## Common bugs / anti-patterns

- Confusing weight (weight units) with vsize (virtual bytes) in API calls. `getmempoolinfo` returns weights; `getmempoolentry` returns both.
- Computing `weight = total_size * 4` (forgetting marker/flag are witness-discounted? actually marker+flag are part of total_size; the formula above is correct because base_size omits them).
- Using `len(rawtx_hex) / 2` as size: that's total_size. For fee calculations, use vsize.
- Charging the user 4 sat/vB but submitting at 1 sat/WU - same number, different fee (4x).
- Forgetting that signatures vary in length (DER 71-72 bytes plus 1-byte sighash). Estimating with 71 once and a real sig comes out 72 -> low feerate.
- Missing the per-item witness length varint in size estimation (wallets often forget the leading "1" for stack count).
- For P2TR signature estimation: 64 with SIGHASH_DEFAULT, 65 with explicit hash type byte.
- Block template builders that maximize sat/B instead of sat/vB - they leave fees on the table by underweighting witness-heavy txs.

## References

- BIP141 weight: https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#witness-program
- Bitcoin Core `src/consensus/validation.h` - `GetTransactionWeight`
- Optech weight estimator: https://bitcoinops.org/en/tools/calc-size/
