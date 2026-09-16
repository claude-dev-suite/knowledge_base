# Initial Block Download Tuning - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/core/operations`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/doc/reduce-traffic.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/core/operations/SKILL.md

## Concept

Initial Block Download (IBD) is dominated by three resources: disk write throughput, RAM available for the UTXO cache, and CPU for script verification. Default settings are tuned for a low-spec home machine and leave a factor of three to five performance on the table for any modern server. The goal of IBD tuning is to maximize the time the validation thread spends doing CPU work and minimize the time it spends flushing the chainstate to disk. The single most effective lever is `dbcache=`; the second is using assumeutxo when available; the third is making sure the device under `chainstate/` is an SSD with enough random-write IOPS.

## Walkthrough / mechanics

IBD splits cleanly into three phases. The headers phase fetches ~970k 80-byte headers (height ~967,000 as of September 2026) from a single peer and is bandwidth-trivial. The block download phase fetches ~770 GB of block data from up to `-blocksonly=0` configurable peers in parallel. The validation phase processes those blocks against the UTXO set; this is where almost all wall-clock time goes.

The chainstate is a LevelDB of ~11 GB on disk (measured at height 967,130, 15 September 2026) plus an in-memory cache (`dbcache`). When the cache fills, Core does a "flush" that writes dirty entries back to LevelDB. Each flush takes seconds to minutes depending on disk speed. Smaller `dbcache` triggers more flushes. With `dbcache=450` - the default up to 30.x, and still the default on hosts where less than 4096 MiB of RAM is detected - a fast machine spends roughly half its IBD time flushing. With `dbcache=8000` on a 16 GB machine, flushes happen perhaps four times during the entire IBD.

Bitcoin Core 31.0 (April 2026) raised the `-dbcache` default to 1024 MiB on systems where at least 4096 MiB of RAM is detected (#34692), so a stock 31.x node is already better tuned than a stock 30.x one. 29.0 (April 2025) removed the upper cap on `-dbcache` "due to recent UTXO set growth"; before that, large values were silently reduced to 16 GiB (1 GiB on 32-bit). 30.0 (October 2025) re-introduced caps on 32-bit systems only: `-maxmempool` 500 MB and `-dbcache` 1 GiB.

`assumevalid=<hash>` skips ECDSA signature verification up to the supplied block, trusting that the hardcoded value in the source matches a real, deeply-buried block. This is on by default and saves hours. `assumeutxo` (newer flag) goes further: load a committed UTXO snapshot at a recent height, jump directly to that height, then validate the rest. The snapshot is verified against a hash hardcoded in the binary, so only Core developers can compromise it; the same trust assumption you already made by running the binary. Mainnet snapshot heights in `src/kernel/chainparams.cpp` are 840,000, 880,000, 910,000 and 935,000 as of 31.1 (July 2026); master has since added 965,000, which has not shipped in any release as of September 2026. Each release adds a newer one, so check the source tree at your tag rather than assuming the height a tutorial used - a binary will reject a snapshot whose height it has no hardcoded hash for.

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

After it returns, the node is fully usable at height 840000 within a few minutes. Background validation continues to backfill from genesis and will eventually overtake. 840,000 is still a valid snapshot height, but on 31.x you would normally take the newest one your binary knows (935,000) to shorten the background backfill.

Post-IBD, lower `dbcache` to free RAM - only worth doing if the box is memory-constrained, since 31.0+ already defaults to 1024 MiB rather than 450:

```bash
$ bitcoin-cli stop
# edit conf: dbcache=450
$ bitcoind -daemon
```

## Common pitfalls

- Setting `dbcache=8000` on a 4 GB box: the OS swaps under load, validation grinds, IBD takes longer than with `dbcache=450`. Never exceed roughly half of physical RAM minus 1 GB for OS.
- Running 31.0+ inside a container and leaving `-dbcache` unset. The 1024 MiB default keys off detected RAM, which in a container can exceed the memory actually available under the cgroup limit, leading to out-of-memory conditions. Set `-dbcache` explicitly to a value that fits the limit; `-dbcache=450` restores pre-31.0 behaviour.
- Using a USB-attached HDD for `chainstate/`. Random-write IOPS, not sequential throughput, dominate. A 5400 RPM HDD turns a 24-hour IBD into a 7-day IBD regardless of dbcache.
- Forgetting that `dbcache` is a hint, not a hard cap. Core can briefly use more during a flush. Leave at least 1 GB of RAM headroom.
- Running the wallet during IBD. `bitcoind` does not block, but every received block triggers a wallet rescan if a watch descriptor was imported with an old timestamp; this slows IBD by 10x. Import descriptors with `timestamp: "now"` or after IBD completes.
- Bind-mounting `blocks/` and `chainstate/` to different filesystems with different cache pressure. Keep them under one mount to give the kernel page cache uniform behavior, unless you have measured the alternative.
- Treating `assumevalid=0` as harder security. It only disables signature checks for a finite range of historical blocks the source already trusts; setting it to zero forces full verification of all history and adds many hours, with no security benefit on a verified binary.

## References

- `doc/reduce-traffic.md` in bitcoin/bitcoin.
- `doc/assumeutxo.md` in bitcoin/bitcoin.
- `src/validation.cpp` for `FlushStateMode` and dbcache flush logic.
- `src/init.cpp` and `src/node/caches.h` for default sizing of `-dbcache` and `-par`.
- `src/kernel/chainparams.cpp` for the current list of assumeutxo snapshot heights.
