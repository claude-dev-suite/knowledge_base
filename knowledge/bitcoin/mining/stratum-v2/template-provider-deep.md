# Stratum V2 Template Provider - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/mining/stratum-v2`.
> Canonical source: <https://stratumprotocol.org/specification/07-Template-Distribution-Protocol/>
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/mining/stratum-v2/SKILL.md

## Concept

The Template Provider (TP) is the SV2 component that turns a Bitcoin Core
mempool into a stream of fresh block templates that downstream Job
Declarator Clients can declare. It is implemented as a feature of Bitcoin
Core (since v25 with the `--templateprovider` build flag) or via the SRI
"pool" tooling.

The TP exposes a thin TCP+Noise interface speaking the Template Distribution
Protocol (TDP, sub-protocol id `0x04`). It pushes:

- `NewTemplate` - a fresh block template (block at height `H`).
- `SetNewPrevHash` - chain tip change (mid-template invalidation).
- `RequestTransactionData` / `.Success` - on demand tx body fetch.
- `SubmitSolution` - upstream from JDC when a found block needs to be
  injected back into bitcoind for relay.

## Walkthrough / mechanics

### Process model

```
[bitcoind w/ TP feature]
   |- mempool, validation, p2p
   |
   v (Noise XK + TDP)
[JDC running on the same or adjacent host]
   |
   v (Noise XK + JDP)
[Pool Job Declarator Server]
```

The TP shares no secrets with the pool; only the JDC speaks to both. This
keeps mempool details private from the pool.

### Setup

After a Noise XK handshake, JDC sends:

```
SetupConnection { protocol: 0x04, ... }
```

Server replies `SetupConnection.Success`. Then JDC sends:

```
CoinbaseOutputDataSize {
  coinbase_output_max_additional_size: 100
}
```

This tells the TP how many bytes the JDC will append to the coinbase
(payout outputs, witness commitment). The TP will craft templates that
respect this budget by reserving sigops/weight for it.

### Template flow

```
TP -> JDC: NewTemplate {
  template_id: 42
  future_template: false
  version: 0x20000000
  coinbase_tx_version: 2
  coinbase_prefix: <bytes>
  coinbase_tx_input_sequence: 0xffffffff
  coinbase_tx_value_remaining: 312_501_250_000   // sats available for outputs
  coinbase_tx_outputs_count: 1
  coinbase_tx_outputs: <serialised outputs - witness commitment, etc.>
  coinbase_tx_locktime: 0
  merkle_path: [<32B>, ...]
}
```

If `future_template = true`, the template is for a height that doesn't yet
have a confirmed parent (used to pre-warm work for the next block).

`merkle_path` is the merkle branch from the coinbase to the root, computed
over current mempool txs minus coinbase. `coinbase_tx_value_remaining` is
the subsidy plus fees not yet committed to the coinbase prefix - the JDC
must allocate this across its own outputs without exceeding it.

### Tip changes

When a new block is found upstream, TP pushes:

```
SetNewPrevHash {
  template_id: 42
  prev_hash: <32B LE>
  header_timestamp: 1714003600
  n_bits: 0x17034219
  target: <32B LE>
}
```

This binds template 42 to the new tip. If template 42 was for a different
prev_hash, the JDC must abandon it and wait for `NewTemplate`.

The TP keeps the last few `template_id`s alive so JDC can quickly switch
between candidates without round-trip stalls.

### Solution submission

When a miner finds a block:

```
JDC -> TP: SubmitSolution {
  template_id: 42
  version: 0x20000000
  header_timestamp: 1714003692
  header_nonce: 0xa1b2c3d4
  coinbase_tx: <full serialised coinbase>
}
```

TP reconstructs the full block (it knows the txs by template_id), validates,
and asks bitcoind to broadcast via `submitblock`. This means the *miner's*
node is the relay path - independent of pool connectivity.

### RequestTransactionData

If the JDS asks JDC for missing txs (see Job Declaration flow), JDC may not
have them in memory. It can ask the TP:

```
JDC -> TP: RequestTransactionData {
  template_id: 42
}

TP -> JDC: RequestTransactionData.Success {
  template_id: 42
  excess_data: <empty>
  transaction_list: [<tx bytes>, <tx bytes>, ...]
}
```

This is the one heavyweight message; for a 4 MB block of `~3500` txs the
payload can be several MB.

## Worked example

A solo miner at 200 TH/s with 32 GB RAM running bitcoind + TP + SRI JDC.
A normal cycle:

1. TP computes a new template with ~3500 txs, 3.06 BTC subsidy+fees.
2. TP sends `NewTemplate { template_id=101, ..., coinbase_tx_value_remaining=312_500_000_000 }` plus pre-built `merkle_path` of 12 hashes.
3. JDC builds coinbase outputs:
   - Output 0: `bc1q...miner` for `312_499_700_000` sats
   - Output 1: OP_RETURN witness commitment `0x6a24aa21a9ed...`
   - Total = 312_499_700_000 + 0 = within `value_remaining`.
4. JDC declares to JDS using template's coinbase_prefix + new outputs +
   suffix; JDS approves.
5. ASICs grind. After 7 minutes a share at network target arrives.
6. JDC sends `SubmitSolution { template_id=101, header_nonce=..., coinbase_tx=... }`.
7. TP runs `processnewblock` internally and broadcasts. Block 893500 accepted.

If during step 5 a competitor block arrives, TP sends `SetNewPrevHash` and
JDC immediately requests a new declaration on the new tip.

## Common pitfalls

- **TP not enabled in bitcoind** - many distros ship without TP support.
  Verify with `bitcoin-cli getnetworkinfo` and look for the TP service
  flag, or check `--templateprovider` in start args.
- **Stale `SetNewPrevHash`** - JDC must drop pending shares on tip change
  or the pool will reject as "stale share, wrong prev_hash".
- **Coinbase oversize** - if JDC builds outputs that exceed
  `coinbase_tx_value_remaining`, the resulting coinbase is invalid (sum
  of outputs > subsidy+fees). Always double-check the cumulative output
  amount vs the announced budget.
- **Mempool eviction during long share search** - if a tx in the
  template is evicted from bitcoind's mempool after `NewTemplate` and
  before `SubmitSolution`, the TP can still serialise the original tx
  for the block (it caches per-template). Don't rely on bitcoind's live
  mempool for solution time.
- **Network jitter** - TP and JDC over WAN add latency that can stale
  shares. Run TP on the same machine as JDC if possible.

## References

- SV2 Template Distribution Protocol spec <https://stratumprotocol.org/specification/07-Template-Distribution-Protocol/>
- Bitcoin Core PR 27433 (initial TP implementation) <https://github.com/bitcoin/bitcoin/pull/27433>
- SRI `template-provider` binary <https://github.com/stratum-mining/stratum/tree/main/roles/template-provider>
- Skill: `bitcoin/mining/stratum-v2`
