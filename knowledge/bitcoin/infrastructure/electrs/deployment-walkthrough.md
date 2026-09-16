# Electrs Deployment Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/electrs`.
> Canonical source: https://github.com/romanz/electrs/blob/master/doc/install.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/electrs/SKILL.md

## Concept

This article walks an end-to-end production deployment of romanz/electrs
v0.12.0 (released 13 September 2026) on a Linux server with a synced Bitcoin
Core node. It covers the prerequisite `bitcoin.conf` flags, building electrs
from source with cargo, hardening with a systemd service, and exposing the
Electrum protocol over TLS via nginx so remote wallets (Sparrow, Electrum,
BlueWallet) can connect securely. The goal is a single-host node where
bitcoind, electrs, and a TLS terminator run as separate services with the
principle of least privilege.

## Walkthrough / mechanics

Step 1 - Bitcoin Core configuration. electrs v0.12.0 indexes through
`bindex` and pulls blocks over bitcoind's REST interface rather than the P2P
protocol, so it needs **Bitcoin Core 31.0+** (released April 2026) with
`rest=1` and pruning off. Edit `/home/bitcoin/.bitcoin/bitcoin.conf`:

```ini
server=1
rest=1
prune=0
txindex=1
blockfilterindex=1
peerblockfilters=1
rpcuser=bitcoin
rpcpassword=<long-random-string>
rpcbind=127.0.0.1
rpcallowip=127.0.0.1
zmqpubrawblock=tcp://127.0.0.1:28332
zmqpubrawtx=tcp://127.0.0.1:28333
```

Restart bitcoind and wait for `getindexinfo` to report all indexes synced.
electrs itself needs neither `txindex` nor the filter indexes - it builds its
own index - but they are kept here for the other services on this host (eclair
and BIP157 clients querying Core directly).

Step 2 - Build electrs. As a non-root user with Rust 1.85.0 or newer (the
MSRV since v0.11.0) and `build-essential libclang-dev` installed:

```bash
git clone https://github.com/romanz/electrs
cd electrs
git checkout v0.12.0
cargo build --locked --release
sudo install -m 755 target/release/electrs /usr/local/bin/electrs
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/electrs electrs
sudo install -d -o electrs -g electrs /var/lib/electrs
```

Step 3 - electrs config at `/etc/electrs/config.toml`:

```toml
network = "bitcoin"
db_dir = "/var/lib/electrs/db"
daemon_dir = "/home/bitcoin/.bitcoin"
daemon_rpc_addr = "127.0.0.1:8332"
electrum_rpc_addr = "127.0.0.1:50001"
monitoring_addr = "127.0.0.1:4224"
log_filters = "INFO"
auth = "bitcoin:<rpcpassword>"
```

`daemon_p2p_addr` is gone in v0.12.0 - blocks arrive over REST on
`daemon_rpc_addr`, and leaving the key in the file aborts startup.

Step 4 - systemd unit `/etc/systemd/system/electrs.service`:

```ini
[Unit]
Description=electrs
After=bitcoind.service
Requires=bitcoind.service

[Service]
User=electrs
Group=electrs
ExecStart=/usr/local/bin/electrs --conf /etc/electrs/config.toml
Restart=on-failure
RestartSec=30
LimitNOFILE=65536
ProtectSystem=full
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

`systemctl enable --now electrs`. Upstream measures the initial index at ~2 h
for ~800 GB of `blocks/*.dat` on a 6-core / 32 GB / NVMe host and ~18 h on an
ODROID-HC1 (electrs `doc/usage.md`, July 2026); slower disks dominate the time.

Step 5 - TLS termination via nginx stream module on port 50002:

```nginx
stream {
  upstream electrs { server 127.0.0.1:50001; }
  server {
    listen 50002 ssl;
    ssl_certificate     /etc/letsencrypt/live/node.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/node.example.com/privkey.pem;
    proxy_pass electrs;
  }
}
```

## Worked example

Verify the daemon answers an Electrum protocol request via raw TCP:

```bash
echo '{"jsonrpc":"2.0","method":"server.version","params":["test","1.4"],"id":0}' \
  | nc -q 1 127.0.0.1 50001
```

Expected: `{"jsonrpc":"2.0","result":["electrs/0.12.0","1.4"],"id":0}` - `1.4`
is the only protocol version electrs implements.

Probe Prometheus metrics during a sync:

```bash
curl -s http://127.0.0.1:4224/metrics | grep electrs_mempool_txs_count
```

`electrs_index_height` is gone in v0.12.0 - indexing moved into `bindex`, and
progress now shows only in the `bindex::chain` log lines. What remains:
`electrs_mempool_txs_count`, `electrs_mempool_txs_vsize`,
`electrs_rpc_duration`, `electrs_server_batch_size`,
`electrs_server_loop_duration`.

Connect Sparrow: Preferences -> Server -> Electrum -> URL `node.example.com`,
port `50002`, SSL on. Sparrow's status bar shows "Connected to electrum
server" and the chain tip in the lower-right.

| Port | Process | Bind | Exposure |
|------|---------|------|----------|
| 8332 | bitcoind RPC | 127.0.0.1 | local |
| 8333 | bitcoind P2P | 0.0.0.0 | public |
| 50001 | electrs | 127.0.0.1 | local |
| 50002 | nginx TLS | 0.0.0.0 | public TLS |
| 4224 | electrs metrics | 127.0.0.1 | local |

## Common pitfalls

- Forgetting `rest=1` on Core, or running Core older than 31.0 - v0.12.0 has
  no P2P fallback and will not be able to fetch blocks at all.
- Upgrading an existing 0.10.x / 0.11.x deployment in place - the bindex format
  in v0.12.0 is new and forces a full reindex, during which electrs cannot
  serve clients; upstream asks for 120 GB free before starting
  (`doc/upgrading.md`, September 2026). v0.11.0 (November 2025) also required
  a reindex, for the rust-rocksdb 0.36 format change.
- Forgetting `peerblockfilters=1` - electrs will index but LDK / btcwallet
  filter-based clients fail to scan against Core.
- Running electrs as root - drop to a dedicated user; the systemd unit above
  enforces this.
- Insufficient `ulimit -n` - electrs opens many RocksDB file handles; lift
  `LimitNOFILE` to 65536+.
- Exposing 50001 without TLS - any sniffer sees address queries; always front
  with nginx or stunnel for remote use.
- Disk full mid-sync - the RocksDB index settles at ~7% of `blocks/*.dat`
  (~56 GB against a ~800 GB chain, July 2026) but peaks near ~14% just before
  the final compaction; budget 1.5 TB for bitcoind + electrs combined and watch
  `df -h /var/lib`.

## References

- electrs install doc: https://github.com/romanz/electrs/blob/master/doc/install.md
- electrs config reference: https://github.com/romanz/electrs/blob/master/doc/config.md
- electrs upgrade notes: https://github.com/romanz/electrs/blob/master/doc/upgrading.md
- electrs release notes: https://github.com/romanz/electrs/blob/master/RELEASE-NOTES.md
- Electrum protocol spec: https://electrumx.readthedocs.io/en/latest/protocol.html
- Bitcoin Core indexes: https://github.com/bitcoin/bitcoin/blob/master/doc/reduce-traffic.md
