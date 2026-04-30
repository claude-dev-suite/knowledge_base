# OP_RETURN Conventions - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/scripts`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/src/policy/policy.cpp (IsStandardTx, datacarrier)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/scripts/SKILL.md

## Concept

`OP_RETURN` defines an unspendable output used to embed application data in the chain without bloating the UTXO set. It is the canonical way to anchor commitments, OP_NULLDATA layered protocols (Omni, RGB shadow, OpenTimestamps, Stamps), the witness-merkle commitment, and many sidechain peg messages. There is a small surface of policy and consensus rules to know - getting them wrong gets your tx silently dropped.

## Walkthrough / mechanics

### Script form

```
scriptPubKey = OP_RETURN <push payload>
```

`OP_RETURN` (0x6a) immediately fails the script when executed (this is what makes the output unspendable). The push that follows is technically dead code; nodes still parse and serialize it.

For the output to be **standard** ("nulldata" type, IsStandard true):

- Exactly one `OP_RETURN` opcode.
- Followed by exactly one or more push opcodes (no other op).
- Total `scriptPubKey` size including OP_RETURN: `<= 83` bytes by default (matches `OP_RETURN <0x4c 0x50 ...80 bytes>` = 1 + 1 + 1 + 80 = 83).
- `nMaxDatacarrierBytes = 80`: the configured max payload (raised from 40 in 0.12).
- At most **one** OP_RETURN output per transaction (`fAcceptDatacarrier`).

Consensus has no per-output OP_RETURN limit other than the general 10_000-byte scriptPubKey ceiling and tx weight.

### Push encoding

For a payload of length L:

| L | Encoding |
|---|----------|
| 0 | `OP_0` (0x00) |
| 1..75 | `<L>` `<bytes>` |
| 76..255 | `OP_PUSHDATA1` `<L>` `<bytes>` |
| 256..65535 | `OP_PUSHDATA2` `<L_le16>` `<bytes>` |
| 65536..2^32-1 | `OP_PUSHDATA4` `<L_le32>` `<bytes>` |

For 80-byte standard payloads: `6a 4c 50 <80 bytes>`.

### Common protocol prefixes (de facto conventions)

| Prefix (hex) | Protocol | Purpose |
|--------------|----------|---------|
| `aa21a9ed` | BIP141 witness commitment (in coinbase only) | wtxid merkle root commitment |
| `4f41` ("OA") | OpenAssets | colored coins, deprecated |
| `6f6d6e69` ("omni") | Omni Layer | Tether USD originally, Counterparty kin |
| `4f4f4f4f` ("OOOO") | OpenTimestamps | ots commitments |
| `5354414d50` ("STAMP") | Counterparty Stamps | NFT-like |
| `df` | Stacks chain anchor | sBTC peg |

### Consensus role - the witness commitment

In a SegWit block, the coinbase tx must contain (anywhere in vout) at least one output of the form:

```
OP_RETURN OP_PUSHDATA1 0x24 0xaa21a9ed <32-byte commitment>
```

i.e. 38 bytes total. If multiple such OP_RETURNs exist, the last one is used. Non-segwit blocks may omit it. This is the only OP_RETURN with consensus meaning.

### Standardness drift

Released changes:
- 0.9: OP_RETURN became standard (for the first time), 40-byte limit.
- 0.11: bumped to 80 bytes.
- 0.12: optional `-datacarriersize` for users.
- 0.21: `-datacarrier` default still true; rules unchanged.
- 25.x+: discussion of removing the size limit (e.g. PR 32359 to remove `nMaxDatacarrierBytes`); consensus has no limit, only relay.
- Knots / runes / inscriptions: alternative data carriers using witness data, not OP_RETURN.

## Worked example

OpenTimestamps commitment (37-byte payload):

```
ots payload = 0x4f4f4f4f04 || <32-byte sha256>            length = 37
scriptPubKey = 6a 25 <37 bytes payload>
              = OP_RETURN PUSH-37 <payload>
output value = 0  (allowed for OP_RETURN; no dust threshold)
```

Constructing in Python (python-bitcoinlib):

```python
from bitcoin.core.script import CScript, OP_RETURN
payload = b"\x4f\x4f\x4f\x4f\x04" + sha256(message)
spk = CScript([OP_RETURN, payload])
out = CTxOut(0, spk)
```

Decoding from rawtx hex `6a 4c 50 <80 bytes>`:

```python
ops = list(CScript(spk_bytes))
assert ops[0] == OP_RETURN
data = ops[1]    # bytes
```

## Common bugs / anti-patterns

- Setting `output.value > 0` and treating it as recoverable - those sats are burned. Use 0 unless you have a reason (some chains/ tools require >= dust due to bugs).
- Two OP_RETURNs in one tx: tx is non-standard, not relayed.
- Payload of 81 bytes: total scriptPubKey 84 bytes, exceeds 83-byte cap, non-standard.
- Encoding the push as raw bytes without an OP_PUSHDATA opcode: `6a <data>` directly is invalid script; you need `6a 4c 50 <data>` for 80 bytes.
- Putting OP_RETURN as the first output in a tx with SIGHASH_SINGLE on input 0 - signature commits to OP_RETURN as "the" output.
- Treating witness commitment OP_RETURN as user data when parsing coinbase txs - filter on the `aa21a9ed` magic prefix.
- Forgetting to pad short payloads when expecting fixed 32-byte hash on the consumer side: 6a 20 <hash> is correct, NOT 6a 4c 20 <hash> (wastes a byte).

## References

- Bitcoin Core: `src/policy/policy.cpp` (`IsStandardTx`, `nMaxDatacarrierBytes`), `src/script/standard.cpp` (`Solver` / `TxoutType::NULL_DATA`)
- BIP141 (commitment OP_RETURN): https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki
- OpenTimestamps spec: https://github.com/opentimestamps/python-opentimestamps
- Mailing list discussion 2024: https://gnusha.org/pi/bitcoindev/?q=op_return
