# Multi-Node Regtest Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/testing/regtest`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/doc/REST-interface.md and https://en.bitcoin.it/wiki/Regression_test_mode
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/testing/regtest/SKILL.md

## Concept

A real production network has dozens of peers, propagation delays, mempool
divergence, and reorgs. A single-node regtest can never reproduce these
behaviours. A multi-node regtest topology makes those phenomena
deterministic and on-demand: open and close peer links with `-connect` /
`-addnode` / `disconnectnode`, partition the network with
`setnetworkactive false`, and force chain forks by mining on each side.

The pattern is one `bitcoind` process per logical peer, each with its own
`-datadir`, `-port` (P2P), and `-rpcport` (RPC). Coordination happens via
RPC; you never share data directories.

## Walkthrough / mechanics

The minimum a node needs to be isolated:

1. Distinct `-datadir`. Without this, two nodes share the same `wallet.dat`
   and chain state and corrupt each other.
2. Distinct `-port` (P2P listen port).
3. Distinct `-rpcport` plus `-rpcuser` / `-rpcpassword` (or shared cookie).
4. `-fallbackfee` set so `sendtoaddress` works on an empty mempool.
5. `-connect=<host:port>` (outbound only, disables DNS seeds and addrman)
   or `-addnode=` (outbound but keeps addrman).

Three standard topologies:

- Chain: n1 - n2 - n3. Tests propagation latency.
- Star: n1 in the centre, n2/n3/n4 connecting to it. Tests rebroadcast.
- Ring: every node connects to every other. Default for sync_blocks tests.

Force a partition by calling `disconnectnode "127.0.0.1:18444"` on each
side, mine independently, then re-connect to trigger a reorg.

## Worked example

Three nodes, each in their own datadir:

```bash
mkdir -p /tmp/n1 /tmp/n2 /tmp/n3

bitcoind -regtest -datadir=/tmp/n1 -port=18444 -rpcport=18443 \
  -fallbackfee=0.00001 -daemon

bitcoind -regtest -datadir=/tmp/n2 -port=18454 -rpcport=18453 \
  -fallbackfee=0.00001 -connect=127.0.0.1:18444 -daemon

bitcoind -regtest -datadir=/tmp/n3 -port=18464 -rpcport=18463 \
  -fallbackfee=0.00001 -connect=127.0.0.1:18444 -daemon

alias n1='bitcoin-cli -regtest -datadir=/tmp/n1'
alias n2='bitcoin-cli -regtest -datadir=/tmp/n2'
alias n3='bitcoin-cli -regtest -datadir=/tmp/n3'

n1 createwallet w1
n2 createwallet w2

ADDR1=$(n1 -rpcwallet=w1 getnewaddress)
n1 -rpcwallet=w1 generatetoaddress 101 $ADDR1

# Wait for sync
n2 getblockcount   # eventually 101

# Test a partition + reorg
n1 disconnectnode "127.0.0.1:18454"
n1 -rpcwallet=w1 generatetoaddress 5 $ADDR1   # n1 chain: 106
ADDR2=$(n2 -rpcwallet=w2 getnewaddress)
n2 generatetoaddress 7 $ADDR2                 # n2 chain: 108 on a fork

# Reconnect; n1 reorgs to n2's longer chain
n1 addnode "127.0.0.1:18454" onetry
sleep 2
n1 getblockcount   # 108
```

Or via the Bitcoin Core test framework:

```python
class ReorgTest(BitcoinTestFramework):
    def set_test_params(self):
        self.num_nodes = 3
        self.extra_args = [["-fallbackfee=0.00001"]] * 3

    def run_test(self):
        self.generate(self.nodes[0], 101)
        self.disconnect_nodes(0, 1)
        self.generate(self.nodes[0], 5, sync_fun=self.no_op)
        self.generate(self.nodes[1], 7, sync_fun=self.no_op)
        self.connect_nodes(0, 1)
        self.sync_blocks([self.nodes[0], self.nodes[1]])
        assert_equal(self.nodes[0].getblockcount(), 108)
```

## Common pitfalls

- Sharing `-datadir` across processes corrupts the LevelDB chainstate
  silently and produces ghost reorgs on next start.
- `-connect=` disables peer discovery; if you want both manual peers and
  addrman discovery use `-addnode=` instead.
- `generatetoaddress` only mines on the node it is called on. To advance
  the *common* chain you must call `sync_blocks` afterwards.
- Forgetting `-fallbackfee` on regtest: `sendtoaddress` fails with
  "Fee estimation failed".
- Default ports overlap mainnet/testnet ranges. On Linux pick something
  >18500 to avoid clashes with running services.

## References

- bitcoin/bitcoin `test/functional/test_framework/test_framework.py`
- `doc/regtest.md` in Bitcoin Core
- Bitcoin Optech topic: Regtest networks
