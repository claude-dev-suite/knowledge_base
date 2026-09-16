# Taproot Channel Feature Bits and Negotiation - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/taproot-channels`.
> Canonical source: https://github.com/lightning/bolts/blob/master/bolt-simple-taproot.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/taproot-channels/SKILL.md

## Concept

Simple taproot channels are specified in an **extension BOLT**, not in
the numbered BOLTs. `bolt-simple-taproot.md` was merged into
`lightning/bolts` on 2026-05-04 (PR #995, "extension-bolt: simple
taproot channels (feature 80/81)") as a standalone file at the repo
root.

The practical consequence catches people out: **`option_simple_taproot`
is not in `09-features.md`**. As of September 2026 the core BOLT 9
feature table ends at `option_onion_messages_only_channels` (66/67) and
contains no taproot entry at all. Bits 80/81 are defined in the
extension BOLT's own feature table, which explicitly says it is
"inheriting the structure put forth in BOLT 9". Grepping the core spec
for the bit and finding nothing does not mean the bit is unassigned.

The second thing to know is that there are **two** pairs of bits, not
one, and both are live on the network as of September 2026.

## Walkthrough / mechanics

### The two bit pairs

From the extension BOLT's feature table:

| Bits | Name | Context | Dependencies |
|------|------|---------|--------------|
| 80/81 | `option_simple_taproot` | IN | `option_channel_type`, `option_simple_close` |
| 180/181 | `option_simple_taproot_staging` | IN | `option_channel_type` |

`I` = presented in `init`, `N` = presented in `node_announcement`.

The spec's rationale for allocating two pairs: the higher pair is
"+100" and is meant for "preliminary experimental deployments before
this protocol extension has been finalized", referred to as the
*staging* feature bit. 80/81 is the final pair.

Note the dependency asymmetry, which is easy to miss and is the source
of real negotiation failures: the **final** bits additionally require
`option_simple_close` (bits 60/61), while the staging bits do not. A
peer that supports taproot channels but not simplified closing can
legally advertise 181 and not 81.

### What LND actually advertises

`lnwire/features.go` at `v0.21.0-beta` defines all four constants:

```go
SimpleTaprootChannelsRequiredFinal   = 80
SimpleTaprootChannelsOptionalFinal   = 81
SimpleTaprootChannelsRequiredStaging = 180
SimpleTaprootChannelsOptionalStaging = 181
```

with display names `simple-taproot-chans` for 80/81 and
`simple-taproot-chans-x` for 180/181 - the `-x` suffix is the only
thing distinguishing them in RPC output.

`feature/default_sets.go` puts **both** optional bits (81 and 181) in
`SetInit` and `SetNodeAnn`. So an LND v0.21 node advertises the final
and the staging bit simultaneously, for backwards compatibility with
peers still on staging-only builds. Seeing 181 on a peer does not mean
that peer is running old software.

### The overlay channel bits are something else

The same file defines a third pair in LND's custom range:

```go
SimpleTaprootOverlayChansOptional = 2025
SimpleTaprootOverlayChansRequired = 2026
```

named `taproot-overlay-chans`. These are for the custom taproot overlay
channel type (the Taproot Assets carrier channel), not for simple
taproot channels, and they are not in the extension BOLT. The numbers
resemble years and are routinely misread as dates.

### Explicit negotiation only

The extension BOLT is emphatic that this channel type "cannot be
announced on the public network (gossip protocol changes are
required)", and therefore "cannot be used as an interchangeable default
channel type". It SHOULD only be used with *explicit* channel
negotiation - `option_simple_taproot` is also a defined **channel
type** bit for the `channel_type` field, not merely an `init` bit.

The failure mode the spec calls out directly: if taproot channels were
treated as an implicit default, "two parties that advertise the feature
bit would not be able to open publicly advertised channels". Hence
taproot channels are unannounced/private in practice as of September
2026.

## Worked example

Checking what a peer negotiated. LND's `Feature` message is keyed by
raw bit number:

```protobuf
message Feature {
    string name = 2;
    bool is_required = 3;
    bool is_known = 4;
}
```

so the features map is read by bit, not by name:

```bash
lncli getinfo | jq '.features | with_entries(
  select(.key | IN("81","181","60","61","2025")))'
```

Interpreting the result on a v0.21 node:

| Present | Meaning |
|---------|---------|
| `81` + `181` | Normal LND v0.21 default - both advertised |
| `181` only | Staging-only peer; open with the staging channel type |
| `81` only | Peer dropped staging support |
| neither | No taproot channels; check the build and config |
| `2025` | Taproot *overlay* (assets) channels - unrelated capability |

If `81` is advertised but `60`/`61` (`option_simple_close`) is not, the
peer is advertising the final bit without its declared dependency; fall
back to the staging type rather than assuming the final type will
negotiate.

## Common pitfalls

- **Grepping `09-features.md` for the bit.** It is not there and will
  not be as long as this lives in an extension BOLT. Read
  `bolt-simple-taproot.md` at the repo root.
- **Assuming 80/81 replaced 180/181.** LND v0.21 advertises both. The
  staging pair is not deprecated as of September 2026.
- **Reading 2025/2026 as years.** They are LND custom feature bits for
  taproot *overlay* channels.
- **Forgetting the dependency gap.** Final bits depend on
  `option_simple_close`; staging bits do not. A peer can legitimately
  offer one pair and not the other.
- **Expecting a public channel.** Gossip changes for taproot channels
  are not specified, so these channels are not announceable. Explicit
  channel-type negotiation, unannounced channels.
- **Matching on the RPC name.** `simple-taproot-chans` and
  `simple-taproot-chans-x` differ by one suffix character; match on the
  bit number instead.

## References

- `lightning/bolts` `bolt-simple-taproot.md` ("Feature Bits" section),
  merged 2026-05-04 via PR #995.
- `lightning/bolts` `09-features.md` at master (no taproot entry as of
  September 2026).
- lnd `lnwire/features.go` and `feature/default_sets.go` at tag
  `v0.21.0-beta`.
- lnd `lnrpc/lightning.proto` (`Feature`, `features` maps).
