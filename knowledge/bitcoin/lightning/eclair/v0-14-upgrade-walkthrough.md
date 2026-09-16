# Eclair v0.13 to v0.14 Upgrade Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/eclair`.
> Canonical source: https://github.com/ACINQ/eclair/releases
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/eclair/SKILL.md

## Concept

Most Eclair upgrades are "stop, replace the zip, restart" - every
v0.14.x release note says exactly that. The v0.13 -> v0.14 step is the
exception, and it is the one that strands nodes, because **v0.14.0
refuses to start** if the node still holds channels it no longer has
code for.

There are two separate one-way gates, and they are cleared by two
different releases:

| Gate | Cleared by | What happens if you skip it |
|------|-----------|-----------------------------|
| Legacy channel **codecs** (pre-v0.13 on-disk encoding) | Run v0.13.0 once | Channel data cannot be deserialized |
| Channels **without anchor outputs** | Close them while on v0.13.1 | v0.14.0 won't start; no code path left to operate them |

Plus a moving Bitcoin Core floor, raised twice inside four months:

| Eclair | Released | Bitcoin Core floor |
|--------|----------|--------------------|
| v0.13.1 | 2025-10-27 | 29.x (notes recommend v29.2) |
| v0.14.0 | 2026-05-21 | 30.x (for v3/TRUC and ephemeral dust) |
| v0.14.1 | 2026-07-29 | 31.x |
| v0.14.2 | 2026-08-26 | 31.x |
| v0.14.3 | 2026-09-14 | 31.x |

The v0.14.3 README states "Eclair requires Bitcoin Core 31 or higher".
Release dates above are from the ACINQ/eclair GitHub releases page as
of September 2026; v0.14.3 is the current release.

Because the floor moved between v0.14.0 and v0.14.1, an operator going
straight from v0.13.1 to v0.14.3 - the sensible target - needs Core
31.x, not 30.x. There is no point staging through v0.14.0.

## Walkthrough / mechanics

### 0. Are you even on v0.13?

Nodes older than v0.13.0 cannot reach v0.14 in one hop. v0.13.1 removed
the code that deserializes channel data written by pre-v0.13 releases,
so v0.13.0 has to run at least once to rewrite it. v0.13.1 additionally
moves closed channels into a dedicated table on first restart, which is
a one-time migration that can take a while on a node with a long
channel history.

Order: `<your version>` -> v0.13.0 -> v0.13.1 -> v0.14.x.

### 1. On v0.13.1, find the non-anchor channels

v0.13.1 is the last release that can operate channels without anchor
outputs. List them while you are still on it:

```sh
eclair-cli channels \
  | jq '.[] | { channelId: .data.commitments.channelParams.channelId, commitmentFormat: .data.commitments.active[].commitmentFormat }' \
  | jq 'select(.["commitmentFormat"] == "legacy")'
```

An empty result is the green light. Anything returned has to be closed
before the upgrade.

### 2. Close them

Cooperative close, peer online:

```sh
eclair-cli close --channelId=<channel_id> \
  --preferredFeerateSatByte=<feerate_satoshis_per_byte>
```

Peer offline or unresponsive - force-close to recover the funds:

```sh
eclair-cli forceclose --channelId=<channel_id>
```

Force-closes settle on the channel's own delay, so this step sets the
real upgrade timeline. Start it days before you plan to swap binaries,
not on the night.

### 3. Raise the Bitcoin Core floor

Upgrade `bitcoind` to 31.x first, while Eclair is still on v0.13.1.
Every release note phrases the dependency as a floor - "newer versions
of Bitcoin Core may be used, but have not been extensively tested" - so
running ahead of the floor is the tolerated direction; running behind
it is not. The Core node still has to be synchronized,
wallet-enabled, non-pruning, tx-indexing and ZMQ-enabled - that has not
changed. The README also warns that an upgraded wallet may need funds
swept to a freshly generated address.

### 4. Verify and install

Eclair releases are signed with key `E04E48E72C205463`, published at
`https://acinq.co/pgp/drouinf2.asc` and on GitHub user `sstone`:

```sh
gpg --import drouinf2.asc
gpg -d SHA256SUMS.asc > SHA256SUMS.stripped
sha256sum -c SHA256SUMS.stripped
```

Builds are deterministic. The reference environment for v0.14.1 through
v0.14.3 is Ubuntu 24.04.1 with Adoptium OpenJDK 21.0.6, built with
`./mvnw clean install -DskipTests`. Eclair targets Java 21 generally.

### 5. Migrate the configuration

Keys that changed across the v0.14 line, in the order they landed:

| Key | Change | Release |
|-----|--------|---------|
| `eclair.relay.fees.reset-existing-channels` | New; set `false` to skip re-applying config fees to every channel on restart | v0.14.0 |
| `eclair.peer-scoring.enabled` | New; `false` by default | v0.14.0 |
| `eclair.onion-messages.relay-policy` | **Removed** | v0.14.2 |
| `eclair.router.sync.max-queries-per-second` | New per-peer gossip-query limit (`5` in v0.14.3 `reference.conf`) | v0.14.2 |
| `eclair.router.sync.max-queries-per-sync` | New; `2000` | v0.14.2 |
| `eclair.channel.max-funding-satoshis` | New; `5000000000` (50 BTC), larger channels rejected | v0.14.2 |
| `eclair.peer-connection.max-pending-incoming-connections` | New; `500`, `0` disables | v0.14.2 |
| `eclair.tor.auth` | Default flipped `password` -> `safecookie` | v0.14.2 |
| `eclair.on-chain-fees.max-funding-feerate` | New; `50` sat/byte cap on funding and splice feerates | v0.14.3 |

`relay-policy = "channels-only"` is the one that silently stops doing
what you meant. Its replacement is a pair of feature bits:

```hocon
eclair.features.option_onion_messages = disabled
eclair.features.option_onion_messages_only_channels = optional
```

Two more behavioural defaults worth reading before restart:

- `eclair.relay.peer-reputation` and
  `eclair.relay.reserved-for-accountable` (`0.5` by default) implement
  the v0.14.0 channel-jamming accountability draft. Eclair collects
  data only - it does **not** yet fail HTLCs that miss the
  restrictions. Set `peer-reputation.enabled = false` and
  `reserved-for-accountable = 0.0` to opt out.
- `eclair.features.option_simple_taproot` is **active by default** in
  v0.14.0 and later (unannounced taproot channels only; public taproot
  channels wait on gossip spec work). `zero_fee_commitments` is not -
  set it to `optional` to opt in.

### 6. Fix the integrations before restarting

v0.14.0 is the breaking release for anything that consumes Eclair's
output:

- **Websocket / plugin events.** `channel-opened` was removed. It
  splits into `channel-funding-created` (funding *or* splice tx signed
  and publishable), `channel-confirmed` (enough confirmations) and
  `channel-ready` (can process payments - may fire *before*
  `channel-confirmed` on a 0-conf channel). Discriminate initial
  funding from a splice with `fundingTxIndex`: `0` is the open,
  anything higher is a splice.
- **Audit DB.** All `audit` tables changed backwards-incompatibly to
  key on `node_id` rather than channel. Old data cannot be migrated, so
  it is renamed with a `_before_v14` suffix and left behind; new tables
  start empty. Historic rows are reachable only by direct SQL.
- **Removed APIs.** `networkfees` and `channelstats` are gone,
  superseded by `relaystats`.
- **Plugins.** JARs in `plugins/` are compiled against Eclair's
  internals and break across versions - rebuild and test before the
  upgrade window, not after.
- **Splicing prototype.** The v0.9.0 prototype protocol was removed in
  v0.14.0. Anything depending on it must move to the BOLT splice.

## Worked example

A routing node on v0.12, September 2026, targeting v0.14.3:

| Step | Action | Blocking wait |
|------|--------|---------------|
| 1 | Install v0.13.0, start, let channel codec migration finish | Minutes |
| 2 | Install v0.13.1, start, let closed-channel table migration finish | Minutes to hours on a long history |
| 3 | Run the `commitmentFormat == "legacy"` query | - |
| 4 | `close` each hit; `forceclose` the unreachable peers | **Days** - to-self-delay |
| 5 | Re-run the query until it returns nothing | - |
| 6 | Upgrade `bitcoind` to 31.x, resync/reindex as needed | Hours |
| 7 | Rebuild plugins; update websocket consumers for the new events | - |
| 8 | Edit `eclair.conf`: drop `relay-policy`, review `tor.auth` | - |
| 9 | Stop, install v0.14.3, start | Minutes |

Step 4 dominates. Everything else is an evening.

Inside the v0.14.x line there is no ceremony: v0.14.1, v0.14.2 and
v0.14.3 each state they are fully compatible with previous versions -
stop, upgrade, restart. v0.14.2 and v0.14.3 are both hardening releases
that ACINQ describes as exploitable by malicious nodes, so treat the
patch upgrades as urgent even though they are cheap.

## Common pitfalls

- **Trying to upgrade with legacy channels open.** v0.14.0 will not
  start, and rolling back to v0.13.1 to close them costs you the
  downtime twice. Run the query first.
- **Assuming Core 30 is enough.** It was, for exactly one release.
  v0.14.1 onward needs 31.x.
- **Jumping from pre-v0.13 straight to v0.14.** The codec migration in
  v0.13.0 is mandatory and not reversible by skipping it.
- **Leaving `relay-policy` in `eclair.conf`.** The key no longer
  exists; the restriction it expressed has to be re-stated as feature
  bits.
- **Remote `bitcoind` over plain RPC.** From v0.14.2 ACINQ documents
  that `bitcoind` must be on the same machine, or behind a tunnel
  giving both encryption and authentication. Plain network RPC is
  treated as a broken setup, not a supported one.
- **Tor control port on another host.** `tor.auth = password` is
  rejected when `eclair.tor.host` is not a local address from v0.14.2
  (v0.14.3 re-allows it on private networks). Note that the onion
  private key is sent to the control port on every startup, so a remote
  control port exposes it regardless of auth method.
- **Expecting the old audit dashboards to keep working.** They query
  tables that are now named `*_before_v14` and no longer being written.
- **Expecting a watchtower to cover the downtime.** Eclair has none -
  no client, no server, no config setting, as of September 2026.
  Breach protection is the node being online. Plan the maintenance
  window accordingly.

## References

- ACINQ/eclair release notes `docs/release-notes/eclair-v0.13.1.md`,
  `eclair-v0.14.0.md`, `eclair-v0.14.1.md`, `eclair-v0.14.2.md`,
  `eclair-v0.14.3.md` - source of every version, default and command
  above.
- `eclair-core/src/main/resources/reference.conf` at tag `v0.14.3` -
  current defaults.
- `README.md` at tag `v0.14.3` - Bitcoin Core and Java requirements.
- Related: [`overview.md`](overview.md) for orientation; the dev-suite
  skill for the full configuration and API surface.
