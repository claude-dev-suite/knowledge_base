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
- MARA Pool (Marathon Digital) ran an "OFAC-compliant" filtered template:
  it began pointing its hashrate at the pool on 1 May 2021 and put out a
  press release on 5 May 2021. The blocks were *not* empty - block
  682,170, mined 6 May 2021 with the coinbase tag
  `\ MARA Pool - OFAC Compliant Block \`, carried 178 transactions and
  was near-full at 3,993,010 WU. What changed was the composition:
  far fewer but much larger transactions than its neighbours (178
  against 1,096 / 1,180 / 1,839 in blocks 682,169 / 682,171 / 682,172)
  earning about a sixth of their fee income (0.051 BTC against
  0.31 / 0.31 / 0.48 BTC), because transactions touching the US
  Treasury SDN list were screened out. Marathon dropped the filter on
  2 June 2021 after community backlash and moved back to stock
  Bitcoin Core 0.21.1.
- F2Pool filtered quietly and admitted it only after being caught.
  0xB10C's miningpool-observer flagged four F2Pool blocks in October
  2023 that omitted OFAC-sanctioned transactions despite having space
  and time to include them (published 20 November 2023); co-founder
  Chun Wang confirmed the patch and said it would be disabled. A
  follow-up on 16 January 2025 found 15 sanctioned transactions missing
  from 14 blocks, 11 of them F2Pool's, and concluded F2Pool "might be
  filtering ... again". No later write-up has appeared on that site as
  of September 2026.
- Single missing-transaction reports against ViaBTC and Foundry USA in the
  2023 dataset were judged likely false positives - fee-based displacement
  and late propagation, not filtering.

Note the geography does not run the way people assume: the pool caught
filtering OFAC-sanctioned transactions is Asian-founded, while the
US-based pool that filtered did so for one month in 2021 and stopped.

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
Off-chain accounting is opaque, and an unpaid miner balance is an unsecured
claim on the operator. Two documented ways that has gone wrong:

- **Breach.** OzCoin lost 923 BTC (~$135k at the time) to a server
  compromise in April 2013. The web wallet StrongCoin traced and returned
  part of it; 354.06 BTC was never recovered. Theft, not an exit scam -
  but the loss landed on pool-held funds.
- **Insolvency.** Poolin, once the largest pool by hashrate, suspended
  Poolin Wallet withdrawals on 5 September 2022 citing liquidity, issued
  ~$163.7M of 1:1 "IOU" tokens to ~11,700 customers on 13 September 2022,
  and filed Chapter 11 in New Jersey on 22 July 2026 with roughly $173M
  of debts.

Modern major pools have track records but are still custodial.

### Trust 5 - Worker identity

Authorization is plaintext username. The pool can't prove who you are; you
can't prove what hashes you submitted. A MITM on a non-TLS Stratum
connection can swap your worker name and route your hashrate.

### Trust 6 - Selfish-mining via pool size

If a single pool exceeds 25-33% of the network hashrate, classical
selfish-mining attacks become profitable. Stratum V1 makes consolidation
easy because miners only need a TCP endpoint - no template work, no
bandwidth scaling. On the 7-day window ending 16 September 2026
(mempool.space, 1071 blocks) the split is Foundry USA 26.3%, AntPool
18.9%, F2Pool 13.8%, ViaBTC 9.4%, SpiderPool 8.8%. Foundry plus
AntPool is 45.2% combined - below the >50% the pair reached in 2025 -
but the top four pools still hold 68.4% and the top five 77.2%, and
Foundry alone sits inside the 25-33% selfish-mining band.

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
- Marathon press release, 5 May 2021 (MARA OFAC Pool)
  <https://www.globenewswire.com/news-release/2021/05/05/2223800/0/en/Marathon-Digital-Holdings-Becomes-the-First-North-American-Enterprise-Miner-to-Produce-Fully-AML-and-OFAC-Compliant-Bitcoin.html>
- The Block, 2 June 2021 - Marathon stops filtering
  <https://www.theblock.co/linked/106865/marathon-ofac-bitcoin-mining-pool-taproot>
- 0xB10C, "Six OFAC-sanctioned transactions missing", 20 November 2023
  <https://b10c.me/observations/08-missing-sanctioned-transactions/>
- 0xB10C, "Fifteen OFAC-sanctioned transactions missing from blocks",
  16 January 2025
  <https://b10c.me/observations/13-missing-sanctioned-transactions-2024-12/>
- Bitcoin Magazine, 24 April 2013 - OzCoin hack and StrongCoin recovery
  <https://bitcoinmagazine.com/business/ozcoin-hacked-stolen-funds-seized-and-returned-by-strongcoin-1366822516>
- CoinDesk, 24 July 2026 - Poolin Chapter 11 filing
  <https://www.coindesk.com/markets/2026/07/24/poolin-was-bitcoin-s-biggest-mining-pool-and-now-it-s-filing-for-bankruptcy>
- "Block Withholding Attacks" - Eyal & Sirer (2014)
- Skill: `bitcoin/mining/stratum-v2`
- Skill: `bitcoin/mining/decentralized-pools`
