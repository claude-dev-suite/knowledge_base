# Python Functional Test Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/testing/core-test-framework`.
> Canonical source: https://github.com/bitcoin/bitcoin/tree/master/test/functional
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/testing/core-test-framework/SKILL.md

## Concept

Bitcoin Core's `test/functional/` runs Python tests that spin up real
`bitcoind` subprocesses, drive them via JSON-RPC, and assert on
node-observable state. Each test is a single `.py` file ending in
`__main__.main()`, runnable in isolation, runnable in the suite via
`test_runner.py`. Since the CMake migration in Bitcoin Core 29.0
(April 2025) the sources still live in `test/functional/` but the
runnable copies are configured into `build/test/functional/`, which is
where both the individual scripts and `test_runner.py` are invoked
from. The framework is also redistributable: many out-of-tree projects
(taproot soft-fork forks, mempool research nodes) vendor
`test_framework/` and write tests against their patched bitcoind.

A functional test is the highest-fidelity Bitcoin test you can write
without leaving Python: every line corresponds to a real RPC call to a
real validator. The cost is per-test wall time (each test boots and
shuts down nodes), so the framework is not a substitute for unit tests
but the canonical place for end-to-end policy and consensus checks.

## Walkthrough / mechanics

A functional test inherits from `BitcoinTestFramework` and overrides:

- `set_test_params(self)`: how many nodes, with which `extra_args`,
  and whether to start from a clean chain.
- `setup_network(self)` (optional): customise peer topology. Default is
  fully connected.
- `run_test(self)`: the actual test logic.

The framework handles:

- Allocating distinct datadirs and ports under `--cachedir`.
- Starting/stopping `bitcoind` instances.
- Wiring connections between nodes.
- Caching a pre-mined chain across tests via `--cachedir`.
- Dumping logs on failure.

Run:

```bash
build/test/functional/feature_my_test.py --loglevel=DEBUG
```

Test runner with parallelism:

```bash
build/test/functional/test_runner.py --jobs=8 feature_csv_activation.py wallet_basic.py
```

Useful framework helpers (from `test_framework/util.py`):

- `assert_equal(a, b)`, `assert_raises_rpc_error(code, msg, fn, *args)`
- `try_rpc(code, msg, fn, *args)`, `assert_greater_than(a, b)`
- `satoshi_round(amount, *, rounding)`, `count_bytes`
- `MiniWallet` (from `test_framework/wallet.py`) for descriptor-free
  output management

Polling is a framework method rather than a `util.py` export:
`self.wait_until(test_function, timeout=60, check_interval=0.05)`.

## Worked example

Test that a CSV-locked output is rejected before maturity and accepted
after, using two nodes:

```python
#!/usr/bin/env python3
from test_framework.test_framework import BitcoinTestFramework
from test_framework.util import (
    assert_equal,
    assert_raises_rpc_error,
)
from test_framework.wallet import MiniWallet
from test_framework.script import (
    CScript,
    OP_CHECKSEQUENCEVERIFY,
    OP_DROP,
    OP_TRUE,
)
from test_framework.messages import COIN

CSV_BLOCKS = 10  # relative locktime in blocks

class CsvMaturityTest(BitcoinTestFramework):
    def set_test_params(self):
        self.num_nodes = 2
        self.setup_clean_chain = True
        self.extra_args = [["-fallbackfee=0.00001"]] * 2

    def run_test(self):
        node = self.nodes[0]
        wallet = MiniWallet(node)
        self.generate(wallet, 200)  # mature coinbases

        # Build a CSV-locked output
        redeem = CScript([CSV_BLOCKS, OP_CHECKSEQUENCEVERIFY, OP_DROP, OP_TRUE])
        utxo = wallet.get_utxo()
        funding = wallet.create_self_transfer(
            utxo_to_spend=utxo,
            target_vsize=110,
            script_pub_key=redeem,
        )
        node.sendrawtransaction(funding["hex"])
        self.generate(node, 1)

        # Try to spend immediately - rejected
        spend = wallet.create_self_transfer(
            utxo_to_spend={"txid": funding["txid"], "vout": 0,
                           "value": funding["new_utxo"]["value"]},
            sequence=CSV_BLOCKS,
        )
        assert_raises_rpc_error(
            -26, "non-BIP68-final",
            node.sendrawtransaction, spend["hex"],
        )

        # Mine CSV_BLOCKS-1 more, still too early
        self.generate(node, CSV_BLOCKS - 1)
        assert_raises_rpc_error(
            -26, "non-BIP68-final",
            node.sendrawtransaction, spend["hex"],
        )

        # One more block: now mature
        self.generate(node, 1)
        node.sendrawtransaction(spend["hex"])
        self.sync_mempools()
        assert_equal(len(self.nodes[1].getrawmempool()), 1)

if __name__ == "__main__":
    CsvMaturityTest().main()
```

Run it standalone:

```bash
cd bitcoin
build/test/functional/feature_csv_maturity.py --loglevel=INFO
# ... 2024-01-01T00:00:00.000Z TestFramework (INFO): Tests successful
```

Debug a failure: pass `--nocleanup` to keep node datadirs, then
`tail -F /tmp/bitcoin_test_runner_.../node0/regtest/debug.log`.

## Common pitfalls

- `setup_clean_chain = False` (the default) reuses the shared cachedir
  with a 200-block pre-mined chain. If your test mutates global state
  (e.g. activates a soft fork at a low height) set it to True.
- `MiniWallet` does not understand descriptors; for taproot tests use
  the descriptor wallet via `node.createwallet`.
- `assert_raises_rpc_error` matches by substring on the error message.
  RPC error message text changes between versions; pin specific bits.
- Mining must go through the framework helpers. Since Bitcoin Core 23.0
  (April 2022) `node.generatetoaddress()`, `node.generateblock()` and
  `node.generatetodescriptor()` have carried a keyword-only guard that
  only the framework helpers pass, so a direct call has failed since
  then; Bitcoin Core 29.0 (April 2025) renamed that guard from
  `invalid_call` to `called_by_framework` and attached the message
  "Direct call of this mining RPC is discouraged". `node.generate()` has
  no guard of its own — it dispatches to `generatetoaddress`, so a
  direct call fails with `TypeError` for the missing keyword-only
  argument instead. Use `self.generate(node, n)` /
  `self.generatetoaddress(node, n, addr)`, which take the generating
  node first and sync afterwards, and pass `sync_fun=self.no_op` when
  you deliberately want the nodes left out of sync.
- Tests must clean up subprocesses on early exit. Use `try/finally`
  around `bitcoind` setup if you fork off custom helpers.

## References

- Bitcoin Core `test/functional/README.md`
- `test/functional/test_framework/test_framework.py`
- Optech topic: Functional tests
- Real-world examples: `feature_taproot.py`, `mempool_packages.py`
