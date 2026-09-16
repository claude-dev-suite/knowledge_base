# Ephemeral Anchor Pattern - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/package-relay`.
> Canonical source: BIP433 and BIP431, related Bitcoin Core mempool policy
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/package-relay/SKILL.md

## Concept

"Ephemeral anchor" is the historical name for the **union of two
separate mechanisms** that Bitcoin Core shipped in different releases.
BIP433 records the split explicitly, and conflating them is the single
most common source of wrong code here:

- **Pay-to-Anchor (P2A)** - a keyless standard output type,
  `OP_1 <0x4e73>`. Standard *to spend* since Bitcoin Core 28.0
  (October 2024); specified as BIP433 (Status: Draft, Informational).
- **Ephemeral dust** - a mempool policy shipped in Bitcoin Core 29.0
  (April 2025) that allows a single dust output in a transaction
  *provided that transaction pays zero fee*, and requires that dust to
  be spent alongside any spend of the transaction's unconfirmed
  outputs.

Neither implies the other. Keyed anchors plus ephemeral dust is also an
accepted pattern, and a P2A output at or above its 240-sat dust limit
needs no ephemeral-dust waiver at all. Combined with TRUC v3, the pair
provides the non-pinnable fee-bump primitive designed for Lightning
commitment transactions.

The skill summarises both; this article walks the script, the dust-rule
waiver and its preconditions, and how this differs from classic keyed
anchor outputs (BOLT3's 330-sat `option_anchors`).

## Walkthrough / mechanics

**The P2A anchor output (BIP433):**

```
scriptPubKey = OP_1 <0x4e73>       # 2-byte witness v1 program
addresses    = bc1pfeessrawgf     (mainnet)
               tb1pfees9rn5nz     (public testnets)
               bcrt1pfeesnyr2tx   (regtest)
dust limit   = 240 sat             (Core default; other dust levels unchanged)
witness      = <>                  (empty - anything else is nonstandard)
```

The two program bytes spell "fees" in bech32m. It is a witness v1
program but it is **not** a taproot output - taproot programs are 32
bytes - which is a real parsing trap for address and script decoders.
Creating P2A outputs has been standard since segwit activated; what
28.0 changed is that *spending* them is standard.

**The ephemeral-dust waiver (Core 29.0, April 2025):**

Standard policy normally rejects outputs below the dust threshold for
their script type (330 sat for P2WSH, 240 sat for P2A). Ephemeral dust
waives that for a single output if:

1. The transaction creating the dust pays **zero fee**.
2. At most one dust output is present in that transaction.
3. Any transaction spending an unconfirmed output of that transaction
   also spends the dust output.

Note what is **not** required: the parent does not have to be TRUC v3,
the output does not have to be zero-valued, and the script does not have
to be `OP_TRUE`. Any output script type may go down to 0 value under
this rule. TRUC is a separate opt-in that BIP433 recommends alongside
P2A, because anyone can attach a child to a keyless anchor and an
oversized low-feerate child is a pinning vector.

**The must-spend-the-dust check:**

```
For each tx T being admitted:
  if T spends any unconfirmed output of parent P
     and P carries an ephemeral dust output D:
    require: T also spends D
```

Because P must be zero fee to carry the dust in the first place, in
practice P and its dust-spending child are submitted together as a
package.

**Why low/zero value?** A 240-sat P2A output is already the cheapest
standard anchor, and under ephemeral dust it can go lower (even 0) when
the parent pays no fee - so no channel capital is locked up as with
BOLT3's 330-sat anchors. The miner who confirms the package collects the
parent + child fees.

**Why keyless?** No signature or witness data is required, so any
watchtower or third party can fee-bump without privileged key material,
and the spending input's footprint is minimal. The trade-off is that
anyone can also grief you, which is what TRUC's topology limits are
for.

## Worked example

**Lightning commitment with ephemeral anchor:**

```
funding_tx (already confirmed):
  out0: 1_000_000 sat to 2-of-2 multisig

commitment_tx (v3, zero fee):
  in0:  funding_tx:0 (multisig spend with both sigs)
  out0: 500_000 sat to local balance
  out1: 499_760 sat to remote balance
  out2: 240 sat with scriptPubKey OP_1 <0x4e73>   # shared P2A anchor

anchor_cpfp (v3):
  in0:  commitment_tx:2 (the P2A anchor, empty witness)
  in1:  unrelated 50_000 sat utxo (signed by local)
  out0: 49_000 sat back to local                  # fee = 1_240 sat

submitpackage [commitment_tx, anchor_cpfp]
```

Mempool checks:

```
1. commitment_tx is v3 -> good.
2. anchor_cpfp is v3 -> good.
3. out2 is a standard P2A output at exactly the 240-sat dust limit ->
   no ephemeral-dust waiver needed. (Had it been below 240 sat, the
   waiver applies only because commitment_tx pays zero fee.)
4. anchor_cpfp spends commitment_tx:2 with an empty witness -> standard.
5. Package fee rate = (commitment fee + cpfp fee) / total vsize.
6. If rate >= mempool min, accept.
```

**Comparison to BOLT3 anchor (legacy):**

| Aspect | BOLT3 keyed anchor (`option_anchors`) | P2A shared anchor (`zero_fee_commitments`) |
|--------|---------------------------------------|--------------------------------------------|
| Value | 330 sat (P2WSH dust threshold) | 240 sat, or lower under ephemeral dust |
| Script | `<funding_pubkey> CHECKSIG IFDUP NOTIF <16> CSV ENDIF` | `OP_1 <0x4e73>` |
| Witness to spend | `<local_sig/remote_sig>`, or `<>` after 16 blocks | `<>` always |
| Count per commitment | Two (one per party) | One, shared |
| Lifecycle | Lives on-chain until spent | Same, unless created as ephemeral dust, in which case it must be spent alongside the parent |
| Dust burden | 330 sat per party per channel | 240 sat, or none when ephemeral |
| Pinning surface | Attacker pins via their own anchor's descendants | Anyone may attach a child; TRUC topology limits are the defence |

BOLT changes merged 2026-05-04 (lightning/bolts #1228) specify
`zero_fee_commitments` using the shared P2A output; BOLT 3 keeps the
keyed `option_anchors` outputs described above for existing channels.

**Replacement scenarios:**

To bump fees:

```
new_anchor_cpfp (v3):
  in0: commitment_tx:2 (anchor)
  in1: different utxo, higher fee
  out0: change

submitpackage [commitment_tx, new_anchor_cpfp]
```

The mempool sees commitment_tx already in mempool with old child;
new package replaces old child via TRUC sibling eviction (rule 7).

## Common bugs / pitfalls

1. **Forgetting the dust spend.** If the commitment carries an
   ephemeral-dust anchor, submitting it alone via `sendrawtransaction`
   fails - it is zero fee, and any spend of its unconfirmed outputs
   must also spend the dust. Use `submitpackage`. The two rejection
   reasons to look for are `dust` / "tx with dust output must be 0-fee"
   and `missing-ephemeral-spends`
   (`src/policy/ephemeral_policy.cpp`, Bitcoin Core 31.1, July 2026).
2. **Assuming ephemeral dust requires TRUC.** It does not: the
   precondition is a zero-fee parent with a single dust output. TRUC is
   a separate opt-in you take on for anti-pinning.
3. **Attaching witness data to the anchor spend.** Core only treats a
   P2A spend as standard when the witness is empty; padding it is
   nonstandard, not merely wasteful.
4. **Treating P2A as taproot.** `OP_1 <0x4e73>` is a witness v1 program
   with a 2-byte program, so taproot decoders that assume 32 bytes will
   mis-parse or reject it.
5. **Bumping later without spending the dust.** If the commitment is
   already in the mempool you can submit a fresh CPFP on its own, but
   that child must still spend the ephemeral-dust output; a child that
   spends only the `to_local` output while leaving the dust unspent is
   rejected.
6. **Fee rate accounting on the parent.** The commitment_tx fee is
   typically zero or negative (after subtracting the funding output's
   total minus channel balance distribution). Effective fee rate is
   only meaningful at package level.
7. **Two children spending same anchor.** Rule 3 of TRUC: max 1
   descendant. Cannot fan out two CPFPs from one anchor.

## References

- BIP433 (Pay to Anchor): https://github.com/bitcoin/bips/blob/master/bip-0433.mediawiki
- BIP431 (TRUC): https://github.com/bitcoin/bips/blob/master/bip-0431.mediawiki
- Bitcoin Core 28.0 release notes (P2A standard): https://bitcoincore.org/en/releases/28.0/
- Bitcoin Core 29.0 release notes (ephemeral dust): https://bitcoincore.org/en/releases/29.0/
- BOLT3 anchor outputs and `shared_anchor`: https://github.com/lightning/bolts/blob/master/03-transactions.md
- Zero-fee commitments: https://github.com/lightning/bolts/pull/1228
- Replacement cycling: https://bitcoinops.org/en/newsletters/2023/10/25/
