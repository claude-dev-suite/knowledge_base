# Initial Block Download Tuning - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/operations`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/doc/reduce-traffic.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/operations/SKILL.md

## Concept

Initial Block Download (IBD) is dominated by three resources: disk write throughput, RAM available for the UTXO cache, and CPU for script verification. Default settings are tuned for a low-spec home machine and leave a factor of three to five performance on the table for any modern server. The goal of IBD tuning is to maximize the time the validation thread spends doing CPU work and minimize the time it spends flushing the chainstate to disk. The single most effective lever is `dbcache=`; the second is using assumeutxo when available; the third is making sure the device under `chainstate/` is an SSD with enough random-write IOPS.

## Walkthrough / mechanics

IBD splits cleanly into three phases. The headers phase fetches ~800k 80-byte headers from a single peer and is bandwidth-trivial. The block download phase fetches ~700 GB of block data from up to `-blocksonly=0` configurable peers in parallel. The validation phase processes those blocks against the UTXO set; this is where almost all wall-clock time goes.

The chainstate is a LevelDB of ~10 GB on disk plus an in-memory cache (`dbcache`). When the cache fills, Core does a "flush" that writes dirty entries back to LevelDB. Each flush takes seconds to minutes depending on disk speed. Smaller `dbcache` triggers more flushes. With `dbcache=450` (default), a fast machine spends roughly half its IBD time flushing. With `dbcache=8000` on a 16 GB machine, flushes happen perhaps four times during the entire IBD.

`assumevalid=<hash>` skips ECDSA signature verification up to the supplied block, trusting that the hardcoded value in the source matches a real, deeply-buried block. This is on by default and saves hours. `assumeutxo` (newer flag) goes further: load a committed UTXO snapshot at a recent height, jump directly to that height, then validate the rest. The snapshot is verified against a hash hardcoded in the binary, so only Core developers can compromise it; the same trust assumption you already made by running the binary.

Other levers: `-par=N` sets the number of script verification threads (defaults to physical cores). `-blocksdir=` and `-datadir=` separate block storage from chainstate so a fast NVMe holds the chainstate while bulk blocks live on cheaper storage.

## Worked example

A mid-range server with 16 GB RAM and an NVMe SSD. Pre-IBD config:

```ini
# /etc/bitcoin/bitcoin.conf
datadir=/var/lib/bitcoind
blocksdir=/var/lib/bitcoind-blocks

dbcache=8000
par=8
maxmempool=300

# IBD optimizations
assumevalid=000000000000000000020c2... # current default works, but pin to be explicit
maxconnections=40
```

Start and watch progress:

```bash
$ bitcoind -daemon
$ tail -f /var/lib/bitcoind/debug.log | grep -E 'UpdateTip|progress=|Flushed'
2026-04-29T12:03:11Z UpdateTip: new best=00000000... height=100000 progress=0.005000
2026-04-29T13:21:04Z UpdateTip: new best=00000000... height=400000 progress=0.067 cache=2.1MiB(13000txo)
2026-04-29T15:48:30Z Flushed 9128471 changes to dbcache (dirty=0/...) in 14237 ms
```

If you have a recent assumeutxo snapshot file (released by core dev or a trusted builder), you can load it instead:

```bash
# Download the .dat snapshot first (verify hash against bitcoincore.org)
$ bitcoin-cli loadtxoutset /tmp/utxo-840000.dat
{"coins_loaded": 158234567, "tip_hash": "...", "base_height": 840000, "path": "..."}
```

After it returns, the node is fully usable at height 840000 within a few minutes. Background validation continues to backfill from genesis and will eventually overtake.

Post-IBD, lower `dbcache` to free RAM:

```bash
$ bitcoin-cli stop
# edit conf: dbcache=450
$ bitcoind -daemon
```

## Common pitfalls

- Setting `dbcache=8000` on a 4 GB box: the OS swaps under load, validation grinds, IBD takes longer than with `dbcache=450`. Never exceed roughly half of physical RAM minus 1 GB for OS.
- Using a USB-attached HDD for `chainstate/`. Random-write IOPS, not sequential throughput, dominate. A 5400 RPM HDD turns a 24-hour IBD into a 7-day IBD regardless of dbcache.
- Forgetting that `dbcache` is a hint, not a hard cap. Core can briefly use more during a flush. Leave at least 1 GB of RAM headroom.
- Running the wallet during IBD. `bitcoind` does not block, but every received block triggers a wallet rescan if a watch descriptor was imported with an old timestamp; this slows IBD by 10x. Import descriptors with `timestamp: "now"` or after IBD completes.
- Bind-mounting `blocks/` and `chainstate/` to different filesystems with different cache pressure. Keep them under one mount to give the kernel page cache uniform behavior, unless you have measured the alternative.
- Treating `assumevalid=0` as harder security. It only disables signature checks for a finite range of historical blocks the source already trusts; setting it to zero forces full verification of all history and adds many hours, with no security benefit on a verified binary.

## References

- `doc/reduce-traffic.md` in bitcoin/bitcoin.
- `doc/assumeutxo.md` in bitcoin/bitcoin.
- `src/validation.cpp` for `FlushStateMode` and dbcache flush logic.
- `src/init.cpp` for default sizing of `-dbcache` and `-par`.
