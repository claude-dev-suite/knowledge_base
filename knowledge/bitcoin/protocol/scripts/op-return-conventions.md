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

- Leading `OP_RETURN` opcode.
- Everything after it push-only (`CScript::IsPushOnly`): any opcode <= `OP_16`, i.e. data pushes, the numeric constants `OP_1NEGATE`/`OP_0`/`OP_1`..`OP_16`, and `OP_RESERVED`. Any number of pushes, not just one (since 0.12).
- `-datacarriersize` caps the **aggregate** `scriptPubKey` size of all nulldata outputs in the transaction (the scriptPubKey's own length prefix is not counted). Default is `100000` since Bitcoin Core 30.0 (2025-10-10) - `MAX_STANDARD_TX_WEIGHT / WITNESS_SCALE_FACTOR`, i.e. effectively uncapped, because the 400_000-weight standard tx limit binds first.
- Any number of OP_RETURN outputs per transaction, also since 30.0. `-datacarrier=0` still disables nulldata relay outright.

The pre-30.0 shape - one nulldata output, `scriptPubKey <= 83` bytes total (`OP_RETURN OP_PUSHDATA1 0x50 <80 bytes>` = 1 + 1 + 1 + 80) - is what Bitcoin Knots gives you by default; on Core, `-datacarriersize=83` restores only the 83-byte cap, now applied to the aggregate across all nulldata outputs, with no per-output count check anywhere in policy. See "Node-software divergence" below. Checked against `src/policy/policy.{h,cpp}` and `src/script/solver.cpp` on Bitcoin Core master, September 2026.

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

For an 80-byte payload (the pre-30.0 standard maximum): `6a 4c 50 <80 bytes>`.

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
- 0.10: `-datacarrier` and `-datacarriersize` added, making it node policy users can configure.
- 0.11: default bumped to 80 bytes (PR #5286).
- 0.12: any sequence of pushdatas allowed after `OP_RETURN` (previously a single pushdata); the limit moved to the whole serialized `scriptPubKey`, 83 bytes by default (the 80-byte payload plus three bytes overhead).
- 0.21: `-datacarrier` default still true; rules unchanged.
- 30.0 (2025-10-10): `-datacarriersize` default raised to 100,000 and multiple OP_RETURN outputs permitted for relay and mining, the limit applying to their aggregate `scriptPubKey` size (PR #32406, merged 2025-06-09). `-datacarriersize=83` reverts to the previous limit. `getmempoolinfo` gained a `maxdatacarriersize` field (PR #29954). The earlier PR #32359 ("Remove arbitrary limits on OP_Return (datacarrier) outputs") was closed without merging. Consensus was never involved - this is relay/mining policy throughout.
- Knots / runes / inscriptions: alternative data carriers using witness data, not OP_RETURN.

### Node-software divergence (as of September 2026)

Since 30.0 the defaults are no longer uniform across the network, so "will it relay?" depends on what your peers run:

| Policy | Bitcoin Core 30.0+ | Bitcoin Knots v29.4.1.knots20260508 |
|--------|--------------------|-------------------------------------|
| `-datacarriersize` default | 100000, aggregate over all nulldata outputs | 83 |
| OP_RETURN outputs per tx | unlimited | 1 (`multi-op-return` reject) |
| Tx with only datacarrier outputs | relayed | rejected unless `-permitbaredatacarrier=1` (default off) |
| Known token protocols | relayed | runes / Counterparty / OLGA rejected by default (`-rejecttokens`) |
| Extra data weighting | witness bytes weigh 0.25 vbyte | `-datacarriercost` charges data at >= 1 vbyte per byte by default |

Knots itself raised its `-datacarriersize` default to 83 in v29.2.knots20251110 (2025-11-10), describing it as temporary - "some legacy protocols still rely on 83-byte datacarrier outputs" - and to be "reverted back to 42 in a future version".

Current release lines: Bitcoin Core 31.1 (2026-07-08), Bitcoin Knots v29.4.1.knots20260508 (2026-09-02). Knots' share of reachable nodes is material but contested: coin.dance showed 4,399 Knots against 21,431 Core of 25,864 public nodes (~17%) on 2026-09-15, and reachable-node counts are a sample, not the network.

Practical upshot: treat >83 bytes or >1 nulldata output as best-effort. It is standard for a Core 30.0+ node, but a minority of listening peers and some miners will not relay or mine it, so confirmation may need direct submission to a pool.

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
- Assuming the pre-30.0 limits still bind everywhere, or assuming they are gone everywhere. An 81-byte payload (84-byte scriptPubKey) is standard on Bitcoin Core 30.0+ but rejected by Knots and by any node run with `-datacarriersize=83`. Multiple nulldata outputs are standard on Core 30.0+ whatever its `-datacarriersize` - two of them still relay on an `=83` node as long as their scriptPubKeys total <= 83 bytes, since only the aggregate is capped - but Knots rejects any tx carrying more than one (`multi-op-return`) regardless of size. Check `getmempoolinfo.maxdatacarriersize` on the node you are actually broadcasting through.
- Encoding the push as raw bytes without an OP_PUSHDATA opcode: `6a <data>` directly is invalid script; you need `6a 4c 50 <data>` for 80 bytes.
- Putting OP_RETURN as the first output in a tx with SIGHASH_SINGLE on input 0 - signature commits to OP_RETURN as "the" output.
- Treating witness commitment OP_RETURN as user data when parsing coinbase txs - filter on the `aa21a9ed` magic prefix.
- Forgetting to pad short payloads when expecting fixed 32-byte hash on the consumer side: 6a 20 <hash> is correct, NOT 6a 4c 20 <hash> (wastes a byte).

## References

- Bitcoin Core: `src/policy/policy.cpp` (`IsStandardTx`), `src/policy/policy.h` (`MAX_OP_RETURN_RELAY`), `src/script/solver.cpp` (`Solver` / `TxoutType::NULL_DATA`)
- Bitcoin Core 30.0 release notes (2025-10-10): https://bitcoincore.org/en/releases/30.0/
- Bitcoin Knots releases: https://github.com/bitcoinknots/bitcoin/releases
- BIP141 (commitment OP_RETURN): https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki
- OpenTimestamps spec: https://github.com/opentimestamps/python-opentimestamps
- Mailing list discussion 2024: https://gnusha.org/pi/bitcoindev/?q=op_return
