# Fulcrum Configuration Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/fulcrum`.
> Canonical source: https://github.com/cculianu/Fulcrum/blob/master/doc/fulcrum-quick-config.conf
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/fulcrum/SKILL.md

## Concept

Fulcrum is a high-throughput Electrum protocol server in C++. It trades RAM
for speed, serving wallet queries with substantially lower p99 latency than
electrs. Initial sync is no longer where it wins: upstream electrs v0.12.0
(13 September 2026) reports a ~2h mainnet index build via the `bindex` library
over bitcoind's REST interface on a 6-core / 32 GB RAM / NVMe box (July 2026),
against the 4 to 20+ hours Fulcrum's own README quotes for a mainnet sync on
SSD (a figure carried unchanged in that README since v1.0, January 2020).
This article covers the production `fulcrum.conf` knobs that matter (RocksDB
memory sizing, worker threads), TLS configuration, an admin interface, and
how to size disk and memory for mainnet vs signet/testnet.

## Walkthrough / mechanics

Install via release tarball or build:

```bash
# v2.1.2, released 20 August 2026 - latest release as of September 2026
wget https://github.com/cculianu/Fulcrum/releases/download/v2.1.2/Fulcrum-2.1.2-x86_64-linux.tar.gz
tar -xzf Fulcrum-2.1.2-x86_64-linux.tar.gz
sudo install -m 755 Fulcrum-2.1.2-x86_64-linux/Fulcrum /usr/local/bin/Fulcrum
sudo install -m 755 Fulcrum-2.1.2-x86_64-linux/FulcrumAdmin /usr/local/bin/FulcrumAdmin
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/fulcrum fulcrum
sudo install -d -o fulcrum -g fulcrum /var/lib/fulcrum
```

Fulcrum 2.0.0 (23 September 2025) redid the database format. A 2.x datadir is
crash-safe - the process may be killed, or lose power, at any moment without
corrupting the DB; at worst the last block is rolled back and re-synced - and
the on-disk format is byte-order-neutral, so a datadir can be copied between
Linux, Windows, macOS and BSD and across architectures. A 1.x datadir is not
readable by 2.x: either re-sync from block 0, or run Fulcrum 2.x once with the
one-time `--db-upgrade` flag (around an hour or more on BTC mainnet). Fulcrum
refuses to start on an old-format datadir unless that flag is given, and the
upgrade itself is the one operation that must not be interrupted - killing it
loses both the 1.x and the 2.x database.

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

# Performance tuning (2.x defaults: db_max_open_files 1000, db_mem 2048)
db_max_open_files = 1000
db_mem = 4096             # MiB RocksDB cache; lower after initial sync
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

`fast-sync` and `utxo_cache` no longer exist: Fulcrum 2.0.0 removed the UTXO
cache outright as incompatible with the new consistency guarantees, and 2.x
aborts at startup with "The UTXO cache facility has been removed, use `db_mem`
instead!" if either key is present in the conf file or on the CLI. The
replacement is `db_mem` - raise it for the initial sync, then stop Fulcrum and
set it back down for steady-state operation.

## Worked example

Watch sync progress via the admin port:

```bash
FulcrumAdmin -p 8000 getinfo | head -30
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

Fulcrum 2.1.1 (7 May 2026) added `blockchain.address.get_status` and
`blockchain.scripthash.get_status`, which return a scripthash status without
opening a subscription - useful for polling clients that would otherwise
accumulate server-side subscription state.

| Setting | Initial sync | Steady state |
|---------|--------------|--------------|
| db_mem (MiB) | 4096 | 2048 |
| worker_threads | cores | 4 |
| Approx RAM | 6-10 GB | 3-5 GB |

## Common pitfalls

- Over-sizing `db_mem` on a small box causes OOM kills mid-sync; upstream
  advises against values in excess of 50% of system RAM (even 256 MiB works
  acceptably on an SSD).
- Self-signed TLS - Sparrow refuses by default; add the cert fingerprint or
  use a real cert, otherwise users see "SSL handshake failed".
- `peering = true` on an unreachable host floods logs with reconnect
  attempts; leave `false` unless you actively want to peer.
- `bitcoind` cookie auth - Fulcrum needs explicit `rpcuser` / `rpcpassword`,
  it does not read `.cookie`.
- Mounting `/var/lib/fulcrum` on slow rotational disk turns a sub-day sync
  into days - upstream's own estimate for HDD mainnet, which it has never
  tested; always use NVMe or SATA SSD.
- Carrying a 1.x `fulcrum.conf` straight to 2.x - besides `fast-sync`, the
  defaults moved (`db_max_open_files` 40 -> 1000, `db_mem` -> 2048 MiB or 25%
  of physical RAM, whichever is smaller), so re-read the shipped example conf.

## References

- Fulcrum release notes: https://github.com/cculianu/Fulcrum/releases
- Fulcrum 2.0.0 release notes (DB format rewrite, 23 Sep 2025): https://github.com/cculianu/Fulcrum/releases/tag/v2.0.0
- Fulcrum 2.1.1 release notes (`get_status` RPC, 7 May 2026): https://github.com/cculianu/Fulcrum/releases/tag/v2.1.1
- Fulcrum example config: https://github.com/cculianu/Fulcrum/blob/master/doc/fulcrum-example-config.conf
- FulcrumAdmin tool: https://github.com/cculianu/Fulcrum/blob/master/README.md#admin-script-fulcrumadmin
