# Mocktime and Block Control - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/testing/regtest`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/src/rpc/misc.cpp (setmocktime)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/testing/regtest/SKILL.md

## Concept

Many Bitcoin behaviours are time-driven: locktime, sequence (BIP68
relative timelocks), median-time-past for CSV, mempool eviction
(`-mempoolexpiry`), wallet rebroadcast intervals, and Lightning
HTLC timeouts. Wall-clock-driven tests are slow and flaky.

`setmocktime` freezes the node's clock at a chosen Unix timestamp. The
node uses that as `GetTime()` everywhere - block headers,
mempool timestamps, peer last-seen. Combined with deterministic block
mining via `generatetoaddress`, you can advance through years of
simulated chain time in milliseconds.

## Walkthrough / mechanics

Two RPCs cooperate:

- `setmocktime <unix_ts>`: forces the node clock to `unix_ts`. Pass
  `0` to release. Affects only the node it is called on.
- `generatetoaddress n addr`: mines `n` blocks instantly. Each block's
  `nTime` is `max(GetMedianTimePast()+1, GetTime())`, so blocks made
  while mocktime is set get the mocked timestamp (or strictly greater).

Median-time-past is computed over the last 11 blocks. To make BIP68
sequence locks expire, you must mine 11 blocks past the desired
timestamp - mocktime alone is not enough.

`-mocktime=<ts>` is a `bitcoind` CLI flag equivalent to setting
mocktime at startup. Useful when a daemon-level cache reads the clock
before the first RPC.

For unit-style time math, `setmocktime` + a single
`generatetoaddress 1` block bumps both `nTime` and MTP correctly.

## Worked example

Test that an OP_CHECKLOCKTIMEVERIFY transaction becomes spendable
after a one-year jump in chain time:

```python
import time
from test_framework.test_framework import BitcoinTestFramework

class CltvTimeTest(BitcoinTestFramework):
    def set_test_params(self):
        self.num_nodes = 1
        self.setup_clean_chain = True
        self.extra_args = [["-fallbackfee=0.00001"]]

    def run_test(self):
        node = self.nodes[0]
        start = 1_700_000_000   # any sane epoch
        node.setmocktime(start)

        addr = node.getnewaddress()
        self.generate(node, 101)
        # CLTV-locked output: spendable only after start + 1 year
        cltv_time = start + 365 * 24 * 3600
        # ... build raw tx with CLTV ...

        # Try to broadcast: rejected (locktime in future)
        # Now jump: bump mocktime + mine 12 blocks (>11 for MTP)
        node.setmocktime(cltv_time + 1)
        self.generate(node, 12)
        assert node.getblockchaininfo()["mediantime"] > cltv_time
        # Broadcast now succeeds.
```

CLI version of the same trick:

```bash
bitcoin-cli -regtest setmocktime 1700000000
ADDR=$(bitcoin-cli -regtest -rpcwallet=w getnewaddress)
bitcoin-cli -regtest -rpcwallet=w generatetoaddress 101 $ADDR

# Inspect block times
bitcoin-cli -regtest getblockheader $(bitcoin-cli -regtest getbestblockhash) | jq .time
# 1700000000

# Jump 24h, mine 12 blocks so MTP advances
bitcoin-cli -regtest setmocktime 1700086400
bitcoin-cli -regtest -rpcwallet=w generatetoaddress 12 $ADDR
bitcoin-cli -regtest getblockchaininfo | jq .mediantime
```

For BIP68 (relative timelock by time):

```python
node.setmocktime(start)
self.generate(node, 101)
# Send to a CSV-locked script with sequence = 0x40000000 | seconds_locked
# Then advance:
node.setmocktime(start + 4096)   # CSV time granularity = 512s; pick >locked
self.generate(node, 12)          # MTP must cross the threshold
```

## Common pitfalls

- Setting mocktime alone does not change MTP. You must mine at least 11
  more blocks for `getblockchaininfo.mediantime` to catch up.
- Mocktime is per-node. In multi-node tests, call `setmocktime` on each
  node or sync drifts.
- The wallet's `bestblockhash` is updated only on block-connect events;
  cached values can lag if you only set mocktime without mining.
- `setmocktime(0)` re-enables real time. If you forget to clear it,
  later RPCs that rely on real time (e.g. p2p ping ages) become wrong.
- If the node is restarted without `-mocktime=`, mocktime is lost.

## References

- Bitcoin Core source: `src/rpc/misc.cpp` `setmocktime`
- BIP68: `https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki`
- BIP65 (CLTV): `https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki`
- `test/functional/feature_cltv.py`, `feature_csv_activation.py`
