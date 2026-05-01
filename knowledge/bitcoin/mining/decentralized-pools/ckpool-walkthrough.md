# CKPool Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/mining/decentralized-pools`.
> Canonical source: <https://bitbucket.org/ckolivas/ckpool> and `solo.ckpool.org` documentation
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/mining/decentralized-pools/SKILL.md

## Concept

CKPool is Con Kolivas's open-source pool server, written in C and designed
for low overhead. Three deployment modes are common:

- `ckpool-solo` - each miner runs their own instance against their bitcoind
  and gets the entire reward when their hardware finds a block.
- `ckpool` (shared) - traditional pool-style with PPLNS payouts.
- `ckpool-proxy` - upstream pool relay for low-bandwidth sites.

The solo flavour is the most popular because it embodies "decentralised
pool" semantics: the pool is just *your* server, the network sees only
your coinbase address, and there is no shared payout liability.

## Walkthrough / mechanics

### Process model (solo)

```
[Your bitcoind]
   |- ZMQ hashblock + RPC
   v
[ckpool-solo] -- TCP 3333 --> [Your ASICs over Stratum V1]
```

Single binary. ckpool spawns processes:
- `proxyrecv` - reads notifications from upstream (or bitcoind in solo).
- `connector` - accepts ASIC TCP connections.
- `stratifier` - issues `mining.notify`, validates shares.
- `generator` - composes block templates from `getblocktemplate`.

### Configuration

Minimal `ckpool.conf` for solo:

```json
{
  "btcd": [
    {
      "url": "127.0.0.1:8332",
      "auth": "rpcuser",
      "pass": "rpcpass",
      "notify": true
    }
  ],
  "btcaddress": "bc1q...minerwallet",
  "btcsig": "/solo.ckpool/",
  "blockpoll": 100,
  "update_interval": 30,
  "serverurl": [
    "0.0.0.0:3333"
  ],
  "mindiff": 1024,
  "startdiff": 65536,
  "logdir": "/var/log/ckpool"
}
```

`btcaddress` is the only payout address - everything goes there. `btcsig`
is the coinbase tag (visible in block explorer's coinbase scriptSig).
`blockpoll` is the ms interval ckpool re-asks bitcoind for updates;
`update_interval` is how often it issues fresh templates regardless.

### Stratum messages

ckpool speaks ordinary Stratum V1:

- `mining.subscribe` - issues 4-byte extranonce1.
- `mining.set_difficulty` - vardiff on, starting at `startdiff`.
- `mining.notify` - new template (every 30 sec or on chain tip).
- `mining.submit` - validated locally; if hash <= block_target, ckpool
  immediately runs `submitblock` against bitcoind.

Vardiff is implemented per-connection: ckpool measures share submission
rate and adjusts `D_share` so each miner submits ~1 share/30 seconds (the
target window is configurable via `update_interval`).

### Block discovery path

When an ASIC submits a share that meets network target:

1. `stratifier` flags the share as a candidate block.
2. ckpool reconstructs the full block from coinbase + branches + tx data
   stored at template build time.
3. ckpool calls bitcoind's `submitblock` RPC with the serialised block.
4. bitcoind validates, accepts, and broadcasts to peers.
5. ckpool logs `BLOCK ACCEPTED` and resets work.

The miner's payout is **the coinbase output to `btcaddress`** - direct,
no off-chain accounting. After 100 confirmations the BTC is mature
spendable balance.

### Shared (PPLNS) mode

For the `ckpool` (non-solo) flavour:

- Multiple miners connect to the same instance.
- Shares are credited per-miner.
- When a block is found, payout = `(subsidy + fees) * (your_shares_in_last_N
  / N)`.
- N defaults to a function of the most recent difficulty (so window
  duration is roughly constant).
- Pool operator's address is the coinbase recipient; the pool then
  distributes manually or via a payout script after maturity.

This mode has the same trust assumptions as a normal pool (operator must
honestly distribute) but with open-source code that miners can audit.

### Public solo.ckpool.org

`solo.ckpool.org` is a public endpoint that *runs the solo flavour* on
behalf of users: you connect with your wallet address as the username
and any worker name, and if your hashrate finds a block, the coinbase
goes to your address minus a 2% fee that goes to Con Kolivas. There is
no off-chain account or balance - pure on-chain payout.

```
stratum+tcp://solo.ckpool.org:3333
username: bc1q...yourwallet.workername
password: x
```

If you're at 200 TH/s and the public solo pool has 5 EH/s of users
combined, your daily probability of finding a block is `~144 * 200e12 /
8e20 = 3.6e-5 = 0.0036%`. Statistically a 200 TH/s solo miner finds a
block roughly once per 76 years - high variance, but the upside is the
full ~3.18 BTC reward direct to wallet.

## Worked example

A small farm with 5x Antminer S19j Pro (104 TH/s each = 520 TH/s):

1. Run bitcoind on `node1`, ckpool-solo on `node2`.
2. Configure ckpool with `btcaddress = bc1q...farm`.
3. Point all 5 ASICs to `stratum+tcp://node2:3333`, username
   `farm.s19-1` through `farm.s19-5`.
4. ckpool aggregates 520 TH/s of work against templates from your
   bitcoind. Each miner submits ~3000 shares/hour at vardiff.
5. Expected solo block time: `95e12 * 2^32 / 520e12 ~ 7.85e8 sec
   ~ 24.9 years`. Variance is enormous; could be tomorrow or 2050.
6. If a block is found: coinbase pays `bc1q...farm` directly the full
   reward (~3.18 BTC). No fees, no withholding.

Compare with FPPS @ 2%: same miner earns `3.185 * 0.98 * 520/9.5e8 *
144 / day = ~2710 sats/day` smoothly, totalling ~3.18 BTC over 25 years
*on average* (matching expectation). Solo's mean is the same; variance
is the difference.

## Common pitfalls

- **Stale templates from slow `update_interval`** - if you set `update_interval = 300`,
  templates may go stale during high-fee periods, missing fresh
  high-fee txs.
- **bitcoind ZMQ not bound** - without `zmqpubhashblock`, ckpool falls
  back to RPC polling at `blockpoll` interval, losing 100ms per cycle.
- **Wallet vs raw address** - `btcaddress` must be an output script that
  bitcoind can encode in the coinbase. P2WPKH (`bc1q`), P2TR (`bc1p`),
  P2SH, and legacy P2PKH all work; descriptor wallets must export the
  raw address.
- **Witness commitment missing** - ckpool computes the segwit commitment
  automatically; older forks may not. Check coinbase txout 1 for OP_RETURN
  with `aa21a9ed` magic.
- **Public solo username typo** - if you mistype the wallet address, the
  block reward goes to whoever owns the typo'd address (likely no one,
  but not you). solo.ckpool.org pays out exactly what you typed.

## References

- ckpool source <https://bitbucket.org/ckolivas/ckpool>
- solo.ckpool.org <https://solo.ckpool.org/>
- Public-Pool (web-dashboard solo variant) <https://web.public-pool.io/>
- Skill: `bitcoin/mining/pool-architectures`
