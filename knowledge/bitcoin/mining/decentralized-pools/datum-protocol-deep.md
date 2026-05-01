# Datum Protocol - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/mining/decentralized-pools`.
> Canonical source: <https://github.com/OCEAN-xyz/datum_gateway>, Ocean Mining docs
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/mining/decentralized-pools/SKILL.md

## Concept

Datum (Decentralised Alternative Templates for Universal Mining) is the
template-distribution protocol used by Ocean Mining. It is conceptually
similar to Stratum V2's Job Declaration / Template Provider split, but it
is a simpler, narrower design that retro-fits onto Stratum V1 ASICs
without requiring SV2 firmware.

The key idea: a miner's local "Datum Gateway" builds a block template from
their own bitcoind, hands it to the pool over a Datum-specific message,
and the pool agrees to mine it as long as the coinbase pays the pool's
required output (which Ocean uses for its non-custodial direct-to-miner
distribution).

## Walkthrough / mechanics

### Architecture

```
[bitcoind w/ mempool]
     |- RPC + ZMQ
     v
[datum_gateway]  ---- Datum/SV1 hybrid ----> [Ocean Pool]
     |
     v (Stratum V1)
[ASIC]
```

`datum_gateway` is a local C process that:
1. Opens an upstream connection to Ocean's Datum endpoint.
2. Speaks Stratum V1 to the ASIC (so existing Antminer/Whatsminer firmware
   works unmodified).
3. Negotiates which template the ASIC mines: by default Ocean's, but
   optionally a locally-built one.

### The "use my template" flag

When a Datum gateway connects to Ocean, it negotiates a session in which:

- The pool accepts that the miner may submit shares for either Ocean's
  central template *or* the miner's locally constructed template.
- For miner-built templates, the gateway sends a Datum message containing:
  - Coinbase prefix + suffix.
  - Tx short-IDs (BIP 152-style) or full TXIDs.
  - Merkle path.
  - Miner's signature over the template.

Ocean validates that the coinbase respects Ocean's required output (so
Ocean's revenue model still works) but otherwise lets the miner pick
the tx set.

If the miner doesn't build a template, the gateway falls through to
Ocean's central template like a normal Stratum V1 client.

### Coinbase output policy

Ocean requires the coinbase to:
- Have one output paying `OP_HASH160 <ocean_pool_address>` (small dust
  marker - actually their accounting / non-custodial split address).
- Have one or more outputs paying miners directly per its non-custodial
  distribution rules.
- Include the witness commitment OP_RETURN.

The gateway enforces this when building local templates. If a malformed
coinbase is submitted, Ocean rejects with an error.

### Stratum V1 wrapping

Datum reuses Stratum V1's wire format for ASIC connections. The ASIC
sees:

```
mining.subscribe        (normal V1)
mining.authorize        (normal V1, username = wallet)
mining.set_difficulty   (normal V1)
mining.notify           (normal V1, but params come from local template)
mining.submit           (normal V1)
```

What changes is *upstream*: instead of the gateway being a transparent
proxy, it injects locally-built notify payloads when Datum negotiation
succeeded. This is invisible to the ASIC.

### Share submission flow

1. ASIC submits share via Stratum V1 to gateway.
2. Gateway validates against the local template's target.
3. If share also meets network target, gateway:
   - Reconstructs the full block.
   - Sends it to the pool via a Datum `block_found` message.
   - In parallel, sends to local bitcoind via `submitblock` (so a slow
     pool connection doesn't lose the block).
4. Pool credits the share for accounting; if block accepted, coinbase
   pays per Ocean's non-custodial scheme directly to miners.

### Why this differs from SV2

| Aspect | SV2 Job Declaration | Datum |
|--------|---------------------|-------|
| Wire format | Binary + Noise | JSON + plain TCP (or TLS) |
| ASIC firmware | SV2 firmware needed | Stock SV1 ASIC works |
| Encryption | Mandatory Noise XK | Optional TLS wrap |
| Template coverage | Full SV2 spec | Targeted at miner-template + V1 wrap |
| Adoption | SRI, Braiins | Ocean Mining only |
| Complexity | Multi-role | Single gateway binary |

Datum trades cryptographic rigour for deployability: any V1 ASIC works
today without firmware changes. SV2's Noise/binary approach is more
future-proof but requires both pool and firmware to support it.

## Worked example

A miner running:
- bitcoind v25 on Ubuntu 24.04, mempool ~70MB.
- `datum_gateway` listening on `0.0.0.0:7777` upstream to
  `mining.ocean.xyz:443`.
- Antminer S19 firmware (stock) pointed at `stratum+tcp://gateway:7777`,
  username `bc1q...miner.s19-1`, password `x`.

What happens for one block:

1. Gateway opens TLS to Ocean, completes Datum auth using miner's wallet.
2. Gateway runs `getblocktemplate` against bitcoind every 30 sec or on
   ZMQ tip change. Picks ~3500 txs.
3. Gateway computes coinbase:
   - Prefix: BIP 34 height push + extranonce slot.
   - Outputs: Ocean's required output + miner's `bc1q...miner` for the
     remainder (3.125 + fees - Ocean dust).
4. Gateway sends Datum `template_proposal` upstream; Ocean validates and
   acks.
5. Gateway pushes `mining.notify` to S19. S19 hashes against the local
   template's target; vardiff produces ~3 shares/sec.
6. After ~7 min an S19 share hits network target. Gateway:
   - Builds full block via locally-cached template.
   - Calls `submitblock` on bitcoind (block accepted, propagates to peers).
   - Sends `block_found` to Ocean.
7. Ocean records the block; the on-chain coinbase already paid the miner
   (no off-chain settlement). Miner sees the reward in their wallet
   after 100 confirmations.

If the gateway loses Ocean connectivity, it falls back to a "solo-style"
mode where it keeps issuing local templates to the ASIC and submits any
found block via `submitblock` only. The block is still valid; only the
share-credit accounting with Ocean is lost.

## Common pitfalls

- **Outdated `datum_gateway` build** - Ocean updates the protocol
  occasionally; running an old binary leads to "unsupported template
  version" errors. Pull from `OCEAN-xyz/datum_gateway` regularly.
- **Wrong coinbase output value** - if the gateway misallocates sats
  between Ocean's dust output and miner's payout, Ocean rejects the
  template. Check the `coinbase_output_max_additional_size` analogue
  carefully.
- **bitcoind not running or pruned** - the gateway needs `getblocktemplate`
  to work; pruned nodes can't serve recent blocks for the template
  context. Run an unpruned node.
- **TLS validation off by default** - some `datum_gateway` versions skip
  cert validation, exposing you to MITM. Verify TLS pinning in config.
- **Confusing Datum with SV2** - they are not the same protocol. SV2
  firmware does *not* enable Datum and vice versa. Ocean's official
  client is `datum_gateway`; SRI's client speaks SV2.

## References

- Datum Gateway source <https://github.com/OCEAN-xyz/datum_gateway>
- Ocean Mining docs <https://ocean.xyz/docs>
- Bitcoin Optech newsletter coverage of Datum (2024-2025)
- Skill: `bitcoin/mining/stratum-v2` (compare/contrast)
