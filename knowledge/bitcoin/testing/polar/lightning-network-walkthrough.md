# Polar Lightning Network Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/testing/polar`.
> Canonical source: https://github.com/jamaljsr/polar
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/testing/polar/SKILL.md

## Concept

Polar packages a complete regtest Lightning topology into Docker
containers driven by an Electron GUI. Beyond the GUI, every node
exposes its standard ports (LND gRPC 10009, CLN RPC socket and gRPC,
Eclair API) on the host, so existing client SDKs talk to Polar without
modification. This makes it a drop-in test fixture for any LN
application: the GUI is for shaping the topology, the SDKs are for
running tests.

The deep value is reproducibility. A saved Polar network is a directory
of node data + a `network.json` manifest. Check that into a fixtures
folder, hand it to a teammate, and they boot the exact same node
identities, channels, and balances.

## Walkthrough / mechanics

A Polar network is a graph of:

- 1 or more `bitcoind` regtest backends.
- N Lightning nodes, each pinned to an image version. Polar 4.0.0
  (6 January 2026) ships LND 0.16.4-0.20.0, Core Lightning
  24.08.1-25.12, Eclair 0.9.0-0.13.1, Lightning Terminal (litd)
  0.14.1-0.16.0 and Taproot Assets 0.3.3-0.7.0, against Bitcoin Core
  26.0-30.0. There is no LDK / LDK Node image.
- Channels between LN nodes with sat capacity and balance ratios.

Workflow:

1. Create network. Polar generates `network.json` and per-node
   directories under `~/.polar/networks/<id>/`.
2. Start: Polar boots Docker containers, mines 100 blocks to fund
   each LN node's on-chain wallet, then opens the requested channels.
3. Fund: each LN node receives a default 1 BTC from the auto-mined
   blocks; right-click "Send Onchain" or use RPC for more.
4. Open channels: GUI form or via RPC; capacity and push amount are
   parameterised.
5. Pay: GUI "Send/Receive" panel or use `lncli`/`lightning-cli`
   directly through Docker exec or by attaching to mapped ports.

Networks persist; you can stop and restart without losing state. To
script tests, hit each node's RPC over TCP via the host port mapping
shown in the GUI.

Polar 4.0.0 (6 January 2026) added two features aimed squarely at
scripted testing:

- **sim-ln activity.** Define activity rules (source node, destination
  node, amount, interval); while the simulation runs, Polar drives
  continuous payments across the topology through
  `bitcoin-dev-project/sim-ln`. Useful for soak tests and for
  generating realistic forwarding history before asserting on it.
- **MCP server.** Polar runs an HTTP bridge on `localhost:37373`, and
  the `@lightningpolar/mcp` stdio server (repo `jamaljsr/polar-mcp`)
  exposes 48 tools to an AI agent. Upstream's own category list in
  `docs/mcp-architecture.md` (read September 2026) enumerates 18
  network, 6 bitcoin, 10 lightning, 9 Taproot Assets and 3 LitD, which
  does not add up to the headline 48; treat the categories as
  indicative and `GET /api/mcp/tools` as authoritative. That bridge is
  also a plain HTTP API - `GET /api/mcp/tools`,
  `POST /api/mcp/execute` - so a test harness can drive Polar without
  the GUI.

## Worked example

Three-node payment chain Alice -> Bob -> Carol, scripted from outside
Polar (assuming Polar already started a network with these names):

```bash
# Polar maps ports incrementally: 10001/10002/10003 etc.
# Inspect via the GUI or:
docker ps --format '{{.Names}} {{.Ports}}' | grep polar

ALICE_RPC=10001
BOB_RPC=10002
CAROL_RPC=10003

LNCLI_ALICE="lncli --rpcserver=localhost:$ALICE_RPC \
  --tlscertpath=$HOME/.polar/networks/1/volumes/lnd/alice/tls.cert \
  --macaroonpath=$HOME/.polar/networks/1/volumes/lnd/alice/data/chain/bitcoin/regtest/admin.macaroon"

# 1. Get Bob and Carol pubkeys
BOB_PUB=$($LNCLI_BOB getinfo | jq -r .identity_pubkey)
CAROL_PUB=$($LNCLI_CAROL getinfo | jq -r .identity_pubkey)

# 2. Open channels (Polar typically pre-opens, but for explicit demo)
$LNCLI_ALICE openchannel --node_key=$BOB_PUB --local_amt=500000 --push_amt=200000
docker exec polar-bitcoind bitcoin-cli -regtest -generate 6
$LNCLI_BOB openchannel --node_key=$CAROL_PUB --local_amt=500000 --push_amt=200000
docker exec polar-bitcoind bitcoin-cli -regtest -generate 6

# 3. Carol creates an invoice
INV=$($LNCLI_CAROL addinvoice --amt=10000 | jq -r .payment_request)

# 4. Alice pays through Bob
$LNCLI_ALICE payinvoice --force $INV
```

Equivalent in Python with `lndgrpc`:

```python
from lndgrpc import LNDClient
import os, time, subprocess

base = os.path.expanduser("~/.polar/networks/1/volumes/lnd")
alice = LNDClient(
    "localhost:10001",
    cert_filepath=f"{base}/alice/tls.cert",
    macaroon_filepath=f"{base}/alice/data/chain/bitcoin/regtest/admin.macaroon",
)
carol = LNDClient(
    "localhost:10003",
    cert_filepath=f"{base}/carol/tls.cert",
    macaroon_filepath=f"{base}/carol/data/chain/bitcoin/regtest/admin.macaroon",
)

inv = carol.add_invoice(value=10_000)
resp = alice.send_payment_sync(payment_request=inv.payment_request)
assert resp.payment_error == "", resp.payment_error
print("preimage", resp.payment_preimage.hex())

# Mine to settle on-chain after closing
subprocess.run(["docker", "exec", "polar-n1-backend", "bitcoin-cli",
                "-regtest", "-generate", "6"], check=True)
```

For test isolation, always start each test from a freshly imported
Polar network ZIP using `Polar > Import Network`. That guarantees pubkeys
and channel IDs are stable.

## Common pitfalls

- Polar's auto-mined regtest stops at ~150 blocks. Long-running tests
  must keep mining (`docker exec polar-bitcoind bitcoin-cli -regtest
  -generate 1`) or HTLCs near expiry will time out.
- Mixed LN implementations (LND + CLN) sometimes fail to negotiate
  features when versions are far apart; pin compatible versions.
- Backend compatibility is pinned too: Polar's `docker/nodes.json`
  (read September 2026) maps LND 0.18.4-beta and newer to bitcoind
  30.0, and LND 0.18.3-beta and older to bitcoind 27.0. Pairing an old
  LND with a new backend is not a supported combination.
- Macaroon paths change when Polar version changes. Treat the location
  shown in the GUI as authoritative.
- Channel reserves: by default, LND keeps 1% of channel capacity as
  reserve. A "0 fee, full balance" payment hits reserve and fails.
- Docker on macOS with VirtioFS can deadlock when many small writes
  happen during channel open; use the gRPC mode in Docker Desktop.

## References

- Polar repo: `https://github.com/jamaljsr/polar`
- Polar v4.0.0 release notes:
  `https://github.com/jamaljsr/polar/releases/tag/v4.0.0`
- Polar MCP architecture:
  `https://github.com/jamaljsr/polar/blob/master/docs/mcp-architecture.md`
- Polar MCP server: `https://github.com/jamaljsr/polar-mcp`
- sim-ln: `https://github.com/bitcoin-dev-project/sim-ln`
- Polar docs site: `https://lightningpolar.com/`
- `lndgrpc` Python client: `https://github.com/adrienemery/lnd-grpc-client`
- Core Lightning JSON-RPC: `https://docs.corelightning.org/`
