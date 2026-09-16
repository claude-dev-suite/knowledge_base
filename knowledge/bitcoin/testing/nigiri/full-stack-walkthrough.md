# Nigiri Full-Stack Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/testing/nigiri`.
> Canonical source: https://github.com/vulpemventures/nigiri
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/testing/nigiri/SKILL.md

## Concept

Nigiri is a CLI-driven regtest stack: a single `nigiri start` boots
`bitcoind`, `electrs`, an Esplora frontend and "Chopsticks" (a JSON
HTTP proxy that re-serves every Esplora REST endpoint and extends it
with `POST /faucet` and automatic block generation on `POST /tx`).
Optional flags add a Liquid/Elements node (`--liquid`), Lightning
(`--ln`, which starts Core Lightning *and* LND *and* the Taproot
Assets daemon) and Ark (`--ark`). The defining feature versus Polar is
the Esplora-compatible REST API that Chopsticks serves at
`http://localhost:3000`, which most modern Bitcoin wallets (BDK,
Sparrow, Mutiny, Phoenix, Blue Wallet) speak natively.

If your application talks to Esplora in production, Nigiri is the most
faithful local fixture you can run: same REST surface, same address
indexing, same UTXO endpoints. Polar is for Lightning topology
sculpting; Nigiri is for "wire my wallet to a real Esplora and watch
on-chain flow."

## Walkthrough / mechanics

`nigiri start` orchestrates Docker Compose under the hood. Default
ports, read from the shipped `docker-compose.yml` at tag `v0.5.17`
(May 2026 release, unchanged on `master` as of September 2026):

- bitcoind RPC: `18443` (user `admin1`, pass `123`)
- bitcoind P2P: `18444`; ZMQ rawblock `28332`, rawtx `28333`
- electrs: `50000` (Electrum protocol), `30000` (electrs HTTP)
- electrum-ws: `50003` (websocat bridge onto electrs `50000`, for
  browser clients that cannot open raw TCP)
- Chopsticks - Esplora REST + `/faucet` + auto-mining: `3000`
- Esplora web explorer UI: `5000`

`nigiri faucet <address> [amount] [asset]` posts to Chopsticks
`/faucet`, which funds the address and mines a block; `nigiri faucet
cln|lnd <amount>` funds the LN nodes directly. `nigiri rpc <method>
<args...>` proxies to bitcoind (`nigiri rpc --liquid ...` to
elementsd). `nigiri logs <service>` tails logs, where `<service>` is
one of `bitcoin`, `electrs`, `chopsticks`, `liquid`,
`electrs-liquid`, `chopsticks-liquid`, `cln`, `lnd`.

Liquid mode (`--liquid`) adds `elementsd` (RPC `18884`, P2P `18886`),
`electrs-liquid` (`50001` Electrum / `30001` HTTP), a Liquid
Chopsticks at `3001` and a Liquid Esplora UI at `5001`. LN mode
(`--ln`) adds Core Lightning (P2P `9935`, RPC `9835`, gRPC `9936`),
LND (P2P `9735`, gRPC `10009`, REST `18080`) and `tapd` (gRPC
`10029`, REST `8089`) - driven via `nigiri cln`, `nigiri lnd` and
`nigiri tap`. Ark mode (`--ark`) adds `arkd` on `7070` plus its
explorer on `7080`.

State persists across `nigiri stop` / `nigiri start`. Use
`nigiri stop --delete` to wipe.

## Worked example

End-to-end test of a wallet that consumes Esplora REST. The wallet
ships with a Python integration test that hits
`http://localhost:3000`:

```bash
nigiri start
# Wait for ready
until curl -sf http://localhost:3000/blocks/tip/height >/dev/null; do
  sleep 1
done

# Fund a regtest address
ADDR=$(python3 -c "from bdkpython import *; print(create_test_addr())")
nigiri faucet $ADDR 1
# Esplora indexes within ~1s
curl -s "http://localhost:3000/address/$ADDR/utxo" | jq .

# Run the wallet's test suite against this Esplora
ESPLORA_URL=http://localhost:3000 pytest tests/integration/
```

Python integration test using BDK + Esplora client:

```python
import os, subprocess, time
import bdkpython as bdk

ESPLORA = os.environ.get("ESPLORA_URL", "http://localhost:3000")

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
nigiri faucet --liquid ert1q... 1
nigiri cln getinfo
nigiri lnd getinfo
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
          ESPLORA_URL: http://localhost:3000
      - if: always()
        run: nigiri stop --delete
```

## Common pitfalls

- Port collisions: another local Chopsticks/Esplora API (3000) or
  electrs (50000) will prevent startup, and on macOS Monterey and
  later AirPlay Receiver already binds 5000 so the Esplora UI fails
  with `address already in use`. `nigiri start` has no port flags (as
  of v0.5.17, May 2026 - its only flags are `--liquid`, `--ln`,
  `--ark`, `--ci`, `--remember`); remap by editing the
  `docker-compose.yml` inside the Nigiri datadir - POSIX and WSL
  `~/.nigiri`, macOS `$HOME/Library/Application Support/Nigiri`,
  native Windows `%LOCALAPPDATA%\Nigiri`, overridable with `--datadir`
  / `NIGIRI_DATADIR` - then `nigiri stop --delete` before restarting.
- Cold start latency: electrs takes ~5-15 seconds to index after
  startup. A naive test that immediately queries `/address/{a}/utxo`
  may see empty results. Wait for `/blocks/tip/height` first.
- Nigiri runs `txindex=1` and `blockfilterindex=1` by default; on tiny
  CI runners this is fine on regtest but be aware of disk usage on
  long sessions.
- The provided faucet address (`bcrt1qxxxxxxxxx...`) is a placeholder;
  use a real regtest address from your wallet for `generatetoaddress`.
- Container image versions drift, and there is no env var to override
  them (`NIGIRI_DATADIR` is the only `NIGIRI_*` variable the CLI reads
  as of v0.5.17). Pin in CI by installing a fixed Nigiri release and,
  if needed, editing the image tags in the datadir's
  `docker-compose.yml`. At v0.5.17 those tags are bitcoind `v30.0`,
  LND `v0.19.3-beta`, CLN `v25.09.3`, tapd `v0.6.1` and `arkd`
  `v0.9.4`; electrs, Esplora and Chopsticks track `latest`.

## References

Project status: actively maintained as of September 2026 - latest
release `v0.5.17` (13 May 2026), repo last pushed 10 July 2026, not
archived.

- Nigiri repo: `https://github.com/vulpemventures/nigiri`
- Shipped compose file (source of truth for ports):
  `https://github.com/vulpemventures/nigiri/blob/master/cmd/nigiri/resources/docker-compose.yml`
- Nigiri Chopsticks: `https://github.com/vulpemventures/nigiri-chopsticks`
- Esplora REST API: `https://github.com/Blockstream/esplora/blob/master/API.md`
- BDK Esplora client: `https://docs.rs/bdk_esplora/`
- electrs: `https://github.com/blockstream/electrs`
