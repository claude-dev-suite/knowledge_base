# BitcoinTestFramework Helpers - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/testing/core-test-framework`.
> Canonical source: https://github.com/bitcoin/bitcoin/tree/master/test/functional/test_framework
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/testing/core-test-framework/SKILL.md

## Concept

The `test_framework/` package is more than a test runner. It is a
nearly-complete pure-Python reimplementation of Bitcoin's wire format,
script engine, key handling, descriptor parser, and PSBT logic - all in
service of letting tests construct and validate transactions without
going through bitcoind. This makes it valuable far beyond Bitcoin Core
itself: any project that needs to build raw txs, parse blocks, or
simulate P2P behaviour can vendor or import these modules.

Knowing the helpers turns a functional test from "a series of RPC
calls" into "a precise specification of what bytes hit the wire and
what response is expected."

## Walkthrough / mechanics

The shipped modules and what they cover (master, as of September 2026):

- `messages.py`: wire types `CTransaction`, `CTxIn`, `CTxOut`, `CBlock`,
  `CBlockHeader`, `CInv`, P2P message classes, `ser_uint256`,
  `tx_from_hex`, `from_hex` helpers.
- `script.py`: `CScript`, all opcodes (`OP_CHECKSIG`,
  `OP_CHECKSEQUENCEVERIFY`, ...), `taproot_construct`, `tagged_hash`,
  Schnorr/ECDSA helpers via `key.py`.
- `key.py`: `ECKey`, `ECPubKey`, and module-level BIP340 helpers
  (`sign_schnorr`, `verify_schnorr`, `compute_xonly_pubkey`,
  `tweak_add_privkey`, `tweak_add_pubkey`). No BIP32 lives here.
- `extendedkey.py`: BIP32 derivation. `ExtendedPrivateKey`
  (`from_seed`, `generate`, `derive_path`, `pubkey`, `to_string`)
  and `ExtendedPublicKey` (`derive_path`, `to_string`). Added to
  master in June 2026; not present in any release up to and
  including Bitcoin Core 31.1 (July 2026), and first ships in the
  32.0 line (present at tag v32.0rc1, September 2026), so a copy
  vendored from an earlier release tag will not have it.
- `descriptors.py`: descriptor checksum compute and verify.
- `psbt.py`: round-trippable PSBT v0/v2.
- `wallet.py`: `MiniWallet` (no-descriptor in-test wallet), helpers to
  fund, create self-transfers, sweep.
- `wallet_util.py`: WIF encode, address encode for all networks.
- `p2p.py`: `P2PInterface` to attach a Python peer to bitcoind and
  inject arbitrary messages.
- `address.py`: bech32/bech32m, base58check.
- `util.py`: `assert_equal`, `assert_raises_rpc_error`,
  `assert_greater_than`, `try_rpc`, `count_bytes`, `satoshi_round`.
  Polling is not here: `util.py` carries only
  `wait_until_helper_internal`, and tests call
  `self.wait_until(test_function, timeout=60, check_interval=0.05)` on
  the framework (or the same method on a `TestNode`).

Three patterns combine these into useful tests:

1. RPC-only: orchestrate via `node.foo()`. Cheap and fast.
2. Raw-tx: build with `CTransaction()` + `messages.py`, sign with
   `key.py`, broadcast via `node.sendrawtransaction`.
3. P2P: open a `P2PInterface`, send `MsgTx`/`MsgBlock` directly to
   bitcoind without RPC, watch for responses.

## Worked example

Build a taproot key-path-only output, fund it via MiniWallet, then
spend it back, all inside one functional test - no descriptor wallet
needed:

```python
from test_framework.test_framework import BitcoinTestFramework
from test_framework.util import assert_equal
from test_framework.wallet import MiniWallet, MiniWalletMode
from test_framework.script import (
    CScript, taproot_construct, OP_TRUE,
    SIGHASH_DEFAULT, TaprootSignatureHash,
)
from test_framework.messages import (
    CTransaction, CTxIn, CTxOut, COutPoint, COIN, CTxInWitness,
)
from test_framework.key import (
    ECKey, compute_xonly_pubkey, sign_schnorr, tweak_add_privkey,
)

class TaprootKeyspendTest(BitcoinTestFramework):
    def set_test_params(self):
        self.num_nodes = 1
        self.extra_args = [["-fallbackfee=0.00001"]]

    def run_test(self):
        node = self.nodes[0]
        wallet = MiniWallet(node, mode=MiniWalletMode.RAW_OP_TRUE)
        self.generate(wallet, 101)

        sk = ECKey()
        sk.generate()
        xonly_pub, _ = compute_xonly_pubkey(sk.get_bytes())
        info = taproot_construct(xonly_pub, [])  # key-path only
        spk = info.scriptPubKey

        # Fund the taproot output
        funding = wallet.send_to(
            from_node=node, scriptPubKey=spk, amount=50_000,
        )
        self.generate(node, 1)

        # Build spend
        spend = CTransaction()
        spend.vin.append(CTxIn(COutPoint(int(funding["txid"], 16),
                                         funding["sent_vout"])))
        spend.vout.append(CTxOut(48_000, CScript([OP_TRUE])))
        spend.wit.vtxinwit.append(CTxInWitness())

        # BIP341 sighash
        sighash = TaprootSignatureHash(
            spend, [CTxOut(50_000, spk)], SIGHASH_DEFAULT,
            input_index=0, scriptpath=False,
        )
        # Key-path spends sign with the *tweaked* secret, and
        # `sign_schnorr` is a module-level function over raw bytes.
        tweaked_sk = tweak_add_privkey(sk.get_bytes(), info.tweak)
        sig = sign_schnorr(tweaked_sk, sighash)
        spend.wit.vtxinwit[0].scriptWitness.stack = [sig]

        node.sendrawtransaction(spend.serialize().hex())
        self.generate(node, 1)
        assert_equal(node.getmempoolinfo()["size"], 0)

if __name__ == "__main__":
    TaprootKeyspendTest().main()
```

Inject a custom P2P message and assert on the response:

```python
from test_framework.p2p import P2PInterface
from test_framework.messages import msg_getdata, CInv, MSG_TX

class GetdataMissingTest(BitcoinTestFramework):
    def set_test_params(self):
        self.num_nodes = 1

    def run_test(self):
        peer = self.nodes[0].add_p2p_connection(P2PInterface())
        # Ask for a tx the node has never seen
        peer.send_message(msg_getdata([CInv(MSG_TX, 0xdeadbeef)]))
        peer.sync_with_ping()
        # bitcoind responds with notfound
        assert peer.last_message["notfound"] is not None
```

`assert_raises_rpc_error` for negative tests:

```python
from test_framework.util import assert_raises_rpc_error

assert_raises_rpc_error(
    -25, "Missing inputs",
    node.sendrawtransaction, broken_tx_hex,
)
```

## Common pitfalls

- `taproot_construct` returns a struct with `scriptPubKey`, `tweak`,
  `internal_pubkey`, etc. Spending key-path uses the *tweaked* secret;
  forgetting to tweak yields invalid signatures.
- `MiniWallet`'s `RAW_OP_TRUE` mode produces non-standard outputs; some
  tests need `MiniWalletMode.ADDRESS_OP_TRUE` for relay.
- `sign_ecdsa` is an `ECKey` method and returns DER; `sign_schnorr` is
  a module-level function in `key.py` taking raw 32-byte key material
  and returning 64 bytes. `sk.sign_schnorr(...)` raises
  `AttributeError` - call `sign_schnorr(sk.get_bytes(), msg)`.
- `P2PInterface` is single-threaded; do not block in callbacks.
- Versions of `test_framework` vendored from older Core releases may
  not include `taproot_construct` or the latest BIP utilities; pin a
  recent commit.

## References

- `test/functional/test_framework/` source tree
- BIP340/341/342: `https://github.com/bitcoin/bips/`
- Real-world tests: `feature_taproot.py`, `p2p_segwit.py`,
  `feature_csv_activation.py`
