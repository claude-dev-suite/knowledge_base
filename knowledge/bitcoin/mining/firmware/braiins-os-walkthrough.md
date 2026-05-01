# Braiins OS+ Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/mining/firmware`.
> Canonical source: <https://braiins.com/os/plus> and `braiins/braiins-os` git repo
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/mining/firmware/SKILL.md

## Concept

Braiins OS+ is an open-source-core firmware for Bitmain Antminer ASICs
(S9, S17, S19 family, others) that replaces stock BMMiner with a Linux
distribution running Braiins's Rust-based mining stack. The "+" tier
adds:

- Per-chip autotuning (J/TH optimisation).
- Stratum V2 native support (mining + job declaration + connection
  encryption).
- Web dashboard, JSON API, Prometheus metrics.
- Profile management (fixed-frequency, fixed-power, manual).
- Automatic hashboard recovery and chip remapping.

The base firmware is open source under GPL; the autotuning and SV2
features are gated behind a Braiins Pool fee discount mechanism (the
firmware is free to use; Braiins earns from optional pool fees).

## Walkthrough / mechanics

### Installation

For an Antminer S19 series:

1. Download Braiins OS+ image from `feeds.braiins-os.com`.
2. Apply via ssh + flashing tool, BOSminer "convert" function from
   stock UI, or USB recovery if device is bricked.
3. First boot reformats stock firmware partition; reboot to BOS+.
4. Web UI on `http://<miner_ip>` (HTTP only by default; configure HTTPS).

After install, the miner exposes:
- `:80` HTTP UI
- `:22` SSH
- `:9100` Prometheus exporter (optional)
- Mining client: connects out to configured pool

### Configuration via TOML

`/etc/bosminer.toml` controls behaviour:

```toml
[format]
version = "1.2.0"
model = "Antminer S19 Pro"
generator = "user"
timestamp = 1714003600

[group]
name = "Default"

[[group.pool]]
enabled = true
url = "stratum+tcp://stratum.braiins.com:3333"
user = "bc1q...wallet.s19-1"
password = "x"

[[group.pool]]
enabled = true
url = "stratum2+tcp://v2.stratum.braiins.com:3336/u95GEReVMjK6k5YqiSFNqqTnKU4ypU2Wm8"
user = "bc1q...wallet.s19-1"

[autotuning]
enabled = true
psu_power_limit = 3250

[temp_control]
mode = "auto"
target_temp = 75.0
hot_temp = 85.0
dangerous_temp = 95.0
```

The pool list is failover-ordered. SV2 URLs use `stratum2+tcp://` and
include the pool's noise public key in the path (after the slash) - this
is the SV2 server identity for the Noise XK handshake.

### Autotuning algorithm

Per-chip frequency tuning targets a power-limited optimum:

1. Boot at conservative frequency profile.
2. Sweep frequency per hashboard while monitoring:
   - Hashrate (THS).
   - Hardware error rate (HW errors).
   - Power draw (W).
   - Per-chip temperature.
3. Find the frequency that maximises `THS / W` (efficiency) subject to
   `HW_error_rate < 1%` and `temp < target_temp`.
4. Lock per-chip frequencies in NVRAM.
5. Periodically re-tune as silicon ages or ambient temp changes.

Result: typical 5-15% efficiency improvement over stock firmware on
S19j Pro. Some chips can't hit the target frequency (silicon lottery)
and are clocked down to maintain HW error budget.

### Power and heat profiles

Three modes:

- **Performance** - max hashrate, accept higher J/TH.
- **Balanced** - default, target Bitmain-spec efficiency.
- **Efficiency** - lower frequency, lower J/TH, ~85-90% of nameplate
  hashrate at ~15% lower power.

Custom: set `psu_power_limit` and let autotuner pick the frequency that
maxes hashrate within the cap.

### Stratum V2 integration

When using a `stratum2+tcp://` URL, BOS+:

1. Resolves the pool's noise pubkey from the URL path.
2. Performs Noise XK handshake (ephemeral X25519 ECDH + static auth).
3. Sends `SetupConnection` for protocol `0x00` (Mining Protocol).
4. Receives `OpenStandardMiningChannel` or `OpenExtendedMiningChannel`.
5. Receives `NewExtendedMiningJob` and starts hashing.

If the configured URL is `stratum+tcp://` (V1), BOS+ falls back to plain
JSON-RPC with no encryption.

### Telemetry and APIs

`http://miner_ip/api/v1/miner/details` returns JSON:

```json
{
  "hashrate_5s": 110.42,
  "hashrate_1m": 109.87,
  "hashrate_15m": 110.05,
  "hashrate_24h": 109.90,
  "boards": [
    {
      "id": 0,
      "chip_count": 76,
      "hashrate": 36.7,
      "temperature": 73.2,
      "voltage": 14.1,
      "frequency_avg": 580.0,
      "hw_error_rate": 0.003
    }
  ],
  "fans": [{"rpm": 4200}, {"rpm": 4180}],
  "uptime_sec": 432000,
  "psu_power": 3245,
  "efficiency_jth": 29.4
}
```

Prometheus scrape on `:9100` exposes the same data as Prometheus
metrics (`bosminer_hashrate`, `bosminer_temperature`, etc.).

## Worked example

An Antminer S19j Pro (stock 104 TH/s @ 3068 W = 29.5 J/TH) flashed
to BOS+ in Balanced mode:

1. After 24h autotune, dashboard shows:
   - Hashrate 1h: 108.3 TH/s (+4.1%)
   - Power: 3010 W (-1.9%)
   - Efficiency: 27.8 J/TH (-5.8%)
2. HW error rate stable at 0.4%, well under 1% budget.
3. Switch to Efficiency mode:
   - Hashrate: 92.5 TH/s
   - Power: 2300 W
   - Efficiency: 24.9 J/TH (-15.6%)

Choice depends on electricity cost. At $0.04/kWh, balanced earns more
gross BTC; at $0.08/kWh, efficiency mode often wins on net margin.

A miner switching from V1 to V2 sets:

```
url = "stratum2+tcp://v2.stratum.braiins.com:3336/u95GE..."
```

Verifies via dashboard that "Connected protocol: Stratum V2" appears.
All shares now flow over Noise-encrypted channel; man-in-the-middle
on the WAN cannot read or alter them.

## Common pitfalls

- **Flashing wrong model image** - cross-flashing an S19 image onto an
  S19j Pro can brick the unit; SD-card recovery may be required. Always
  match exact model.
- **Aggressive autotune target** - setting `psu_power_limit` too high
  can exceed PSU rating and trip the unit. Keep limit at PSU spec
  (typical 3250W for S19 Pro).
- **Unflashed stratum URL** - if you flash BOS+ but leave Bitmain's
  default pool URL, you mine to Bitmain. Always replace pool config
  on first boot.
- **HTTP-only dashboard exposed to WAN** - the dashboard has admin
  controls; expose only on LAN or behind a VPN.
- **Pool fee discount expectations** - the discount only applies on
  Braiins Pool. Using BOS+ with another pool is fine but no fee
  discount.
- **Forgetting heat profile match** - Efficiency mode runs cooler;
  if you reduced fan speed for noise, then switch to Performance, you
  may overheat.

## References

- Braiins OS+ docs <https://braiins.com/os/plus>
- Source repo <https://github.com/braiins/braiins-os>
- Braiins API spec <https://braiins.com/os/plus/docs/api>
- Stratum V2 spec <https://stratumprotocol.org/specification/>
- Skill: `bitcoin/mining/stratum-v2`
