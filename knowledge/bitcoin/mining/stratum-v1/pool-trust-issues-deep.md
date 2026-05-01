# Stratum V1 Pool Trust Issues - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/mining/stratum-v1`.
> Canonical source: Stratum V2 design doc, "Censorship Resistant Mining" papers
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/mining/stratum-v1/SKILL.md

## Concept

In Stratum V1, the pool dictates the entire block. The miner contributes only
hashing power against a header whose merkle_root is fully determined by
pool-supplied `coinb1`, `coinb2`, `merkle_branch`, and the miner's small
`extranonce2`. This concentrates several trust assumptions in the pool that
are economically and politically significant.

Each trust assumption maps to a concrete attack surface; Stratum V2 was
designed specifically to dissolve them.

## Walkthrough / mechanics

### Trust 1 - Coinbase output address

The pool builds the coinbase TX, including the output(s) that pay the block
reward. The miner never sees the coinbase outputs in `mining.notify` (only
the parts split around the extranonce). A malicious pool could swap the
output to its own address while telling the miner they're mining for their
own wallet. The miner cannot detect this without independently parsing
`coinb1` and `coinb2` and reconstructing the outputs.

In practice, large pools use multi-output coinbases (FPPS pools pay miners
later, not in the coinbase), so the coinbase pays the *pool*, and the pool
distributes via off-chain bookkeeping. This is by design - but it places
all unmined-block trust in the pool.

### Trust 2 - Transaction set selection

The coinbase parts and merkle branch fully fix which other transactions are
in the block. The miner cannot add or remove a tx from a pool-supplied
template; they can only abandon the job. This means the pool decides:

- Which mempool transactions get mined.
- Which addresses get blacklisted (e.g., OFAC-tagged outputs).
- Whether to mine "empty" blocks (zero-fee, just coinbase) for faster
  propagation.

Concrete public examples:
- Multiple US-based pools have admitted to filtering OFAC-listed addresses
  since 2022.
- MARA Pool published an "OFAC-compliant" empty-block experiment in 2023.

The miner has no way to override these policies short of switching pools.

### Trust 3 - Share difficulty honesty

Pools issue share difficulty via `mining.set_difficulty`. The pool could
inflate the announced difficulty to make miners' shares worth less than
they actually are. Detection: a miner can independently verify that their
submitted hash is below the announced target - but they cannot verify
that the *announced* target equals the difficulty the pool internally
counts toward payouts.

PPS-style pools mostly mitigate this with audited share logs and public
"recent shares" feeds. But the trust is irreducible without cryptographic
commitments.

### Trust 4 - Honest payout

The pool promises to pay out per its stated reward scheme (PPS, FPPS, PPLNS).
Off-chain accounting is opaque; some pools have absconded with deposits
(historically, ICCMiner, OZcoin, etc., though never on the scale of an
exchange exit). Modern major pools have track records but are still
custodial.

### Trust 5 - Worker identity

Authorization is plaintext username. The pool can't prove who you are; you
can't prove what hashes you submitted. A MITM on a non-TLS Stratum
connection can swap your worker name and route your hashrate.

### Trust 6 - Selfish-mining via pool size

If a single pool exceeds 25-33% of the network hashrate, classical
selfish-mining attacks become profitable. Stratum V1 makes consolidation
easy because miners only need a TCP endpoint - no template work, no
bandwidth scaling. This is partly why hashrate pooled into Foundry and
AntPool reaches >50% combined as of 2025.

## Worked example

A miner at 200 TH/s connecting to `stratum+tcp://pool.example.com:3333`
submits 10000 shares per hour at vardiff. Each share carries:

- `worker` (plaintext)
- `job_id` (pool-controlled)
- `extranonce2`, `ntime`, `nonce` (miner-controlled, but only over a tiny
  search space)

The miner's economic exposure is roughly `payout_per_hour =
shares_per_hour * D_share * 2^32 * subsidy_plus_fees / 600 /
network_difficulty / 2^32 = ...` - but the miner cannot independently audit:

1. Whether their share count matches what the pool counts.
2. Whether the pool's network-target submission included their found
   block (a "block-withholding" attack).
3. Whether the pool correctly prorated transaction fees to FPPS miners.

A block withholding attack works as follows: the pool (or a miner mining at
a competing pool) accepts shares but discards any that hit network target.
The attacker's pool gets share credit but loses block reward; this is
profitable only when attacking a *competitor* pool (a sabotage attack).
Stratum V1 has no defence; SV2 doesn't fully solve it either, but the
miner's local block discovery via Job Declaration prevents the attacker
from hiding the block from the network.

## Common pitfalls

- **Assuming pool fee = total cost** - censorship and address-swap risk
  aren't priced into the headline fee. A 2% fee on a censoring pool may
  effectively cost more than 4% on a non-censoring one.
- **Treating SSL/stunnel as a solution** - TLS prevents MITM but doesn't
  reduce pool trust. The pool still controls the template.
- **Ignoring block withholding** - small pools have been ruined by
  competing pools' miners submitting shares with HW-error rates near 100%.
- **Forgetting payout cycles** - a pool that pays daily holds up to a day's
  worth of your earnings as float. A pool that pays per-block holds less
  but exposes you to per-block solvency risk.
- **Trusting "non-custodial" claims with V1** - even Ocean/Public-Pool
  must build the coinbase if using V1; only SV2 + Job Declaration removes
  the trust.

## References

- "Stratum V2: A Specification" - Braiins Pool whitepaper (2019)
- Bitcoin Optech newsletter coverage of OFAC pool filtering (2022-2023)
- "Block Withholding Attacks" - Eyal & Sirer (2014)
- Skill: `bitcoin/mining/stratum-v2`
- Skill: `bitcoin/mining/decentralized-pools`
