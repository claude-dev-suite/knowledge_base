# BIP125 RBF Rules Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/rbf-cpfp`.
> Canonical source: BIP125 https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/rbf-cpfp/SKILL.md

## Concept

BIP125 defines **opt-in Replace-by-Fee**: a transaction is replaceable if
at least one of its inputs has `nSequence < 0xfffffffe`.

Bitcoin Core's actual policy has since drifted away from the BIP. The
live specification is `doc/policy/mempool-replacements.md` in the Core
tree, where the rules keep BIP125's numbering but two of them now read
"(Removed)". Core 28.0 made full-RBF the default node policy, so
non-opt-in transactions can also be replaced; Core 29.0 removed the
`-mempoolfullrbf` opt-out entirely (PR #30592), leaving no supported way
to run default Core with opt-in-only replacement; and Core 31.0
(April 2026) reimplemented the mempool as a *cluster mempool*, which
removed one rule and rewrote two others. The rules that remain still
protect against denial-of-service attacks via mempool churn, and every
wallet building an RBF replacement still has to satisfy them.

## The rules (mempool perspective, Bitcoin Core 31.1, July 2026)

For a candidate replacement `R` of original transaction set `O = {o1, ..., on}`
— the directly conflicting transactions plus their in-mempool descendants:

**Rule 1 — (Removed).**
This was the opt-in signalling requirement: `O` had to have at least one
input with `nSequence < 0xfffffffe`. Full-RBF became the default in
Core 28.0 and PR #30592 dropped signalling as a requirement altogether,
so a default-policy node replaces regardless of nSequence — see
[Signalling is now a fingerprint](#signalling-is-now-a-fingerprint).

**Rule 2 — (Removed in Core 31.0, April 2026).**
This was "no new unconfirmed inputs": `R` could not spend a UTXO created
by an unconfirmed transaction that was not also being replaced. Core's
own rationale recorded it as a temporary restriction dating from before
the mempool tracked ancestor feerates. `R` may now draw on unconfirmed
inputs that the original did not spend.

**Rule 3 — Higher absolute fee.**
`fee(R) >= sum(fee(oi))`, and strictly greater once rule 4's surcharge
is applied.

**Rule 4 — Pays for relay bandwidth.**
`fee(R) >= sum(fee(oi)) + R.vsize * incrementalrelayfee`. The default
`incrementalrelayfee` was lowered from 1 sat/vB to **0.1 sat/vB** in
Core 30.0 (October 2025, PR #33106, together with `minrelaytxfee` and
`blockmintxfee`), so a 200 vB replacement must add at least 20 sats over
the sum of replaced fees on a 30.0+ node, and 200 sats on an older one.
Size the surcharge for the highest `incrementalrelayfee` you expect your
peers to run, not the lowest.

**Rule 5 — Conflicts with no more than 100 distinct clusters.**
A *cluster* is a connected component of the mempool's parent/child
graph. Before 31.0 this rule counted transactions instead: the sum of
the directly conflicting transactions' descendant counts had to be
<= 100.

**Rule 6 — Strictly improves the mempool's feerate diagram.**
Added in 31.0 with cluster mempool. The mempool that results from the
replacement must be strictly better, by feerate diagram, than the one
before it; this eliminates the known cases where a replacement left the
mempool worse off for a miner. For a **singleton** — a transaction alone
in its cluster — a higher fee *and* a higher feerate is sufficient, which
is the case most wallet fee bumps fall into. Before 31.0, rule 6 was the
narrower Core 0.12 constraint that `R` beat the feerate of every directly
conflicting transaction.

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
| 1 | Removed — no replaceability signal is required. N/A |
| 2 | Removed — a new unconfirmed input would be allowed. N/A |
| 3 | fee(R) = 5000 > 1000 = fee(o1). PASS |
| 4 | 5000 - 1000 = 4000 > 140 (vsize) * 0.1 sat/vB = 14. PASS |
| 5 | One conflicting tx with no descendants, so one cluster. PASS |
| 6 | o1 is a singleton; R has both higher fee and higher feerate. PASS |

Then test before broadcast:

```bash
bitcoin-cli testmempoolaccept "[\"$signed_hex\"]"
# [{"txid":"...","allowed":true,"vsize":140,"fees":{...}}]
```

## Pinning attacks (and why rules matter)

Rules 3 and 4 (pay the sum of the replaced fees, plus a relay surcharge)
create the **transaction pinning** problem: an attacker spends a tx and
hangs many cheap descendants off it. The honest party who wants to bump
fee must now pay the sum of all those descendants' fees plus relay
surcharge — a prohibitive amount. Rule 5's cap adds a second edge: past
100 distinct conflicting clusters the replacement is refused outright,
whatever it pays.

Example: HTLC sender spams cheap children of the timeout output. Honest
receiver trying to claim with their preimage tx must outbid the sum of
all of them → loses.

Cluster mempool did not fix this. It bounds how large a pin can grow —
since Core 31.0 a cluster is capped at 64 transactions and 101 kB of
virtual size — but within those bounds the fee arithmetic is unchanged,
and rule 6 adds one more way for an otherwise-adequate replacement to be
rejected.

Mitigations are the subject of TRUC v3 (BIP431) — see related article
[../../wallets/rbf-cpfp/lightning-anchor-bumping.md](lightning-anchor-bumping.md).

## What full-RBF changed

Pre-Core-28: `R` could only replace `O` if at least one input of any
`oi` opted in via `nSequence < 0xfffffffe`. Many merchant flows relied
on this — they accepted "non-replaceable" zero-conf transactions as
final.

Post-Core-28: any tx can be replaced if the remaining rules hold.
Wallets must treat **all** zero-conf transactions as soft. Some payment
processors require N confirmations now; others rely on the originating
wallet trusting their own funds.

## Signalling is now a fingerprint

If the signal no longer decides replaceability, the only thing the
`nSequence` value still carries is information about *which wallet built
the transaction*. That turned it into a wallet-fingerprinting surface.

In May 2026 rkrux opened Bitcoin Core #35405 ("wallet: switch default
optinrbf from true to false") to have the Core wallet emit `MAX-1`
instead — where `MAX` is `0xffffffff`, so `MAX-1 = 0xfffffffe`
(non-signalling) and `MAX-2 = 0xfffffffd` (signalling) — and in June 2026
took the broader proposal, that wallets stop signalling opt-in RBF and
converge on one shared value, to the Bitcoin-Dev mailing list.

Murch and Electrum Wallet contributor SomberNight pushed back: `MAX-2`
is already the dominant value, used by roughly 75–78% of transactions as
of June 2026 — the mainnet-observer chart, read as ~75% by Optech #410
and as ~78% in the #35405 thread itself — and by nearly all Electrum
transactions. Because most transactions still signal, moving Core to a
non-signalling `MAX-1` would make Core's transactions *stand out* rather
than blend in — the opposite of the goal. rkrux closed the PR in light
of that feedback.

Net effect for wallet authors, as of September 2026:

| Value | Meaning | Ecosystem position |
|-------|---------|--------------------|
| `0xffffffff` (`MAX`) | final; disables nLockTime | non-signalling; distinctive |
| `0xfffffffe` (`MAX-1`) | non-signalling, nLockTime still usable | minority; distinctive |
| `0xfffffffd` (`MAX-2`) | BIP125 opt-in signal | ~75–78% of txs (June 2026); the blend-in choice |

Bitcoin Core's wallet still defaults to signalling
(`DEFAULT_WALLET_RBF = true`, emitting
`MAX_BIP125_RBF_SEQUENCE = 0xfffffffd` in `src/util/rbf.h`). Emit
`0xfffffffd` on every input unless you specifically need `nSequence` for
a BIP68 relative timelock; picking a distinctive value to "opt out" of
replacement achieves nothing under full-RBF policy and leaks which
software you run.

## Common pitfalls

- Assuming a replacement may never **add** a new unconfirmed input: that
  was rule 2, removed in Core 31.0 (April 2026). Default policy allows it
  now, so only code that must also work against pre-31.0 nodes still has
  to draw the extra fee from confirmed UTXOs only.
- Budgeting the rule 4 surcharge at 1 sat/vB: the `incrementalrelayfee`
  default has been 0.1 sat/vB since Core 30.0 (October 2025), so a node
  may accept a cheaper bump than a hard-coded 1 sat/vB assumption builds.
- Assuming a higher fee and feerate is always enough: that holds for a
  singleton, but a transaction with unconfirmed relatives must also
  improve the whole mempool's feerate diagram (rule 6, Core 31.0+).
  `getmempoolcluster` shows what else is in the cluster.
- Building `R` that pays the same fee as `O` (mistaking "bump" for
  "rebuild") → rejected (rules 3 and 4). Always increase strictly.
- Forgetting incremental relay: if `R.vsize > O.vsize`, the surcharge
  scales with the size delta. A larger replacement (e.g., adding
  inputs) needs proportionally more fee.
- Computing fee in `size` vs `vsize`: SegWit txs have witness data
  excluded from `vsize`. Underpaying because you used `size` is a
  classic bug.
- Ignoring full-RBF in zero-conf flows → loss of funds. Treat
  confirmed only as final.
- Setting `nSequence` to `0xffffffff` or `0xfffffffe` believing it makes
  the transaction non-replaceable → it does not under default policy,
  and it fingerprints the wallet as a minority implementation.
- Bumping a tx that has descendants not in your wallet (e.g., the
  recipient's tx that spends to a third party): you replace your own
  tx, but the recipient's descendant becomes unconfirmed/orphan
  forever. This is fine technically; not fine ethically.

## References

- BIP125: https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki
- Core's live RBF policy, rules 1 and 2 marked "(Removed)": https://github.com/bitcoin/bitcoin/blob/v31.1/doc/policy/mempool-replacements.md
- Bitcoin Core 31.0 release notes (cluster mempool, feerate-diagram RBF, CPFP carveout removal): https://github.com/bitcoin/bitcoin/blob/v31.0/doc/release-notes.md
- Bitcoin Core #33106 (blockmintxfee / incrementalrelayfee / minrelaytxfee lowered, 30.0): https://github.com/bitcoin/bitcoin/pull/33106
- Cluster mempool overview (Delving Bitcoin): https://delvingbitcoin.org/t/an-overview-of-the-cluster-mempool-proposal/393
- Pinning attack discussion: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-June/017997.html
- Bitcoin Core 28 release notes (mempoolfullrbf default): https://github.com/bitcoin/bitcoin/blob/v28.0/doc/release-notes.md
- Bitcoin Core 29 release notes (mempoolfullrbf option removed, #30592): https://github.com/bitcoin/bitcoin/blob/v29.0/doc/release-notes.md
- rkrux, "Removing RBF signalling from wallet transactions" (Bitcoin-Dev, June 2026): https://groups.google.com/g/bitcoindev/c/C7zNIk8llew/m/YAdpwe33AgAJ
- Bitcoin Core #35405 (closed): https://github.com/bitcoin/bitcoin/pull/35405
- Optech Newsletter #410 (19 June 2026), summary of the discussion: https://bitcoinops.org/en/newsletters/2026/06/19/
- mainnet-observer, transactions signalling explicit RBF: https://mainnet.observer/charts/transactions-signaling-explicit-rbf/
- See also: [package-cpfp-bip331.md](package-cpfp-bip331.md), [lightning-anchor-bumping.md](lightning-anchor-bumping.md)
