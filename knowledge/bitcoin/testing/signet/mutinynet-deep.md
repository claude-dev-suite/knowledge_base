# Mutinynet Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/testing/signet`.
> Canonical source: https://blog.mutinywallet.com/mutinynet/ and https://faucet.mutinynet.com/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/testing/signet/SKILL.md

## Concept

Mutinynet is a custom signet operated by the MutinyWallet team with two
deliberate departures from default signet:

1. 30-second target block interval (vs 10 minutes on default signet).
2. A predictable, public Esplora + faucet + block explorer stack.

The 30-second blocks make Mutinynet uniquely useful for Lightning
development. Channel funding confirmations, HTLC timeouts, and on-chain
sweep flows that take hours on mainnet take minutes here, while still
exercising real network propagation, taproot, BOLT12, and any policy a
mainnet-shaped network has. It is the realistic substitute for "let me
just deploy on mainnet to see how it behaves."

## Walkthrough / mechanics

To join Mutinynet, your node must agree on three things:

- The signet challenge (`signetchallenge=`) - this defines the network.
- A bootstrap peer (`addnode=` or `signetseednode=`).
- The activation height for any soft fork the network has merged.

The 30-second cadence is enforced by the signing daemon, not by
consensus. Validators only check signatures; if the signers stop, the
chain stalls. The Mutiny team publish current parameters and challenge
on their blog; the values rotate occasionally for protocol upgrades.

Faucet API: `https://faucet.mutinynet.com/api/onchain` accepts
`{"address": ..., "sats": ...}` and returns a txid that confirms in
~30s. There is also a Lightning faucet for inbound liquidity.

Esplora: `https://mutinynet.com/api/` exposes the standard Esplora REST
API (`/address/{a}/utxo`, `/tx/{tx}/hex`, etc.) so existing Esplora
clients (BDK, Sparrow, Mutiny, LDK) work unchanged.

## Worked example

Run a Mutinynet-aware Bitcoin Core node and exercise the Lightning
flow against an LND backend in 30 seconds per confirmation:

```bash
mkdir -p /tmp/mutiny
cat > /tmp/mutiny/bitcoin.conf <<'EOF'
signet=1
[signet]
signetchallenge=512102f7561d208dd9ae99bf497273e16f389bdbd6c4742ddb8e6b216e64fa2928ad8f51ae
addnode=45.79.52.207:38333
fallbackfee=0.00001
txindex=1
EOF
bitcoind -datadir=/tmp/mutiny -daemon

# Wait until tip is current
bitcoin-cli -datadir=/tmp/mutiny -signet getblockchaininfo | jq '.headers, .blocks'

# Top up the wallet from the faucet
bitcoin-cli -datadir=/tmp/mutiny -signet createwallet w
ADDR=$(bitcoin-cli -datadir=/tmp/mutiny -signet -rpcwallet=w getnewaddress -addresstype bech32m)
curl -s -X POST https://faucet.mutinynet.com/api/onchain \
  -H 'Content-Type: application/json' \
  -d "{\"address\":\"$ADDR\",\"sats\":1000000}"

# 30s later
bitcoin-cli -datadir=/tmp/mutiny -signet -rpcwallet=w getbalance
```

End-to-end Lightning channel test (using LND pointed at the node):

```bash
lnd --bitcoin.signet \
    --bitcoin.node=bitcoind \
    --bitcoind.dir=/tmp/mutiny \
    --bitcoind.rpcuser=$(...) --bitcoind.rpcpass=$(...) \
    --neutrino.connect=45.79.52.207:38333

lncli newaddress p2tr
# fund via faucet, wait one block (~30s)
lncli connect <peer-pubkey>@<peer>
lncli openchannel --node_key=<peer-pubkey> --local_amt=200000
# confirms in 6 blocks = ~3 minutes (vs 1 hour on mainnet)
```

Python integration test using `python-bitcoinrpc` + Esplora REST:

```python
import requests
from bitcoinrpc.authproxy import AuthServiceProxy

mutiny = AuthServiceProxy("http://user:pass@127.0.0.1:38332")
ADDR = mutiny.getnewaddress("", "bech32m")
r = requests.post(
    "https://faucet.mutinynet.com/api/onchain",
    json={"address": ADDR, "sats": 100_000},
    timeout=15,
)
txid = r.json()["txid"]
# Poll Esplora for confirmation
import time
while True:
    s = requests.get(f"https://mutinynet.com/api/tx/{txid}/status").json()
    if s["confirmed"]:
        break
    time.sleep(3)
```

## Common pitfalls

- Hard-coding the challenge: it has rotated at least once when the
  signers added taproot activation parameters. Always re-read the
  current value from the Mutiny blog before a long-lived deploy.
- Treating Mutinynet as eternal: there are no guarantees of stability.
  Do not rely on UTXOs or channels surviving longer than a few months.
- Esplora rate-limits. For load tests, run your own electrs/Esplora
  pointed at your local Mutinynet node.
- 30-second blocks make `feerate` estimation jumpy. For wallet tests
  prefer a fixed `fallbackfee`.
- Confusing default signet and Mutinynet wallets: they share the
  `tb1...` bech32 HRP but coins are not interchangeable.

## References

- Mutiny blog: `https://blog.mutinywallet.com/`
- Faucet: `https://faucet.mutinynet.com/`
- Esplora: `https://mutinynet.com/`
- BIP325: `https://github.com/bitcoin/bips/blob/master/bip-0325.mediawiki`
