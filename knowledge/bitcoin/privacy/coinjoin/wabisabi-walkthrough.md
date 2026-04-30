# WabiSabi Coordinator Protocol - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/coinjoin`.
> Canonical source: https://github.com/zkSNACKs/WabiSabi/blob/master/Wabisabi.pdf
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

After zkSNACKs shutdown, community coordinators (Kruw, Coinjoin.fr,
GingerWallet) maintain forks with similar parameters.

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
- **Legal risk**. Operating a coordinator carries unclear regulatory
  exposure (Samourai indictment April 2024). Users running their
  own coordinators should evaluate jurisdictional risk.

## References

- WabiSabi paper: https://github.com/zkSNACKs/WabiSabi/blob/master/Wabisabi.pdf
- Wasabi 2.0 docs: https://docs.wasabiwallet.io/
- KVAC primer: https://eprint.iacr.org/2013/516
