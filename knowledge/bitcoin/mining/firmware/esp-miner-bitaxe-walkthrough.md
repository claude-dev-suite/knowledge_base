# ESP-Miner / AxeOS (Bitaxe) Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/mining/firmware`.
> Canonical source: <https://github.com/bitaxeorg/ESP-Miner> and the per-model
> hardware repos under <https://github.com/bitaxeorg>
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/mining/firmware/SKILL.md

## Concept

The **Bitaxe** is fully open-source Bitcoin mining hardware: an ESP32-S3
microcontroller, a buck regulator, a current DAC for core voltage, a
power meter, a PWM fan controller, and one (or a few) Bitmain SHA-256
ASIC(s) harvested from a modern Antminer hashboard. The ESP32 is the
controller - it does not hash. That distinguishes Bitaxe from the
ESP32-only NerdMiner class, where the microcontroller itself hashes in
the KH/s range.

The firmware is **ESP-Miner**, whose web dashboard is branded **AxeOS**.
It is maintained by Open Source Miners United (OSMU) at
`github.com/bitaxeorg/ESP-Miner`; the latest release as of this writing
is v2.15.1 (29 August 2026). The design goal is an appliance: plug in
5 V, join Wi-Fi from a phone, point it at a pool, done.

Unlike Braiins OS+ or Vnish - which *replace* a manufacturer's stock
firmware on an existing Antminer - ESP-Miner is the only firmware the
hardware has ever had, and the board schematics ship alongside it.

## Hardware models

Design files live in per-model repos under `github.com/bitaxeorg`. As of
September 2026:

| Board | ASIC | Harvested from | Vendor efficiency claim |
|---|---|---|---|
| Max | BM1397 | Antminer S17 | - |
| Ultra | BM1366 | Antminer S19 XP | 0.021 J/GH (21 J/TH) |
| Supra | BM1368 | Antminer S21 | 17.5 J/TH |
| Gamma | BM1370 | Antminer S21 Pro | 15 J/TH |
| Gamma Turbo (GT) | 2x BM1370 | Antminer S21 Pro | - |
| Gamma Hex | 6x BM1370 | Antminer S21 Pro | - |

The efficiency column is Bitmain's claim for the bare chip, quoted by
each hardware repo; a Bitaxe gets "pretty close" to it (the project's
own wording for the Gamma) because it carries no hashboard-level
overhead but also no hashboard-level cooling.

Hashrate follows directly from chip count. An Antminer S21 Pro is
nominally 234 TH/s across 3 hashboards of 65 chips = 195 BM1370s, so a
single-chip Gamma is `234 / 195 ~= 1.2 TH/s`. A GT with two chips is
about double that, a Gamma Hex about six times.

Other boards in the same org explore non-Bitmain or multi-chip silicon:
`bitaxeAura` (two Auradine Treasure ASICs), `bitaxeBonanza` (Intel BZM2),
`bitaxeNaja` (dual BM1340, 12 V).

### Power and cooling

- **5 V DC only.** Anything else destroys the board. The Ultra and
  Supra take a 5.5x2.5 mm center-positive barrel jack; the Gamma is
  5.5x2.1 mm (its repo notes 5.5x2.5 mm plugs also work).
- An Ultra needs ~15 W. A Gamma needs a PSU that holds 5 V past 4 A, so
  in practice a 25-30 W supply.
- Active cooling is mandatory - a 40 mm 5 V 4-pin PWM fan on a 40x40 mm
  heatsink, with real thermal compound. The BM1366 has no die temp
  sensor; the board places an EMC2101 next to the ASIC and uses its
  internal sensor as a proxy.

## Walkthrough / mechanics

### Flashing

Two paths, both on the `bitaxeorg` org:

1. **Bitaxe Web Flasher** (`bitaxeorg/bitaxe-web-flasher`) - flashes from
   the browser over the board's USB-C port. Easiest.
2. **bitaxetool** - a Python CLI:

```bash
pip install --upgrade bitaxetool

# factory image - must match the BOARD VERSION, not just the model name
bitaxetool --firmware ./esp-miner-factory-401-v2.4.2.bin

# NVS config only
bitaxetool --config ./configs/config-401.csv

# both; the CSV overrides the config baked into the factory image
bitaxetool --config ./configs/config-401.csv \
           --firmware ./esp-miner-factory-401-v2.4.2.bin
```

`bitaxetool` is pinned against `esptool` v4.9.0 - it does not work with
esptool v5.x. Version 0.6.1 of the tool locks that dependency.

Since **v2.15.0 (August 2026)** the AxeOS web bundle is compiled into
`esp-miner.bin`; earlier versions required uploading a separate
`www.bin` alongside it. That release also moved the project to
ESP-IDF 6.0.2 and added mDNS, so the dashboard is reachable at the
miner's hostname rather than only its IP.

### Pool configuration and TLS

Pools are configured from AxeOS. ESP-Miner speaks Stratum V1 natively
and supports **per-pool TLS**: each pool entry carries a `tls_mode` and
an optional custom CA certificate, and the V1 task builds an
`esp_transport_ssl` handle with the pool hostname as the expected common
name. A fallback pool is a separate entry; since v2.15.0 the fallback is
*disabled* by default rather than silently on.

### Stratum V2

SV2 support landed in **v2.14.0 (June 2026)** as a dedicated
`components/stratum_v2` component (`sv2_noise.c` for the Noise handshake,
`sv2_protocol.c` for framing) plus a `stratum_v2_task`, sitting beside
the V1 task under a shared `protocol_coordinator`.

**v2.15.0 (August 2026)** hardened it:

- SV2 frame buffer raised to 8 KiB.
- SV2 frames written in a single socket write.
- Minimum SV2 extranonce size reduced from 6 to 2 bytes.
- Fractional pool difficulty preserved, to stop "difficulty too low"
  rejects.
- Per-pool "require authentication" toggle.
- Pending SV2 shares surfaced on the dashboard.

So a 2026 Bitaxe can talk to an SV2 pool or a local SV2 job-declaration
setup, not just V1 - which matters because the same box is often the
demo hardware for a home SV2 stack.

### API

AxeOS exposes a REST + WebSocket API (OpenAPI spec at
`main/http_server/openapi.yaml` in the repo):

```bash
curl http://bitaxe.local/api/system/info          # system information
curl http://bitaxe.local/api/system/asic          # ASIC settings
curl http://bitaxe.local/api/system/statistics    # logged statistics
curl http://bitaxe.local/api/system/scoreboard    # top 20 best shares
```

`POST /api/system/restart`, `/api/system/identify` and `/api/system/OTA`
handle lifecycle and firmware upload; `PATCH /api/system` writes
settings. `/api/ws` is a text log stream and `/api/ws/live` a JSON
stream of partial system-info updates - the latter is what the
dashboard charts consume.

### Tuning

There is no Braiins-style per-chip autotuner. Frequency and core voltage
are user settings in AxeOS, and the community convention is to sweep
them with an external benchmark script (e.g.
`n4rr0w-87/nerdqaxe-benchmark`, `SerpentXSF/NerdQAxe_hashrate_benchmark`
for the NerdQAxe forks) and pick the J/TH knee. The firmware does run a
configurable self-test with adjustable warm-up temperature and fan
speed, and since v2.15.0 falls back to the default frequency during
self-test as well.

## Worked example

A home operator wants a quiet solo-mining box.

1. Buy an assembled Gamma from a seller on the project's own
   `legitlist` (`bitaxeorg/legitlist`, which backs `bitaxe.org/legit.html`)
   rather than a generic marketplace clone.
2. Flash the current release with the web flasher; confirm the board
   version in the image name matches the silkscreen (a 6xx Gamma image,
   not a 401 Ultra image).
3. Join Wi-Fi from the AxeOS setup flow, then browse to the miner's
   mDNS hostname.
4. Point it at a zero-fee solo pool. Set the payout address as the
   Stratum user.
5. Leave frequency/voltage at stock for a week, watch
   `/api/system/statistics`, then sweep upward in small steps while
   watching the hardware-error counter and chip temperature.

Expectation setting matters more than tuning. ~1.2 TH/s against a
network difficulty of roughly 1.34e14 (block 957,382, July 2026) is a
lottery ticket, not an income stream - the expected time to a solo block
at that hashrate is on the order of fifteen thousand years
(~958 EH/s network hashrate at that difficulty).

Which is exactly why the outlier is famous: on **10 July 2026** a solo
miner on Public Pool found block **957,382** - coinbase tagged
`Public-Pool`, a single payout of 3.138 BTC (3.125 subsidy plus fees) to
one address - and it was widely reported as having been found by a
Bitaxe.

## Common pitfalls

- **Wrong board-version image** - images are keyed to board revision
  (401, 6xx, ...), not to the marketing model name. The wrong image
  boots with wrong voltage/frequency defaults.
- **esptool version** - `bitaxetool` breaks against esptool 5.x; pin
  `bitaxetool==0.6.1` or install esptool <= 4.9.0.
- **Under-rated PSU** - many "5 V 5 A" supplies sag below 5 V under
  real load. Sag shows up as hardware errors, not as an obvious power
  fault.
- **12 V fan on a 5 V header** - it spins, slowly, and the board cooks.
- **Confusing Bitaxe with NerdMiner** - NerdMiner is ESP32-only, KH/s,
  no ASIC. NerdQAxe/NerdQAxe+ *are* ASIC boards but run a fork of
  ESP-Miner (`shufps/ESP-Miner-NerdQAxePlus`), so upstream release notes
  do not automatically apply to them.
- **Expecting an autotuner** - unlike Braiins OS+, ESP-Miner does not
  hunt the J/TH optimum for you.
- **Dashboard on the WAN** - AxeOS exposes restart, OTA and settings
  writes over plain HTTP on the LAN. Keep it off the public internet or
  behind a VPN.

## References

- ESP-Miner firmware <https://github.com/bitaxeorg/ESP-Miner>
- ESP-Miner releases <https://github.com/bitaxeorg/ESP-Miner/releases>
- Hardware repos <https://github.com/bitaxeorg>
- Bitaxe Gamma hardware <https://github.com/bitaxeorg/bitaxeGamma>
- Project page <https://bitaxe.org>
- Block 957,382 <https://mempool.space/block/957382>
- Skill: `bitcoin/mining/stratum-v2`, `bitcoin/mining/decentralized-pools`
