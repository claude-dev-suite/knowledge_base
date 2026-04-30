# BIP125 RBF Rules Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/rbf-cpfp`.
> Canonical source: BIP125 https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/rbf-cpfp/SKILL.md

## Concept

BIP125 defines **opt-in Replace-by-Fee**: a transaction is replaceable if
at least one of its inputs has `nSequence < 0xfffffffe`. A replacement
transaction must satisfy five rules to be accepted by the mempool.

Bitcoin Core 28.0 made full-RBF the default node policy, so non-opt-in
transactions can also be replaced. But the **five replacement rules**
remain, because they protect against denial-of-service attacks via
mempool churn. Every wallet building an RBF replacement still has to
satisfy them.

## The five rules (mempool perspective)

For a candidate replacement `R` of original transaction set `O = {o1, ..., on}`
where each `oi` is a conflicted ancestor:

**Rule 1 — Either RBF-signaled or full-RBF.**
Either `O` has at least one input with `nSequence < 0xfffffffe`, or the
node operates in full-RBF mode (Core 28+ default).

**Rule 2 — No new unconfirmed inputs.**
`R` cannot spend a UTXO that was created by an unconfirmed transaction
not also being replaced. This prevents an attacker from forcing the node
to do a deep mempool walk validating the replacement.

**Rule 3 — Higher absolute fee.**
`fee(R) > sum(fee(oi))` strictly.

**Rule 4 — Pays for relay bandwidth.**
`fee(R) >= sum(fee(oi)) + R.vsize * incrementalrelayfee`. The default
`incrementalrelayfee = 1 sat/vB`, so a 200 vB replacement must add at
least 200 sats over the sum of replaced fees.

**Rule 5 — Replaces no more than 100 conflicting transactions.**
The total descendant set of `O` (including `O` itself) must be <= 100.

A subtle additional constraint exists: `R` must have a **strictly higher
feerate** than each individual replaced tx (Rule 6 in some references).
This was added in Core 0.12 to stop free relay attacks.

## Worked example

Suppose you sent:

```
o1: spends UTXO_A (1.0 BTC), pays Bob 0.4 BTC, change 0.59999 BTC, fee 0.00001 (1000 sats, vsize 140)
    nSequence on input = 0xfffffffd  (RBF opt-in)
```

Now you want to bump fee to 5000 sats (~36 sat/vB) without changing the
recipient amount.

```bash
# Build replacement using Core's helper
bitcoin-cli -rpcwallet=hot psbtbumpfee <txid_o1> '{"fee_rate": 36}'

# Or manually via PSBT:
psbt=$(bitcoin-cli -rpcwallet=hot walletcreatefundedpsbt \
  '[{"txid":"abc...","vout":0}]' \
  '[{"<bob_addr>":0.4}]' \
  0 \
  '{"feeRate":0.000036, "replaceable":true}')
```

Validate against the rules:

| Rule | Check |
|------|-------|
| 1 | o1 had nSequence=0xfffffffd → opt-in. PASS |
| 2 | R reuses UTXO_A; no new mempool-derived inputs. PASS |
| 3 | fee(R) = 5000 > 1000 = fee(o1). PASS |
| 4 | 5000 - 1000 = 4000 > 140 (vsize) * 1 sat/vB = 140. PASS |
| 5 | Single tx replaced, descendants 0. PASS |

Then test before broadcast:

```bash
bitcoin-cli testmempoolaccept "[\"$signed_hex\"]"
# [{"txid":"...","allowed":true,"vsize":140,"fees":{...}}]
```

## Pinning attacks (and why rules matter)

Rule 5 (max 100 replaced) and Rule 4 (sum-of-replaced fees) combine to
create the **transaction pinning** problem: an attacker spends a tx with
many cheap descendants. The honest party who wants to bump fee must now
pay the sum of all those descendants' fees plus relay surcharge — a
prohibitive amount.

Example: HTLC sender spams 99 cheap children of the timeout output.
Honest receiver trying to claim with their preimage tx must outbid the
sum of all 99 → loses.

Mitigations are the subject of TRUC v3 (BIP431) — see related article
[../../wallets/rbf-cpfp/lightning-anchor-bumping.md](lightning-anchor-bumping.md).

## What full-RBF changed

Pre-Core-28: `R` could only replace `O` if at least one input of any
`oi` opted in via `nSequence < 0xfffffffe`. Many merchant flows relied
on this — they accepted "non-replaceable" zero-conf transactions as
final.

Post-Core-28: any tx can be replaced if rules 2-6 hold. Wallets must
treat **all** zero-conf transactions as soft. Some payment processors
require N confirmations now; others rely on the originating wallet
trusting their own funds.

## Common pitfalls

- Sending a replacement that **adds** a new unconfirmed input from
  someone else's tx → rejected (rule 2). To bump fee, draw from your
  own wallet's confirmed UTXOs only.
- Building `R` that pays the same fee as `O` (mistaking "bump" for
  "rebuild") → rejected (rule 3). Always increase strictly.
- Forgetting incremental relay: if `R.vsize > O.vsize`, the surcharge
  scales with the size delta. A larger replacement (e.g., adding
  inputs) needs proportionally more fee.
- Computing fee in `size` vs `vsize`: SegWit txs have witness data
  excluded from `vsize`. Underpaying because you used `size` is a
  classic bug.
- Ignoring full-RBF in zero-conf flows → loss of funds. Treat
  confirmed only as final.
- Bumping a tx that has descendants not in your wallet (e.g., the
  recipient's tx that spends to a third party): you replace your own
  tx, but the recipient's descendant becomes unconfirmed/orphan
  forever. This is fine technically; not fine ethically.

## References

- BIP125: https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki
- Pinning attack discussion: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-June/017997.html
- Bitcoin Core 28 release notes (mempoolfullrbf default): https://github.com/bitcoin/bitcoin/blob/v28.0/doc/release-notes.md
- See also: [package-cpfp-bip331.md](package-cpfp-bip331.md), [lightning-anchor-bumping.md](lightning-anchor-bumping.md)
