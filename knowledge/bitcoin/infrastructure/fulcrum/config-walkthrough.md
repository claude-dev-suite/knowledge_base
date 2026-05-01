# Fulcrum Configuration Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/fulcrum`.
> Canonical source: https://github.com/cculianu/Fulcrum/blob/master/doc/fulcrum-quick-install.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/fulcrum/SKILL.md

## Concept

Fulcrum is a high-throughput Electrum protocol server in C++. It trades RAM
for speed, indexing the chain in roughly half the wall-clock time of electrs
and serving wallet queries with substantially lower p99 latency. This article
covers the production `fulcrum.conf` knobs that matter (cache sizing, worker
threads, fast-sync mode), TLS configuration, an admin interface, and how to
size disk and memory for mainnet vs signet/testnet.

## Walkthrough / mechanics

Install via release tarball or build:

```bash
wget https://github.com/cculianu/Fulcrum/releases/download/v1.11.1/Fulcrum-1.11.1-x86_64-linux.tar.gz
tar -xzf Fulcrum-1.11.1-x86_64-linux.tar.gz
sudo install -m 755 Fulcrum-1.11.1-x86_64-linux/Fulcrum /usr/local/bin/Fulcrum
sudo install -m 755 Fulcrum-1.11.1-x86_64-linux/FulcrumAdmin /usr/local/bin/FulcrumAdmin
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/fulcrum fulcrum
sudo install -d -o fulcrum -g fulcrum /var/lib/fulcrum
```

Generate a self-signed TLS cert (for local use; replace with Let's Encrypt
for public deployments):

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes \
  -keyout /etc/fulcrum/key.pem -out /etc/fulcrum/cert.pem \
  -subj "/CN=node.example.com"
```

`/etc/fulcrum/fulcrum.conf`:

```ini
datadir = /var/lib/fulcrum/db
bitcoind = 127.0.0.1:8332
rpcuser = bitcoin
rpcpassword = <long-random-string>

# Listeners
tcp = 0.0.0.0:50001
ssl = 0.0.0.0:50002
cert = /etc/fulcrum/cert.pem
key = /etc/fulcrum/key.pem

# Performance tuning
fast-sync = 4096          # MiB; raises during initial sync
db_max_open_files = 400
db_mem = 1024             # MiB RocksDB cache
worker_threads = 4
peering = false

# Admin RPC (local only)
admin = 127.0.0.1:8000

# Limits
max_clients_per_ip = 12
max_buffer = 4000000
```

systemd unit `/etc/systemd/system/fulcrum.service`:

```ini
[Unit]
Description=Fulcrum
After=bitcoind.service
Requires=bitcoind.service

[Service]
User=fulcrum
Group=fulcrum
ExecStart=/usr/local/bin/Fulcrum /etc/fulcrum/fulcrum.conf
Restart=on-failure
RestartSec=30
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

For initial sync set `fast-sync = 4096` (or higher if RAM allows); after the
first sync completes you may drop it to `0` to free memory.

## Worked example

Watch sync progress via the admin port:

```bash
FulcrumAdmin -p 8000 stats | head -30
# {
#   "Daemon": { ... "blocks": 868234 },
#   "Storage": { "DB Stats": { ... }, "DB Sizes": { "blkinfo": "12.4 MiB", ... } },
#   "Controller": { "TxNum": 1042138291, "header count (mempool)": 1 }
# }
```

Stop and restart cleanly via admin:

```bash
FulcrumAdmin -p 8000 stop
```

Connect from Sparrow as in the electrs example but on TLS port 50002.
Compare query throughput: a `blockchain.scripthash.get_history` for a
heavily-used scripthash typically returns in <50 ms on Fulcrum vs 200-400 ms
on electrs.

| Setting | Initial sync | Steady state |
|---------|--------------|--------------|
| fast-sync (MiB) | 4096 | 0 |
| db_mem (MiB) | 1024 | 512 |
| worker_threads | cores | 4 |
| Approx RAM | 6-10 GB | 3-5 GB |

## Common pitfalls

- Setting `fast-sync` too high on a 4 GB box causes OOM kills mid-sync;
  keep `fast-sync + db_mem` below 60% of system RAM.
- Self-signed TLS - Sparrow refuses by default; add the cert fingerprint or
  use a real cert, otherwise users see "SSL handshake failed".
- `peering = true` on an unreachable host floods logs with reconnect
  attempts; leave `false` unless you actively want to peer.
- `bitcoind` cookie auth - Fulcrum needs explicit `rpcuser` / `rpcpassword`,
  it does not read `.cookie`.
- Mounting `/var/lib/fulcrum` on slow rotational disk turns a 24h sync into
  a week; always use NVMe or SATA SSD.

## References

- Fulcrum release notes: https://github.com/cculianu/Fulcrum/releases
- Fulcrum example config: https://github.com/cculianu/Fulcrum/blob/master/doc/fulcrum-example-config.conf
- FulcrumAdmin tool: https://github.com/cculianu/Fulcrum/blob/master/doc/fulcrum-admin.md
