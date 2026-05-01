# Umbrel vs Start9 vs RaspiBlitz Comparison - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/infrastructure/node-distros`.
> Canonical source: https://github.com/getumbrel/umbrel | https://start9.com/ | https://raspiblitz.org/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/infrastructure/node-distros/SKILL.md

## Concept

The three flagship self-hosted Bitcoin node distros take noticeably
different stances on architecture, philosophy, and target user. Umbrel is
the "iPhone of node distros": opinionated, slick UX, broad app store,
docker-compose under the hood, optional self-hosted x86 mode. Start9
EmbassyOS is privacy-maximalist: Tor-by-default, signed package
manifests, dedicated immutable OS, no anonymous telemetry. RaspiBlitz is
the bash-script-and-HDMI distro: SSH/TUI-driven, transparent shell
scripts, runs on a Raspberry Pi with a touchscreen. This article
compares them on installation, app architecture, security model, update
mechanics, and operational concerns.

## Walkthrough / mechanics

Installation (mainnet, similar hardware: Pi 4/5 + 1 TB SSD):

```
Umbrel (Pi):
  1. Flash umbrelOS image to SD card via Raspberry Pi Imager.
  2. Boot, get IP from router.
  3. Open http://umbrel.local, set username/password.
  4. Bitcoin Core auto-starts; IBD ~3-5 days on Pi 4.

Umbrel (x86 Linux installer):
  curl -L https://umbrel.sh | bash
  - installs umbrelOS components onto existing Ubuntu/Debian.

Start9 (Raspberry Pi 4):
  1. Flash startos.img to USB drive.
  2. Boot Pi from USB; OS installs to disk.
  3. Connect via .local mDNS or Tor address.
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
  Tor v3 hidden service per app

Start9 (EmbassyOS):
  immutable base OS (S9pk packages with signed manifests)
  per-service container with declared deps
  Tor by default, optional clearnet
  signed binaries, reproducible builds

RaspiBlitz:
  Raspberry Pi OS Lite + bash scripts
  services as systemd units (bitcoind, lnd, electrs, btcpay)
  optional Docker for select apps
  SSH-first, Web UI optional
```

App ecosystem snapshot (mainnet apps available, both BTC- and LN-related):

| App | Umbrel | Start9 | RaspiBlitz |
|-----|--------|--------|------------|
| Bitcoin Core | yes | yes | yes |
| LND | yes | yes | yes |
| Core Lightning (CLN) | yes | yes | yes |
| Eclair | yes | no | no |
| BTCPay Server | yes | yes | yes |
| mempool.space | yes | yes | yes |
| Sparrow Server | yes | no | no |
| Specter Desktop | yes | no | yes |
| Electrs / Fulcrum | yes | yes | yes |
| RTL (web LN) | yes | yes | yes |
| ThunderHub | yes | yes | yes |
| JoinMarket | community | community | yes (built-in) |
| Sphinx Chat | yes | no | no |
| Filebrowser / Nextcloud | yes | yes | community |

Update mechanics:

- Umbrel: app store push; one-click updates per app; OS updates via
  background service.
- Start9: marketplace push with signed manifest; user must approve each
  update; dependency-aware (won't break a service whose dep is at
  incompatible version).
- RaspiBlitz: `sudo raspiblitz` opens TUI, "UPDATE" menu; per-service
  updates often pinned to known-good versions.

Networking defaults:

- Umbrel: Tor on by default for app exposure; HTTPS LAN via
  `umbrel.local` with self-signed cert.
- Start9: Tor on by default for everything; clearnet must be explicitly
  enabled per service; signed cert support via UI.
- RaspiBlitz: Tor optional; SSH default; Tor activation via TUI menu.

## Worked example

Compare a typical day-1 install of BTCPay Server on each:

```
Umbrel:
  App Store -> BTCPay Server -> Install
  ~5 min, auto-configured to local bitcoind + LND
  Access at https://btcpay.local or Tor onion

Start9:
  Marketplace -> BTCPay Server -> Install
  Configure: bitcoind dependency selected automatically
  Tor onion auto-published; clearnet via setting
  ~7 min

RaspiBlitz:
  ssh admin@raspiblitz; main menu -> SERVICES -> BTCPay Server
  Select Lightning impl + DNS hostname
  Lets-Encrypt cert prompt
  ~15-20 min (compiles some pieces from source)
```

Hardware sizing benchmarks (full sync + BTCPay + mempool.space + RTL):

| Resource | Umbrel (Pi 5) | Start9 (Embassy Pro) | RaspiBlitz (Pi 5) |
|----------|---------------|----------------------|--------------------|
| CPU steady | 25-40% | 20-35% | 30-45% |
| RAM | ~4 GB | ~3.5 GB | ~3.5 GB |
| Disk after sync | 720 GB | 750 GB | 720 GB |
| Idle power | 8 W | 12 W | 8 W |
| First-boot to fully synced | 3-5 days | 4-6 days | 3-5 days |

## Common pitfalls

- microSD wear - all three benefit from booting from SSD; long-term
  microSD use causes silent corruption in 6-18 months under continual
  write load.
- ISP NAT - inbound P2P / LN ports blocked behind CGNAT; use Tor (default
  on Umbrel/Start9) or rent a VPS with port-forwarded reverse SSH.
- Umbrel "x86 mode" misses Tor - the dotfiles installer on stock Ubuntu
  doesn't auto-configure Tor onion services for apps; verify with
  `tor-resolve` after install.
- Start9 marketplace approval - some FOSS apps lag behind the latest
  upstream because each release must be manually packaged and signed;
  power users may diverge from the curated catalog.
- RaspiBlitz custom builds - the menu compiles binaries from source for
  some apps; failed builds (e.g., npm module mismatch) leave a partial
  install; rerun the install action to recover.
- Cross-distro migration - moving from Umbrel to Start9 (or back) is not
  a "restore from backup" operation; expect to re-IBD or import a
  blockchain snapshot, then restore wallets/channels separately.
- Family member access - all three default to single-admin; multi-user
  is bolted on via Nextcloud / Bitwarden apps, not native.

## References

- Umbrel: https://umbrel.com/ | https://github.com/getumbrel/umbrel
- Start9: https://start9.com/ | https://github.com/Start9Labs
- RaspiBlitz: https://raspiblitz.org/ | https://github.com/raspiblitz/raspiblitz
- Comparison thread: https://bitcoiner.guide/node/
