# RPC Error Codes Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/rpc`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/src/rpc/protocol.h
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/rpc/SKILL.md

## Concept

Bitcoin Core RPC errors come in two layers. The transport layer reuses the JSON-RPC 2.0 codes (`-32600` invalid request, `-32601` method not found, `-32602` invalid params, `-32603` internal error, `-32700` parse error). The application layer uses Bitcoin-specific codes defined in `src/rpc/protocol.h` (`RPC_*`). HTTP status is largely unused for application errors; you must inspect the JSON body. A robust integration treats the application code as the primary signal and the message string only as a hint, since the message text is not API-stable.

## Walkthrough / mechanics

Common application codes you must handle:

- `-1 RPC_MISC_ERROR`: generic. Usually paired with a more specific message ("Block not found in any chain"). Treat as "retry might help, log everything".
- `-3 RPC_TYPE_ERROR`: parameter type wrong (e.g., passed string where boolean expected).
- `-5 RPC_INVALID_ADDRESS_OR_KEY`: object asked for does not exist on this node. The same code covers an unknown txid, an unknown address, an unknown blockhash, and a not-yet-imported descriptor.
- `-8 RPC_INVALID_PARAMETER`: parameter shape valid but value rejected (out-of-range height, malformed hex).
- `-22 RPC_DESERIALIZATION_ERROR`: serialized data invalid (bad PSBT, bad hex tx).
- `-25 RPC_VERIFY_ERROR`: validation failed for a tx submission attempt.
- `-26 RPC_VERIFY_REJECTED`: tx rejected by mempool policy. The message starts with a TRR (tx rejection reason) like `txn-mempool-conflict`, `min relay fee not met`, `bad-txns-inputs-missingorspent`, `dust`.
- `-27 RPC_VERIFY_ALREADY_IN_CHAIN`: txid already confirmed. Idempotency win, not an error in most workflows.
- `-32 RPC_OUT_OF_MEMORY`: resource exhaustion, retry later.
- `-4 RPC_WALLET_ERROR` and the `-13`..`-19` family for wallet specifics (passphrase wrong, wallet locked, key not in keypool, insufficient funds).

Wrap RPC calls so `-5` and `-25/-26/-27` are not collapsed into a generic "failed". Each demands a different response.

## Worked example

Reproduce three common errors against a synced node and inspect the body:

```bash
# -5: unknown txid
$ bitcoin-cli getrawtransaction 0000000000000000000000000000000000000000000000000000000000000000
error code: -5
error message:
No such mempool transaction. Use -txindex or provide a block hash to enable blockchain transaction queries. Use gettransaction for wallet transactions.

# -8: out of range height
$ bitcoin-cli getblockhash 999999999
error code: -8
error message:
Block height out of range

# -26: tx rejected (broadcast a duplicate)
$ HEX=$(bitcoin-cli getrawtransaction <known_confirmed_txid> 0 <blockhash>)
$ bitcoin-cli sendrawtransaction "$HEX"
error code: -27
error message:
Transaction already in block chain
```

Programmatic handling:

```python
from bitcoinrpc.authproxy import AuthServiceProxy, JSONRPCException

rpc = AuthServiceProxy("http://user:pw@127.0.0.1:8332/")

def safe_get_tx(txid: str):
    try:
        return rpc.getrawtransaction(txid, True)
    except JSONRPCException as e:
        if e.code == -5:
            return None  # unknown
        if e.code == -8:
            raise ValueError("malformed txid") from e
        raise

def safe_broadcast(hex_tx: str):
    try:
        return ("ok", rpc.sendrawtransaction(hex_tx))
    except JSONRPCException as e:
        if e.code == -27:
            return ("already_mined", None)
        if e.code == -26:
            return ("rejected", e.message)  # parse e.message for TRR
        raise
```

For `-26`, the message string is the truth. Examples and how to react:

| Message fragment | Cause | Action |
|---|---|---|
| `min relay fee not met` | feerate below `-minrelaytxfee` | bump fee, rebroadcast |
| `txn-mempool-conflict` | input already spent by mempool tx | replace via RBF or wait |
| `bad-txns-inputs-missingorspent` | input already spent on chain | input is gone, redo |
| `non-BIP68-final` | `nSequence` locktime not satisfied | wait for blocks |
| `tx-size` | transaction too large per policy | split or simplify |
| `dust` | output below dust threshold | raise output value |

## Common pitfalls

- Trusting HTTP status. A JSON-RPC error returns HTTP 500 (legacy) or HTTP 200 (newer Core) depending on the error class. Always parse the body.
- Pattern-matching `e.message` instead of `e.code`. Messages are localized in some clients and have changed across releases (`-25` versus `-26` for different rejection paths).
- Treating `-27 already in chain` as failure. For most resubmission flows it is success.
- Catching all `JSONRPCException` and retrying. Retrying `-8 invalid parameter` will never succeed; retrying `-32 out of memory` might.
- Forgetting that `walletprocesspsbt` returns a JSON object (not an error) when it cannot finalize. Absence of `complete: true` is a soft failure, not an exception.

## References

- `src/rpc/protocol.h` in bitcoin/bitcoin (RPCErrorCode enum).
- `src/policy/policy.cpp` for tx rejection reasons (TRRs).
- `doc/JSON-RPC-interface.md`.
