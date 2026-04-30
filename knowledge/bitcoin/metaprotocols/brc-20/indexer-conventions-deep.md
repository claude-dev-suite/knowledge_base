# BRC-20 Indexer Conventions - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/metaprotocols/brc-20`.
> Canonical source: https://github.com/bestinslot-xyz/OPI
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/metaprotocols/brc-20/SKILL.md

## Concept

BRC-20 has no on-chain validator; correctness lives in indexer rules.
The de-facto reference is the Open Protocol Indexer (OPI) maintained
by Best in Slot, which produces a deterministic state root after each
block and is followed by most marketplaces. Diverging from OPI rules
means your wallet will display balances no one else recognises, and
exchange deposits will fail. Understanding the rules is therefore
practical, not academic.

## Walkthrough / mechanics

Order of evaluation per block:

1. Walk the block's inscriptions in **inscription number order** (the
   order ord assigns them, which is input-then-witness-envelope
   order across the block).
2. For each inscription, parse body as JSON; ignore on parse failure.
3. Apply rules:
   - `deploy`: ticker length 1..4 chars (originally exactly 4; some
     indexers allow 5 for ticker `5-byte` after a community rule
     change in 2024). Ticker stored case-insensitively. First valid
     deploy wins.
   - `mint`: must follow an existing deploy. `amt <= lim`. Cap
     enforcement: if `total + amt > max`, indexer caps to remaining
     supply (some indexers reject entirely; OPI canonical is "cap to
     remaining"). FCFS by inscription number.
   - `transfer`: holder must have `available >= amt`. Lock that amt
     into transferable. The inscription itself is the bearer asset.
4. After mints/transfers in this inscription, also process any
   inscription-as-input movements in this block: if a transferable
   inscription is spent in a tx, its amt moves from sender's
   transferable to recipient's available.

Self-issued numbers (`brc20-deploy-inscribe-blockheight` etc.) and
sat-tracking are pulled from the underlying ord index.

## Worked example  (encoded data, real txs, configs)

Cap-overflow scenario for ticker `ordi` (max 21_000_000, lim 1_000):

```
mint #20_999_500: amt=1000 -> total 20_999_500 + 1000 = ok (21_000_500???)
                   wait, total before this mint = 20_999_500
                   total + 1000 = 21_000_500 > 21_000_000
                   OPI rule: cap to 500 (remaining)
                   user receives 500 ordi, not 1000
mint #20_999_500+1: amt=1000 -> total = 21_000_000, cap reached
                   rejected entirely (0 ordi credited)
```

Decimal handling for ticker `sats` (`dec=18`, `lim=100_000_000`):

```
mint payload: { ..., "amt": "100000000" }
indexer parses as decimal "100000000" with 18 decimals = 100M.satoshis
availability before: 0
availability after:  100_000_000
```

Indexer state root format (OPI canonical):

```
state_root = sha256(
    sorted(
        for each (address, ticker):
            sha256(address || ticker || available || transferable)
    )
)
```

Two indexers that produce the same state root after the same block
have identical state.

## Trade-offs / pitfalls

- **Inscription ordering bugs**: ord's inscription numbering changed
  semantics around the "jubilee" block (824_544). Pre-jubilee
  "cursed" inscriptions had negative numbers; post-jubilee they were
  un-cursed. Indexers needed code paths for both eras.
- **Reorg handling**: a 6-block reorg may invalidate mints. OPI
  stores per-block diffs and rolls back; naive indexers that store
  only the latest state cannot.
- **OFAC compliance**: some indexers blacklist addresses or
  inscriptions. Forks of OPI maintain unfiltered state; your dApp
  should pin to a specific source.
- **Performance**: a full BRC-20 indexer plus ord index sits at ~1.2
  TB and takes ~5-7 days to sync from genesis on a NVMe machine.
  Many wallets use third-party JSON-RPC instead.
- **Privacy leak**: the address-to-ticker-to-balance map is fully
  public and trivially correlatable across wallets. Mixing BRC-20
  with privacy tooling (CoinJoin, silent payments) is hostile - the
  inscription itself is a UTXO that loses its identity if mixed with
  others.
- **Tick squatting**: as soon as someone notices an under-deployed
  ticker on social media, bots inscribe it within blocks. Treat any
  deploy as front-runnable.
- **Two-step transfer hazards**: leaving step 1 inscribed but never
  step-2 sending it leaves the balance permanently locked in
  "transferable" until the inscription is moved or burned.

## References

- OPI: https://github.com/bestinslot-xyz/OPI
- Layer1 BRC-20 docs: https://layer1.gitbook.io/layer1-foundation/protocols/brc-20
- Inscription jubilee: https://docs.ordinals.com/inscriptions/cursed.html
