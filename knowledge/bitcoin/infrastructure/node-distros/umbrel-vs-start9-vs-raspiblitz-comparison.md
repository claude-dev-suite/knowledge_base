# Umbrel vs Start9 vs RaspiBlitz Comparison - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/node-distros`.
> Canonical source: https://github.com/getumbrel/umbrel | https://start9.com/ | https://raspiblitz.org/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/node-distros/SKILL.md

## Concept

The three flagship self-hosted Bitcoin node distros take noticeably
different stances on architecture, philosophy, and target user. Umbrel is
the "iPhone of node distros": opinionated, slick UX, broad app store,
docker-compose under the hood, and since umbrelOS 1.0 a standalone OS
image rather than a layer on top of an existing Linux. Start9 StartOS
(branded Embassy / EmbassyOS before the rename) is privacy-maximalist:
signed S9PK package manifests, a dedicated immutable OS, no anonymous
telemetry - though since the 0.4.0 rewrite (July 2026) Tor is an
optional plugin rather than the default transport. RaspiBlitz is
the bash-script-and-HDMI distro: SSH/TUI-driven, transparent shell
scripts, runs on a Raspberry Pi with a touchscreen. This article
compares them on installation, app architecture, security model, update
mechanics, and operational concerns.

## Walkthrough / mechanics

Installation (mainnet, similar hardware: Pi 4/5 + 1 TB SSD):

```
Umbrel (Raspberry Pi 5):
  1. Flash the umbrelOS Pi image to an NVMe or USB drive via
     Raspberry Pi Imager; as of September 2026 the README lists
     Pi 5 only and tells you not to boot from microSD.
  2. Boot, get IP from router.
  3. Open http://umbrel.local, set username/password.
  4. Bitcoin Core auto-starts; IBD ~3-5 days on a Pi.

Umbrel (Intel/AMD, VM):
  Install the umbrelOS ISO from umbrel.com/downloads directly onto
  the machine. Since umbrelOS 1.0 there is no install-on-top-of-
  Linux path: as of September 2026 `curl -L https://umbrel.sh | bash`
  only prints a notice pointing at that download page.

Start9 (Raspberry Pi 4):
  1. Flash the StartOS Pi image (0.4.0 ships
     startos-<ver>_raspberrypi.img.gz; x86_64/aarch64/riscv64 get
     .iso installers) to SD or USB.
  2. Boot Pi from USB; OS installs to disk.
  3. Connect via .local mDNS (or a Tor address if the Tor plugin
     is installed).
  4. Web setup wizard, recovery passphrase generated.
  5. Install Bitcoin Core service from marketplace -> IBD.

RaspiBlitz:
  1. Flash raspiblitz.img to SD.
  2. Boot, plug in HDMI + keyboard or SSH.
  3. TUI walks through hostname, password, lightning impl
     (LND or CLN), wallet creation.
  4. IBD via Bitcoin Core or imported from a snapshot.
```

Architecture:

```
Umbrel:
  base OS (umbrelOS) -> docker -> per-app container
  apps installed via JSON manifest from umbrel-apps repo
  Tor v3 hidden service per app, but only after the opt-in
    "Remote Tor access" switch is turned on

Start9 (StartOS 0.4.0, July 2026):
  immutable base OS (S9PK packages with signed manifests)
  per-service LXC container with declared deps
    (LXC replaced Docker/Podman in 0.4.0)
  clearnet-capable networking stack: LAN port forwarding,
    WireGuard gateways, private/public domains, Let's Encrypt,
    built-in DNS; Tor is an optional plugin
  signed binaries, reproducible builds

RaspiBlitz:
  Raspberry Pi OS Lite + bash scripts
  services as systemd units (bitcoind, lnd, electrs, btcpay)
  optional Docker for select apps
  SSH-first, Web UI optional
```

App ecosystem snapshot (mainnet apps available, both BTC- and
LN-related). Checked on 16 September 2026 against the
`getumbrel/umbrel-apps` store repo, the `Start9Labs` and
`Start9-Community` `*-startos` package repos, and RaspiBlitz's
`00settingsMenuServices.sh` on the `dev` branch:

| App | Umbrel | Start9 | RaspiBlitz |
|-----|--------|--------|------------|
| Bitcoin Core | yes | yes | yes |
| LND | yes | yes | yes |
| Core Lightning (CLN) | yes | yes | yes |
| Eclair | no | yes | no |
| BTCPay Server | yes | yes | yes |
| mempool.space | yes | yes | yes |
| Sparrow Server | no | no | no |
| Specter Desktop | yes | community | yes |
| Electrs / Fulcrum | yes | yes | yes |
| RTL (web LN) | yes | yes | yes |
| ThunderHub | yes | community | yes |
| JoinMarket / Jam | yes (Jam) | yes (Jam v2) | yes (built-in) |
| Sphinx Relay | legacy | no | no |
| Filebrowser / Nextcloud | yes | yes | no |

"community" means the package lives in the `Start9-Community` registry
rather than Start9's own. Cells worth spelling out, all as of September
2026:

- Eclair is a first-party Start9 package (`Start9Labs/eclair-startos`,
  still being pushed to in September 2026) with no counterpart in the
  Umbrel store or the RaspiBlitz menu.
- Sparrow is a desktop wallet and none of the three ship a server
  package for it; point Sparrow at the node's own Electrum server
  instead.
- Sphinx Relay is still installable on Umbrel (`sphinx-relay` 2.5.2),
  but upstream `stakwork/sphinx-relay` has had no push since July 2024
  and the Umbrel package was last version-bumped in October 2024 -
  treat it as legacy rather than a current messaging option.
  RaspiBlitz still carries a `bonus.sphinxrelay.sh` script but no
  longer offers it from the SERVICES menu.
- JoinMarket is first-party on both Umbrel (`jam`) and Start9
  (`jam-v2-startos`); RaspiBlitz ships both the JoininBox TUI and Jam.
- RaspiBlitz has no Filebrowser or Nextcloud service - Nextcloud appears
  only as an upload destination for backups (`nextcloud.upload.sh`).

Update mechanics:

- Umbrel: app store push; one-click updates per app; OS updates via
  background service.
- Start9: marketplace push with signed manifest; user must approve each
  update; dependency-aware (won't break a service whose dep is at
  incompatible version).
- RaspiBlitz: `sudo raspiblitz` opens TUI, "UPDATE" menu; per-service
  updates often pinned to known-good versions.

Networking defaults:

- Umbrel: Tor is **off** by default. It is an opt-in switch at
  Settings -> Advanced -> "Remote Tor access"; umbreld writes
  `torEnabled: false` into its store on first start (checked in
  `getumbrel/umbrel` at tag 1.7.4, July 2026, and on `master`,
  September 2026). Out of the box apps are reachable over HTTPS on
  the LAN via `umbrel.local` with a self-signed cert.
- Start9: up to StartOS 0.3.x, Tor on by default for everything with
  clearnet enabled explicitly per service. From 0.4.0 (July 2026) the
  stack is clearnet-first - LAN, WireGuard gateways, private/public
  domains with Let's Encrypt certs, plus the StartTunnel reverse tunnel
  to publish a service on a public domain without exposing the home IP;
  Tor is installed as an optional plugin.
- RaspiBlitz: Tor is **on** by default - a fresh setup writes
  `runBehindTor='on'` into `raspiblitz.conf` and provisioning then
  runs `tor.network.sh on`, so bitcoind and LND talk over Tor from
  the start (checked in `_bootstrap.sh` at v1.12.1 and on `dev`,
  September 2026); SSH stays the primary admin path and the TUI menu
  toggles Tor afterwards.

## Worked example

Compare a typical day-1 install of BTCPay Server on each:

```
Umbrel:
  App Store -> BTCPay Server -> Install
  ~5 min, auto-configured to local bitcoind + LND
  Access at https://btcpay.local; a Tor onion only once
    "Remote Tor access" has been switched on

Start9:
  Marketplace -> BTCPay Server -> Install
  Configure: bitcoind dependency selected automatically
  0.4.0: reachable on LAN by default; a public or private domain
    (or StartTunnel) must be configured; an onion address requires
    the Tor plugin
  ~7 min

RaspiBlitz:
  ssh admin@raspiblitz; main menu -> SERVICES -> BTCPay Server
  Select Lightning impl + DNS hostname
  Lets-Encrypt cert prompt
  ~15-20 min (compiles some pieces from source)
```

Hardware sizing (full sync + BTCPay + mempool.space + RTL). None of
the three projects publishes measured benchmarks, so the rows below
are community rules of thumb with no primary source behind them -
read them as order-of-magnitude planning figures, not measurements:

| Resource | Umbrel (Pi 5) | Start9 (Server One)  | RaspiBlitz (Pi 5) |
|----------|---------------|----------------------|--------------------|
| CPU steady | 25-40% | 20-35% | 30-45% |
| RAM | ~4 GB | ~3.5 GB | ~3.5 GB |
| Idle power | 8 W | 12 W | 8 W |
| First-boot to fully synced | 3-5 days | 4-6 days | 3-5 days |

Disk after sync is not a per-distro property - all three store the
same chain. Block data alone was 768,755 MB (~769 GB, ~716 GiB) on
15 September 2026 and was growing ~241 MB/day (~88 GB/year) over the
preceding week, per blockchain.com's Blockchain Size chart;
chainstate sits on top of that, and an Electrum index (electrs or
Fulcrum) or `txindex` adds more again. 1 TB is the practical floor
for an unpruned node as of September 2026, and at that growth rate a
1 TB drive is a two-year decision, not a permanent one.

## Common pitfalls

- microSD wear - all three benefit from booting from SSD; long-term
  microSD use causes silent corruption in 6-18 months under continual
  write load.
- ISP NAT - inbound P2P / LN ports blocked behind CGNAT; use Tor (as
  of September 2026 opt-in on Umbrel via a Settings switch and on
  StartOS 0.4.0+ via a plugin, already on by default on RaspiBlitz),
  StartTunnel on Start9, or rent a VPS with port-forwarded reverse
  SSH.
- Assuming Umbrel gives you onion addresses - it does not until you
  turn on Settings -> Advanced -> "Remote Tor access". A
  privacy-motivated reader who skips that switch is running a
  clearnet-only node. (This also supersedes the old "x86 mode misses
  Tor" caveat: the install-on-top-of-Ubuntu path it referred to was
  retired with umbrelOS 1.0, and Tor is now the same opt-in on every
  umbrelOS install target.)
- StartOS 0.3.x -> 0.4.0 upgrade - 0.4.0 (July 2026) is a rewrite, not a
  point release: Start9 documents it as an update "between two
  essentially distinct operating systems", pre-0.4.0 backups are
  incompatible, and the only supported path is the published 0.4.0
  update guide. Update every service and take a fresh backup
  immediately after.
- Start9 marketplace approval - some FOSS apps lag behind the latest
  upstream because each release must be manually packaged and signed;
  power users may diverge from the curated catalog.
- RaspiBlitz custom builds - the menu compiles binaries from source for
  some apps; failed builds (e.g., npm module mismatch) leave a partial
  install; rerun the install action to recover.
- Cross-distro migration - moving from Umbrel to Start9 (or back) is not
  a "restore from backup" operation; expect to re-IBD or import a
  blockchain snapshot, then restore wallets/channels separately.
- Family member access - as of September 2026 all three still default
  to single-admin, with multi-user bolted on via Nextcloud / Bitwarden
  apps; the exception is umbrelOS 2.0.0-beta.1 (September 2026, beta
  channel only), which adds native per-user accounts each with their
  own home screen and private home folder.

## References

- Umbrel: https://umbrel.com/ | https://github.com/getumbrel/umbrel
- Start9: https://start9.com/ | https://docs.start9.com/ |
  https://github.com/Start9Labs/start-technologies (monorepo for
  StartOS, StartWRT, StartTunnel, start-sdk, start-cli, start-registry)
- StartOS 0.4.0 release notes / update guide:
  https://github.com/Start9Labs/start-technologies/releases/tag/start-os/v0.4.0 |
  https://docs.start9.com/start-os/0.4.0.x/update-040.html
- RaspiBlitz: https://raspiblitz.org/ | https://github.com/raspiblitz/raspiblitz
- App-matrix sources: https://github.com/getumbrel/umbrel-apps |
  https://github.com/orgs/Start9Labs/repositories |
  https://github.com/orgs/Start9-Community/repositories |
  https://github.com/raspiblitz/raspiblitz/blob/dev/home.admin/00settingsMenuServices.sh
- umbrelOS Tor default (`torEnabled: false` on first start):
  https://github.com/getumbrel/umbrel/blob/1.7.4/packages/umbreld/source/modules/apps/apps.ts
- Chain size series: https://api.blockchain.info/charts/blocks-size?format=json
- RaspiBlitz Tor default (`runBehindTor='on'` on fresh setup):
  https://github.com/raspiblitz/raspiblitz/blob/v1.12.1/home.admin/_bootstrap.sh
- Comparison thread: https://bitcoiner.guide/node/
