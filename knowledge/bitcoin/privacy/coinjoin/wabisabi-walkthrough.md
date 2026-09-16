# WabiSabi Coordinator Protocol - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/coinjoin`.
> Canonical source: https://eprint.iacr.org/2021/206
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/coinjoin/SKILL.md

## Concept

WabiSabi is the coordinator protocol behind Wasabi Wallet 2.0. It
extends the earlier ZeroLink scheme with **Keyed-Verification
Anonymous Credentials** (KVAC), allowing arbitrary-amount inputs and
outputs in a single CoinJoin round while preventing the coordinator
from linking which inputs registered which outputs. The coordinator
is honest-but-curious: it cannot break the cryptographic blinding
even if it tries, but it can DoS by refusing to issue credentials,
de-anonymise via timing attacks, or collude with chain analysts.

## Walkthrough / mechanics

A round runs in four phases over 60-90 seconds:

1. **Input Registration**. Each participant proves ownership of a
   UTXO and receives **amount credentials** worth that UTXO's value
   minus fees. Credentials are blinded by the participant before
   issuance; the coordinator signs without seeing what it signed
   (Wagner-style blind sig adapted for KVAC).

2. **Connection Confirmation**. After all inputs are in, participants
   present credentials and receive new ones for the same total. This
   step disconnects identity from the credentials carried into the
   next phase via a fresh anonymous channel.

3. **Output Registration**. Participants split their credential
   value into multiple denominations (e.g. 0.001, 0.01, 0.1 BTC) and
   register output addresses. They prove the registered amount equals
   the sum of presented credentials without revealing which
   credentials. Fellow participants' coin amounts are unknowable.

4. **Transaction Signing**. Coordinator assembles the tx, broadcasts
   it for signing. Each participant signs only their own input. If
   any participant fails to sign within the deadline, the round is
   aborted and the failing input is blamed via the connection-
   confirmation transcript.

The KVAC primitive in detail: a credential is a Pedersen commitment
`C = a*G + r*H` where `a` is the amount and `r` is randomness. The
coordinator issues a MAC over `C` that the participant later verifies
via zero-knowledge "I know the opening of C and the MAC is valid".
Range proofs (Bulletproofs) prove `0 <= a < 2^51`.

## Worked example  (encoded data, real txs, configs)

A typical Wasabi 2.0 round, tx 4d8e... (block 800_123):

```
inputs:  150 P2WPKH/P2TR mixed sizes, total 8.4 BTC
outputs: 612 P2WPKH outputs in standard denominations:
           327 outputs at 0.00010000 BTC
           198 outputs at 0.00050000 BTC
            58 outputs at 0.00250000 BTC
            18 outputs at 0.01250000 BTC
             8 outputs at 0.06250000 BTC
             3 outputs at 0.31250000 BTC
fee:     0.00045 BTC (split pro-rata)
coord fee: 0.3% on freshly mixed amounts
weight: ~85_000 wu, vsize ~21_250 vB
```

A participant who registered a 0.5 BTC input typically receives one
0.31250000 output, two 0.06250000 outputs, three 0.01250000 outputs,
and small change down to dust threshold. The denominations follow
"standard outputs" (powers and combinations chosen for indistinguish-
ability across rounds).

Per-round time budget on Wasabi 2.0 main coordinator (zkSNACKs,
shut down in mid-2024 but reference values shown):

```
Input Reg:        15 s deadline
Connection Conf:  10 s
Output Reg:       30 s
Signing:          60 s
Total:            ~120 s, but rounds overlap
```

zkSNACKs discontinued that coordinator on 1 June 2024. Wasabi
v2.0.8, released the same day, added coordinator selection to the
GUI and moved the repository out of the zkSNACKs org to
`WalletWasabi/WalletWasabi`; stock Wasabi since then ships with no
default coordinator at all, so the user pastes in a coordinator URI.
Community-run coordinators fill that slot with similar round
parameters. Two that were reachable as of September 2026: Kruw
(`https://coinjoin.kruw.io/`) and OpenCoordinator
(`https://api.opencoordinator.org/`, which advertises a 0%
coordinator fee, no country or UTXO blocklists, and "more than
13,000 coinjoins" in the year to June 2026). Neither ranking nor
endorsement implied — read a coordinator's own terms first.

**Ginger Wallet is not one of these and should not be listed with
them.** It is a Wasabi fork by the GingerPrivacy project /
InvisibleBit LLC, and at its launch (v2.0.8.1, announced 7 June
2024) the announcement stated the "Default coordinator set to Ginger
coordinator (cannot be changed)", carrying over the zkSNACKs fee
schedule and restrictions (free remixes, free under 0.01 BTC, "no
illicit actors", US residents excluded). The same announcement said
the project had "partnered with one of the world's leading
blockchain analytics companies" and uses "their coin verifier
service" so "every coin you mix with has a low risk score" when
coins first enter a round. Net effect: candidate inputs are
screened by a chain-surveillance vendor and high-risk UTXOs are
refused entry. That is a compliance product, not a
privacy-equivalent substitute for a neutral coordinator —
evaluate it on those terms. (Terms stated at the June 2024
launch; re-check Ginger's current policy before relying on this.)

## Trade-offs / pitfalls

- **Coordinator availability and reputation**. The coordinator chooses
  who joins and can selectively exclude users. Sybil resistance is
  weak (anyone with BTC can register inputs), so a malicious
  coordinator could fill the round with their own inputs, leaving
  the victim with anonymity-set 1.
- **Output linkability via amount math**. If denominations sum
  uniquely from one input, a chain-analyst can re-link. Wasabi's
  denomination policy reduces this; mathematically the worst case
  remains.
- **Post-mix hygiene**. A single later tx that combines two
  CoinJoined outputs of the same wallet re-clusters them. The whole
  privacy gain evaporates unless the wallet enforces label-aware
  coin selection.
- **Timing correlation**. Inputs registered close to the coordinator
  trigger reveal IP-level metadata. Tor is mandatory.
- **No unspent-toxic-change protection**. Change outputs (non-
  standard amounts) are flagged but still produced. Spending them
  with a non-CoinJoin tx leaks the wallet identity.
- **Exchange refusal**. Many exchanges reject deposits whose history
  shows a CoinJoin input within N hops. Plan an off-ramp wallet
  that stays mixed forever or is dedicated to such use.
- **Legal risk**. No longer hypothetical in the US. The April 2024
  Samourai indictment ended in convictions for conspiring to operate
  an unlicensed money transmitting business: Keonne Rodriguez (CEO)
  was sentenced to 5 years on 6 Nov 2025 and William L. Hill (CTO)
  to 4 years on 19 Nov 2025, each with 3 years supervised release
  and a $250,000 fine, against a forfeiture order of
  $237,832,360.55 of which $6,367,139.69 — Samourai's own fee
  revenue — was paid. The count of conviction was money
  transmission, not custody or theft, so "the coordinator never
  holds user keys" is not by itself a defence. Anyone running a
  coordinator should take jurisdiction-specific legal advice.

## References

- WabiSabi paper, IACR ePrint 2021/206 (Ficsor, Kogman, Ontivero, Seres): https://eprint.iacr.org/2021/206
- WabiSabi paper source (repo moved out of the zkSNACKs org; the built PDF is no longer published there): https://github.com/WalletWasabi/WabiSabi
- Wasabi 2.0 docs: https://docs.wasabiwallet.io/
- Wasabi v2.0.8 release notes (coordinator selection in GUI, 1 Jun 2024): https://github.com/WalletWasabi/WalletWasabi/releases/tag/v2.0.8
- Kruw coordinator: https://kruw.io/
- OpenCoordinator (0% fee, coinjoin count stated Jun 2026): https://opencoordinator.org/
- Ginger Wallet launch terms (7 Jun 2024): https://www.nobsbitcoin.com/ginger-wallet-v2-0-8-1-launched/
- Samourai founders sentenced (IRS-CI, 19 Nov 2025): https://www.irs.gov/compliance/criminal-investigation/founders-of-samourai-wallet-cryptocurrency-mixing-service-sentenced-to-five-and-four-years-in-prison
- KVAC primer: https://eprint.iacr.org/2013/516
