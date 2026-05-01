# btcd Node Configuration - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/btcd`.
> Canonical source: https://github.com/btcsuite/btcd
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/btcd/SKILL.md

## Concept

`btcd` is a full-node Bitcoin implementation in Go by the btcsuite team.
It implements the consensus rules, P2P networking, mempool, and a JSON-
RPC server compatible with most `bitcoind` calls. It pairs with
`btcwallet` (separate daemon) for HD wallet functionality and
`btcctl` for the CLI client.

Adoption is small relative to Bitcoin Core (well under 5% of full
nodes), so production exchanges and treasuries should still prefer
`bitcoind`. Where btcd shines: Go-native services that want an
in-tree node, a backend for early-version LND deployments, and
educational study of clean Go code.

## Configuration walkthrough

`btcd` reads `btcd.conf` (TOML-ish key=value) plus command-line flags.
Common config:

```ini
# /var/lib/btcd/btcd.conf
datadir=/var/lib/btcd
logdir=/var/log/btcd

rpcuser=btcdrpc
rpcpass=$(openssl rand -hex 32)
rpclisten=127.0.0.1:8334
rpccert=/etc/btcd/rpc.cert
rpckey=/etc/btcd/rpc.key

listen=0.0.0.0:8333
externalip=203.0.113.5:8333

txindex=1
addrindex=1                  # required by some tools

testnet=0                    # mainnet (set 1 for testnet3)

# Optional: peers to seed connections
addpeer=node1.example:8333
addpeer=node2.example:8333

# Resource limits
maxpeers=125
banduration=24h
```

Generate the RPC TLS keypair with `gencerts`:

```bash
btcd --datadir=/var/lib/btcd  # first run generates rpc.cert
```

Then start the daemon:

```bash
btcd --configfile=/var/lib/btcd/btcd.conf
```

## Worked example: query and watch via Go RPC client

```go
package main

import (
    "io/ioutil"
    "log"

    "github.com/btcsuite/btcd/rpcclient"
)

func main() {
    cert, err := ioutil.ReadFile("/var/lib/btcd/rpc.cert")
    if err != nil { log.Fatal(err) }

    cfg := &rpcclient.ConnConfig{
        Host:         "127.0.0.1:8334",
        Endpoint:     "ws",
        User:         "btcdrpc",
        Pass:         "secret",
        Certificates: cert,
    }

    ntfns := &rpcclient.NotificationHandlers{
        OnBlockConnected: func(hash *chainhash.Hash, height int32, t time.Time) {
            log.Printf("block: %d %s", height, hash)
        },
    }
    client, err := rpcclient.New(cfg, ntfns)
    if err != nil { log.Fatal(err) }
    defer client.Shutdown()

    if err := client.NotifyBlocks(); err != nil { log.Fatal(err) }

    info, err := client.GetBlockChainInfo()
    if err != nil { log.Fatal(err) }
    log.Printf("height=%d chain=%s", info.Blocks, info.Chain)

    client.WaitForShutdown()
}
```

`rpcclient` supports both HTTP POST (set `HTTPPostMode: true`,
`DisableTLS` for local plaintext) and persistent WebSocket
notifications (above). LND-derived tooling typically uses the latter.

## CLI

```bash
btcctl --rpcuser=btcdrpc --rpcpass=... getblockchaininfo
btcctl --rpcuser=btcdrpc --rpcpass=... getrawmempool
btcctl --rpcuser=btcdrpc --rpcpass=... sendrawtransaction <hex>
```

## Common pitfalls

- Soft-fork support sometimes ships months after Core. Check
  `getblockchaininfo` `softforks` field before depending on a
  feature; if a fork hasn't activated for btcd's tracker, you may
  see consensus splits in tests.
- Mempool standardness rules differ from Core: a tx accepted into
  bcoin's mempool may not relay to Core peers. For wallets you ship
  to end users, broadcast through bitcoind too if available.
- TLS certs are mandatory by default. `--notls` exists but disables
  WebSocket notifications; `--listen=` empty + `--rpclisten=127.0.0.1`
  is the sane default for local dev.
- `addrindex` is required for tools like `btcwallet` to look up by
  address; flip it on at first sync since rebuilding it later costs
  a full rescan.
- For LND: prefer `bitcoind` backend in modern deployments; btcd
  backend remains supported but receives less testing.

## References

- Repo: https://github.com/btcsuite/btcd
- rpcclient docs: https://pkg.go.dev/github.com/btcsuite/btcd/rpcclient
- Sample config: https://github.com/btcsuite/btcd/blob/master/sample-btcd.conf
- Companion: [btcsuite/SKILL.md](../btcsuite/SKILL.md), [lnd-go/SKILL.md](../lnd-go/SKILL.md)
