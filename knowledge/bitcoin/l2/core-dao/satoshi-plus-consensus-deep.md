# Core DAO Satoshi Plus Consensus - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/core-dao`.
> Canonical source: https://github.com/coredao-org/core-chain
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/core-dao/SKILL.md

## Concept

Satoshi Plus is a validator-election scheme, not a block-production
scheme. Core produces blocks with a Geth-derived proof-of-staked-
authority engine (`params/config.go` line 626 prints `Consensus:
Satoshi (proof-of-staked--authority)`, double hyphen verbatim
upstream, checked 15 September 2026); what Satoshi Plus decides is
*which* addresses are in the authority set for the next round, and
how the round's rewards split.

Three independent delegation markets feed one hybrid score:

1. **Delegated hash power (DPoW)** — Bitcoin miners commit to a Core
   validator from inside their Bitcoin coinbase transaction.
2. **Delegated CORE stake (DPoS)** — ordinary token delegation.
3. **Delegated BTC stake** — BTC timelocked on Bitcoin under CLTV,
   with the delegation target written into an `OP_RETURN`.

All three are *proven on Core* rather than attested by a committee,
because Core runs an on-chain Bitcoin light client.

Round cadence, from `CoreChainConfig` (fetched 15 September 2026):

```
Period: 3       // seconds per block
Epoch:  200     // blocks
Round:  86400   // seconds -- one election round per 24 hours
```

## Walkthrough / mechanics

### The Bitcoin light client

Per the `core-chain` README the light client is split in two:

- a **stateless precompiled contract** doing Bitcoin header
  verification, coinbase transaction verification and Merkle proof
  verification;
- a **stateful Solidity contract** storing Bitcoin block hashes and
  headers.

Relayers feed Bitcoin headers into the stateful contract. Anything
wanting to prove a Bitcoin fact to Core — "this coinbase delegated
to me", "this CLTV output exists" — supplies the transaction plus a
Merkle branch, and the precompile checks it against a stored header.
This is SPV, with the usual SPV caveat: it proves inclusion under
the heaviest chain the relayers have reported, not validity of the
Bitcoin transaction's inputs.

### Genesis system contracts

Staking state lives in genesis-deployed contracts
(`core/systemcontracts/const.go`, master, 15 September 2026):

| Contract | Address | Role |
|---|---|---|
| `PledgeCandidateContract` | `0x…1007` | validator candidacy |
| `StakeHubContract` | `0x…1010` | router across stake types |
| `BTCAgentContract` | `0x…1013` | BTC delegation agent |
| `BTCStakeContract` | `0x…1014` | timelocked BTC stake accounting |
| `BTCLSTStakeContract` | `0x…1015` | BTC liquid-staking module |
| `BTCLSTTokenContract` | `0x…10001` | the BTC LST itself |

`StakeHub` is the aggregation point: the three stake classes are
separate accounting systems that meet there to produce a single
per-validator weight and a single per-round reward split.

### The BTC staking output

Core's reference tooling is `coredao-org/btc-staking-tool`. A stake
transaction has two relevant outputs: a CLTV-locked value output and
an `OP_RETURN` carrying the delegation.

The lock script is a CLTV-wrapped P2PKH. The tool's own README shows
this redeem script for `--locktime 1712847585`:

```
04 e1fa1766                                   <locktime> (LE)
b1                                            OP_CHECKLOCKTIMEVERIFY
75                                            OP_DROP
76                                            OP_DUP
a9 14 13597ff6de21ec27b297e296ef03b9897459086b  OP_HASH160 <pubKeyHash>
88                                            OP_EQUALVERIFY
ac                                            OP_CHECKSIG
```

`0x6617fae1` = 1712847585, matching the requested locktime. The
script is then wrapped as P2SH or P2WSH, and the tool prints both
the address and the redeem script — the redeem script is required
to build the later redeem transaction, so losing it loses the exit
path even though the key is intact.

`src/constant.ts` also defines `MULTI_SIG_SCRIPT` and
`MULTI_SIG_HASH_SCRIPT` variants, so the locked branch can be an
m-of-n rather than a single key.

### The OP_RETURN delegation payload

`buildOPReturnScript` in `src/script.ts` concatenates, in order:

| Field | Width | Notes |
|---|---|---|
| flag | 4 bytes | ASCII `SAT+` |
| version | 1 byte | currently `1` |
| chain ID | 2 bytes | `1116` mainnet, `1114` testnet2 |
| reward address | 20 bytes | Core EVM address receiving rewards |
| validator address | 20 bytes | Core validator being delegated to |
| core fee | 1 byte | relayer fee, `0` in the reference tool |
| tail | variable | the redeem script for the non-multisig P2PKH case, otherwise the 4-byte locktime |

So the Bitcoin transaction is self-describing: a relayer can scan
for `SAT+`, pull out the validator and reward addresses, and submit
the transaction plus Merkle proof to `BTCStakeContract`, which
re-verifies it through the light-client precompile.

Note that the staker's BTC key and the Core reward address are
unrelated. The BTC stays spendable only by the Bitcoin key after
locktime; the CORE-denominated rewards accrue to the EVM address
named in the `OP_RETURN`.

### Redeeming

After the locktime the staker spends the CLTV output back to
themselves with `redeem --redeemscript … --destaddress …`. There is
no early-exit path and no cooperative-close path — CLTV is absolute
and unconditional. The Core-side stake weight simply stops counting.

The tool's README also notes a practical floor on lock duration: on
Devnet only staking transactions longer than ~6 hours (locktime
minus confirmation time at 4 blocks) are counted as feasible, and it
recommends locking for more than 8 hours.

## Pitfalls / gotchas

- **Hash-power delegation is not hash-power security.** Miners
  signal a preference inside a coinbase they were already mining.
  Nothing about that makes Core's own chain expensive to reorg; Core
  blocks are produced by Core's authority set at 3s intervals.
  Delegated PoW buys Sybil-resistant *validator election*, which is
  a real property, but it is not "secured by Bitcoin".
- **SPV relayer liveness.** Every Bitcoin-derived fact reaches Core
  through relayed headers. Stalled relayers stall new delegations
  and BTC-stake accounting.
- **Losing the redeem script bricks the exit.** The private key is
  necessary but not sufficient for a P2SH/P2WSH CLTV spend. Back up
  the redeem script hex alongside the key.
- **Absolute locktime, no early unbond.** Unlike a CSV-based
  unbonding period, the staker cannot start an exit early. Anything
  needing liquidity has to go through the `BTCLST*` contracts, which
  swaps Bitcoin-enforced custody for EVM contract risk — the exact
  trade the non-custodial staking was supposed to avoid.
- **Locktime granularity.** Very short locks are ignored; see the
  ~6-hour feasibility floor above. Do not assume a 1-hour lock
  registers.
- **Do not conflate chain DeFi TVL with BTC delegated.** DefiLlama's
  CORE chain DeFi TVL was ~$4.32M on 15 September 2026; delegated
  BTC is tracked separately and is a much larger number. Cite the
  metric you actually measured.

## References

- `coredao-org/core-chain` — README, `params/config.go`,
  `core/systemcontracts/const.go` (all fetched 15 September 2026).
- `coredao-org/btc-staking-tool` — README, `src/script.ts`,
  `src/transaction.ts`, `src/constant.ts`.
- `coredao-org/core-genesis-contract` — system contract sources.
- Core white paper: https://whitepaper.coredao.org/
