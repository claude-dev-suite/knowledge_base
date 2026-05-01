# Stratum V2 Job Declaration Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/mining/stratum-v2`.
> Canonical source: <https://stratumprotocol.org/specification/06-Job-Declaration-Protocol/>
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/mining/stratum-v2/SKILL.md

## Concept

Job Declaration is the SV2 sub-protocol that lets a miner choose their own
block content and have a pool agree to mine it. The flow is:

1. Miner gets a block template from a Template Provider (typically their own
   bitcoind).
2. Miner declares "I want to mine block X with this coinbase output for the
   pool's reward script" via the Job Declarator Client.
3. The pool's Job Declarator Server validates the declaration and issues a
   `mining.new_extended_job` to mine against.
4. The miner's ASIC hashes the SV2-supplied job; valid shares go to the pool.

This decouples *who builds the template* from *who pays out*. Pools no
longer dictate transaction selection.

## Walkthrough / mechanics

### Roles

```
[bitcoind] -- ZMQ + RPC --> [Template Provider]
                                    |
                                    v
[Job Declarator Client (JDC)] -- TCP+Noise --> [Job Declarator Server (JDS) at pool]
       |                                                    |
       v                                                    v
[Mining Proxy (SV1 ASIC bridge)] <----------------- [Pool's Mining Server]
       |
       v
[ASICs over SV1 or SV2]
```

The JDC + JDS pair handles declaration. The mining server then issues
extended jobs over the SV2 Mining Protocol channel.

### Binary framing (vs V1's JSON)

Every SV2 message uses a 6-byte header followed by length-prefixed payload:

```
| ext_type (2) | msg_type (1) | msg_len (3 LE) | payload | MAC (16) |
```

The MAC is the ChaCha20-Poly1305 tag from the Noise session. There is no
JSON; types are defined by the SV2 spec (`SetupConnection`,
`AllocateMiningJobToken`, `DeclareMiningJob`, etc.).

### Setup

JDC -> JDS:

```
SetupConnection {
  protocol: 0x02              // Job Declaration Protocol
  min_version: 2
  max_version: 2
  flags: 0x00000000
  endpoint_host: "pool.example.com"
  endpoint_port: 34254
  vendor: "sri-jdc"
  hardware_version: "1.0"
  firmware: "1.4.0"
  device_id: "miner-eu-1"
}
```

JDS responds with `SetupConnection.Success` or `.Error`.

### Token allocation

Before declaring a job, the miner asks the pool to mint a one-time token
representing pool's commitment to mine the future declaration:

```
AllocateMiningJobToken {
  user_identifier: "bc1q...minerwallet"
  request_id: 1
}
```

Server responds:

```
AllocateMiningJobToken.Success {
  request_id: 1
  mining_job_token: <32 bytes>
  coinbase_output_max_additional_size: 100
  async_mining_allowed: true
  coinbase_output: <pool's required output script + value>
}
```

`coinbase_output` is a short serialised txout the pool requires (could be
the pool's payout address). `coinbase_output_max_additional_size` lets the
miner add their own outputs (e.g., for FPPS bookkeeping or self-payout).

### Declaration

```
DeclareMiningJob {
  request_id: 2
  mining_job_token: <token from above>
  version: 0x20000000
  coinbase_prefix: <bytes>          // up to extranonce
  coinbase_suffix: <bytes>          // after extranonce
  tx_short_hash_list: [<6 bytes>, ...]  // BIP 152 short-IDs
  tx_hash_list_hash: <32 bytes>     // SHA256d of full TXIDs concatenated
  excess_data: <empty or extra>
}
```

The miner sends short IDs, not full transactions. The server reconstructs
the tx set from its own mempool (or asks for missing txs via
`ProvideMissingTransactions`). This keeps the message under 1500 bytes
in the common case.

Server response:

```
DeclareMiningJob.Success {
  request_id: 2
  new_mining_job_token: <new 32 bytes>
}
```

Or `DeclareMiningJob.Error` with reason ("invalid-coinbase-output",
"missing-transactions", "tx-rejected").

### Missing tx round trip

```
ProvideMissingTransactions {
  request_id: 3
  unknown_tx_position_list: [3, 17, 42]   // indices into tx_short_hash_list
}

ProvideMissingTransactions.Success {
  request_id: 3
  transaction_list: [<raw tx bytes>, ...]
}
```

### Job dispatch

Once the JDS accepts the declaration, the pool's Mining Server pushes:

```
NewExtendedMiningJob {
  channel_id: 1
  job_id: 42
  min_ntime: 1714003600
  version: 0x20000000
  version_rolling_allowed: true
  merkle_path: [<32-byte hashes>, ...]   // ordered branch
  coinbase_tx_prefix: <bytes>
  coinbase_tx_suffix: <bytes>
}
```

ASICs hash. Shares come back via `SubmitSharesExtended`.

## Worked example

Solo-style miner runs:
- bitcoind with mempool ~80MB, ZMQ on 28332.
- Template Provider (bitcoind v25+ with `--templateprovider`).
- SRI Job Declarator Client.
- SRI Mining Proxy bridging an Antminer S19 over Stratum V1.

Cycle for one block:

1. Template Provider notifies JDC of a new template at height 893500
   with `~3500` txs, total fees `~0.06 BTC`.
2. JDC computes coinbase: prefix includes height push + extranonce slot;
   suffix has miner's payout output `bc1q...miner` (3.125 + 0.06 BTC) plus
   pool's required output (0 BTC dust marker).
3. JDC sends `DeclareMiningJob` with 3500 short tx IDs (= 21 KB).
4. JDS already has 3490 in its mempool; asks for 10 missing via
   `ProvideMissingTransactions`. JDC returns those 10 raw txs (~5 KB).
5. JDS validates: total fees match, coinbase output structure compliant,
   no banned txs. Returns `Success`.
6. Mining Server emits `NewExtendedMiningJob` to all ASICs on channel.
7. After 4-5 minutes, an ASIC submits a share that hits network target;
   JDS verifies and broadcasts the block. Miner's payout is in the
   coinbase directly (non-custodial).

## Common pitfalls

- **TP/JDC version skew** - bitcoind's TP API and SRI's JDC must match
  versions; mismatched encoding silently produces invalid templates.
- **Mempool divergence** - if JDS doesn't have a tx the JDC declared, the
  declaration fails. Ensure both nodes peer well or the JDC propagates
  txs to JDS upstream.
- **Forgetting witness commitment** - the JDC must include the segwit
  commitment OP_RETURN in the coinbase suffix; otherwise the block is
  invalid even if the share is valid.
- **`min_ntime` violation** - submitting a share with `ntime` below
  `min_ntime` is rejected; this is the SV2 equivalent of MTP enforcement.
- **Token replay** - tokens are one-time. Reusing a token in a second
  `DeclareMiningJob` will be rejected with an "expired-token" error.

## References

- SV2 Job Declaration spec <https://stratumprotocol.org/specification/06-Job-Declaration-Protocol/>
- SRI Reference Implementation <https://github.com/stratum-mining/stratum>
- BIP 152 (compact-block short IDs - reused in JD message format)
- Skill: `bitcoin/mining/stratum-v2`
