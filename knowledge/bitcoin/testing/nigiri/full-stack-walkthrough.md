# Nigiri Full-Stack Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/testing/nigiri`.
> Canonical source: https://github.com/vulpemventures/nigiri
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/testing/nigiri/SKILL.md

## Concept

Nigiri is a CLI-driven regtest stack: a single `nigiri start` boots
`bitcoind`, `electrs`, an Esplora frontend, and optionally a Liquid
node, an LN node (CLN), and "Chopsticks" (a deterministic signing
service). The defining feature versus Polar is the Esplora REST API at
`http://localhost:3000/api/`, which most modern Bitcoin wallets (BDK,
Sparrow, Mutiny, Phoenix, Blue Wallet) speak natively.

If your application talks to Esplora in production, Nigiri is the most
faithful local fixture you can run: same REST surface, same address
indexing, same UTXO endpoints. Polar is for Lightning topology
sculpting; Nigiri is for "wire my wallet to a real Esplora and watch
on-chain flow."

## Walkthrough / mechanics

`nigiri start` orchestrates Docker Compose under the hood. Default
ports:

- bitcoind RPC: `18443` (user `admin1`, pass `123`)
- bitcoind P2P: `19444`
- electrs: `50001` (Electrum protocol)
- Esplora REST/HTTP UI: `3000`
- Chopsticks: `3001`

`nigiri faucet <address> [amount_btc]` mines a coinbase, sends the
output to the address, mines 1 confirmation. `nigiri rpc <method>
<args...>` proxies to bitcoind. `nigiri logs <service>` tails logs.

Liquid mode adds an `elementsd` + Liquid Esplora at port `3001`. LN
mode adds a CLN node at port `9735` with an HTTP REST plugin.

State persists across `nigiri stop` / `nigiri start`. Use
`nigiri stop --delete` to wipe.

## Worked example

End-to-end test of a wallet that consumes Esplora REST. The wallet
ships with a Python integration test that hits
`http://localhost:3000/api/`:

```bash
nigiri start
# Wait for ready
until curl -sf http://localhost:3000/api/blocks/tip/height >/dev/null; do
  sleep 1
done

# Fund a regtest address
ADDR=$(python3 -c "from bdkpython import *; print(create_test_addr())")
nigiri faucet $ADDR 1
# Esplora indexes within ~1s
curl -s "http://localhost:3000/api/address/$ADDR/utxo" | jq .

# Run the wallet's test suite against this Esplora
ESPLORA_URL=http://localhost:3000/api pytest tests/integration/
```

Python integration test using BDK + Esplora client:

```python
import os, subprocess, time
import bdkpython as bdk

ESPLORA = os.environ.get("ESPLORA_URL", "http://localhost:3000/api")

def faucet(addr, btc=0.5):
    subprocess.run(["nigiri", "faucet", addr, str(btc)], check=True)

def mine(n=1):
    subprocess.run(
        ["nigiri", "rpc", "generatetoaddress", str(n),
         "bcrt1qxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"],
        check=True,
    )

def test_receive_and_spend():
    desc = bdk.Descriptor("wpkh(...)", bdk.Network.REGTEST)
    wallet = bdk.Wallet(desc, None, bdk.Network.REGTEST,
                        bdk.MemoryDatabase())
    addr = wallet.get_address(bdk.AddressIndex.NEW).address
    faucet(addr.as_string(), 1.0)
    time.sleep(2)

    blockchain = bdk.Blockchain(bdk.BlockchainConfig.ESPLORA(
        bdk.EsploraConfig(ESPLORA, None, 5, 20, None)))
    wallet.sync(blockchain, None)
    assert wallet.get_balance().total > 0

    txb = bdk.TxBuilder()
    txb.add_recipient(addr.script_pubkey(), 50_000)
    psbt, details = txb.finish(wallet)
    wallet.sign(psbt, None)
    blockchain.broadcast(psbt.extract_tx())
    mine(1)
    wallet.sync(blockchain, None)
    assert details.fee > 0
```

Liquid + LN combined:

```bash
nigiri start --liquid --ln
nigiri liquid faucet ert1q... 1
docker exec nigiri-cln lightning-cli getinfo
```

CI usage in GitHub Actions:

```yaml
jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: curl https://getnigiri.vulpem.com | bash
      - run: nigiri start
      - run: pytest tests/e2e/
        env:
          ESPLORA_URL: http://localhost:3000/api
      - if: always()
        run: nigiri stop --delete
```

## Common pitfalls

- Port collisions: another local Esplora (3000) or electrs (50001) will
  prevent startup. Use `nigiri start --port-bitcoin-rpc 18444 ...` to
  remap, or stop the conflicting service.
- Cold start latency: electrs takes ~5-15 seconds to index after
  startup. A naive test that immediately queries `/address/{a}/utxo`
  may see empty results. Wait for `/blocks/tip/height` first.
- Nigiri runs `txindex=1` and `blockfilterindex=1` by default; on tiny
  CI runners this is fine on regtest but be aware of disk usage on
  long sessions.
- The provided faucet address (`bcrt1qxxxxxxxxx...`) is a placeholder;
  use a real regtest address from your wallet for `generatetoaddress`.
- Container image versions drift; pin them in CI by setting
  `NIGIRI_BITCOIND_IMAGE` and friends to specific tags.

## References

- Nigiri repo: `https://github.com/vulpemventures/nigiri`
- Esplora REST API: `https://github.com/Blockstream/esplora/blob/master/API.md`
- BDK Esplora client: `https://docs.rs/bdk_esplora/`
- electrs: `https://github.com/blockstream/electrs`
