# CLTV vs CSV - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/timelocks`.
> Canonical sources: BIP65, BIP68, BIP112
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/timelocks/SKILL.md

## Concept

CLTV (CheckLockTimeVerify) and CSV (CheckSequenceVerify) are the two
script-level timelock primitives. They look superficially similar but
encode fundamentally different temporal references:

- **CLTV** asks: "Has the absolute deadline (block height or unix
  timestamp) passed?" It compares against `tx.nLockTime`.
- **CSV** asks: "Has enough time elapsed since *the parent UTXO* was
  mined?" It compares against the input's `nSequence`.

The wrong choice silently breaks the security property a vault or HTLC
intends to enforce. Picking the right one is mostly a question of
*relative to what?*.

## Walkthrough / mechanics

### CLTV (BIP65)

```
<TARGET> OP_CHECKLOCKTIMEVERIFY
```

Behaviour:
- If `TARGET < 500_000_000` → interpret as block height.
- If `TARGET >= 500_000_000` → interpret as unix timestamp.
- The script aborts unless `tx.nLockTime` >= `TARGET` AND
  `tx.nLockTime` matches the same height/time domain.
- For `nLockTime` to be enforced, **at least one input must have
  `nSequence < 0xffffffff`** (and `< 0xfffffffe` to also be RBF-eligible).
- Time comparisons use **median time past** (BIP113), not the current
  block timestamp.

```
TARGET = 750_000        -> "valid only at block height >= 750000"
TARGET = 1_700_000_000  -> "valid only at unix time >= 2023-11-15"
```

The opcode does **not** pop its argument; an `OP_DROP` traditionally
follows.

### CSV (BIP112)

```
<DELTA> OP_CHECKSEQUENCEVERIFY
```

Behaviour:
- Operates on the *current input*'s `nSequence` field.
- If bit 22 (`0x00400000`) is set: low 16 bits × 512s = relative
  seconds.
- Otherwise: low 16 bits = relative blocks.
- Bit 31 (`0x80000000`) being set on input nSequence → relative locks
  disabled.
- The transaction's `tx.version` must be >= 2 (BIP68) or relative
  locks are silently ignored.

```
DELTA = 144            -> "input must be 144+ blocks deep"
DELTA = 0x00400048     -> "input must be ~10 hours old (72 * 512s)"
```

CSV is enforced by comparing the input's `nSequence` field to `DELTA`,
matching the bit-encoding domain.

### Why two opcodes?

CLTV is for **calendar deadlines**: "this payment is refundable after
2025-12-31"; "this contract expires at block 1 000 000". The deadline is
not relative to the funding tx — it's a wall-clock event.

CSV is for **cooldowns**: "wait 1 day after receiving this UTXO before
spending it"; "channel commitment delay starts when the channel closes."
The delay starts whenever the parent UTXO is mined, which the script
doesn't need to know in advance.

## Worked example

### CLTV: scheduled inheritance payment

Want a tx that becomes spendable on block 900_000:

```
script_pubkey:
   <900000> OP_CHECKLOCKTIMEVERIFY OP_DROP
   <heir_pubkey> OP_CHECKSIG

Spending tx:
   nLockTime = 900000
   inputs[0].nSequence = 0xfffffffd   (< 0xffffffff so nLockTime is honored)
   tx.version = 2  (any value >=1 OK; BIP65 only needs nSequence)
```

If you forget `nLockTime` (or set it to 0), the script aborts at CLTV
because `0 < 900000`. If you forget `nSequence < 0xffffffff` on at least
one input, `nLockTime` is *ignored entirely* and the script's CLTV
check happens against `nLockTime = 0` — also aborts.

### CSV: 1-day vault cooldown

Want a tx that can spend a vault output 144 blocks after the vault
funding tx confirms:

```
script_pubkey:
   <144> OP_CHECKSEQUENCEVERIFY OP_DROP
   <hot_pubkey> OP_CHECKSIG

Spending tx:
   tx.version = 2
   inputs[0].nSequence = 144   (low 16 = 144, bit 31 = 0)
   nLockTime = 0  (irrelevant for CSV)
```

If you set `tx.version = 1`, BIP68 is not engaged; the CSV check is a
no-op and the spend works immediately — silently breaking the vault.

## Combined use: HTLC

Lightning HTLC scripts combine both:

```
OP_IF
   OP_HASH160 <preimage_hash> OP_EQUALVERIFY
   <recipient_pk> OP_CHECKSIG
OP_ELSE
   <timeout> OP_CHECKLOCKTIMEVERIFY OP_DROP
   <sender_pk> OP_CHECKSIG
OP_ENDIF
```

Recipient claims with preimage (no timelock). Sender refunds via the
ELSE branch after the absolute timeout. CLTV here represents a hard
calendar deadline ("refund unlocks at block 850 000"), so CLTV is the
right choice. Some channel commitment outputs *also* add a CSV cooldown
on top of the recipient branch — using both opcodes in one taptree is
not unusual.

## Common pitfalls

- **Forgetting `tx.version = 2`** for CSV. The most common silent
  bug. Always set version 2.
- **`nSequence = 0xffffffff`** on the input that needs CSV — disables
  BIP68 entirely.
- **Using CLTV for relative time** by computing `current_height + 144`
  at signing. Works once, then breaks the next time the wallet
  rebuilds the tx (because `current_height` advanced).
- **Mixing block-height and unix-time domains in CLTV**: passing
  `nLockTime = 750000` (height) but `TARGET = 1_700_000_000` (time)
  aborts because BIP65 enforces same-domain comparison.
- **MTP gotcha**: using "now" as the unix timestamp is fine only if
  you account for MTP lag. The current MTP can be ~1 hour behind real
  time on testnets.
- **CSV with bit 22 set but value > 65535**: the script encoding of
  CSV requires the value to fit in the low 16 bits when bit 22 is
  set; larger values overflow into the bit positions that change
  semantics.

## References

- BIP65 (CLTV): https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki
- BIP68 (Sequence): https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki
- BIP112 (CSV): https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki
- BIP113 (MTP): https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki
- See also: [mtp-rule-bip113.md](mtp-rule-bip113.md), [../vaults/single-sig-csv-walkthrough.md](../vaults/single-sig-csv-walkthrough.md)
