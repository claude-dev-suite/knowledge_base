# Electrs Deployment Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/electrs`.
> Canonical source: https://github.com/romanz/electrs/blob/master/doc/install.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/electrs/SKILL.md

## Concept

This article walks an end-to-end production deployment of romanz/electrs
on a Linux server with a synced Bitcoin Core node. It covers the prerequisite
`bitcoin.conf` flags, building electrs from source with cargo, hardening with
a systemd service, and exposing the Electrum protocol over TLS via nginx so
remote wallets (Sparrow, Electrum, BlueWallet) can connect securely. The goal
is a single-host node where bitcoind, electrs, and a TLS terminator run as
separate services with the principle of least privilege.

## Walkthrough / mechanics

Step 1 - Bitcoin Core configuration. Edit `/home/bitcoin/.bitcoin/bitcoin.conf`:

```ini
server=1
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

Step 2 - Build electrs. As a non-root user with rust installed:

```bash
git clone https://github.com/romanz/electrs
cd electrs
git checkout v0.10.7
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
daemon_p2p_addr = "127.0.0.1:8333"
electrum_rpc_addr = "127.0.0.1:50001"
monitoring_addr = "127.0.0.1:4224"
log_filters = "INFO"
auth = "bitcoin:<rpcpassword>"
```

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

`systemctl enable --now electrs`. Initial sync takes 12-36h depending on disk
speed (NVMe ideal).

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

Expected: `{"jsonrpc":"2.0","result":["electrs/0.10.7","1.4"],"id":0}`.

Probe Prometheus metrics during a sync:

```bash
curl -s http://127.0.0.1:4224/metrics | grep electrs_index_height
# electrs_index_height 868234
```

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

- Forgetting `peerblockfilters=1` - electrs will index but LDK / btcwallet
  filter-based clients fail to scan.
- Running electrs as root - drop to a dedicated user; the systemd unit above
  enforces this.
- Insufficient `ulimit -n` - electrs opens many RocksDB file handles; lift
  `LimitNOFILE` to 65536+.
- Exposing 50001 without TLS - any sniffer sees address queries; always front
  with nginx or stunnel for remote use.
- Disk full mid-sync - the RocksDB index alone is 6-10 GB; budget 1.5 TB for
  bitcoind + electrs combined and watch `df -h /var/lib`.

## References

- electrs install doc: https://github.com/romanz/electrs/blob/master/doc/install.md
- electrs config reference: https://github.com/romanz/electrs/blob/master/doc/config.md
- Electrum protocol spec: https://electrumx.readthedocs.io/en/latest/protocol.html
- Bitcoin Core indexes: https://github.com/bitcoin/bitcoin/blob/master/doc/reduce-traffic.md
