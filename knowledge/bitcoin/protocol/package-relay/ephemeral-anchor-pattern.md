# Ephemeral Anchor Pattern - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/package-relay`.
> Canonical source: BIP431, related Bitcoin Core mempool policy
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/package-relay/SKILL.md

## Concept

An ephemeral anchor is a zero-value output with a script that is
trivially spendable, which Bitcoin Core's mempool policy admits ONLY
if it's spent in the same package as the tx that creates it. The
anchor exists exclusively to enable CPFP fee bumping; it never lives
in a finalized chain state. Combined with TRUC v3, ephemeral anchors
provide a non-pinnable fee-bump primitive specifically designed for
Lightning commitment transactions.

The skill mentions ephemeral anchors briefly; this article walks the
script choices, the dust-rule waiver, the package-validation hook
that enforces "spent-in-same-package", and how this differs from
classic anchor outputs (BOLT3's 330-sat anchors).

## Walkthrough / mechanics

**Anchor output:**

```
scriptPubKey = OP_TRUE                       # 1 byte: 0x51
value        = 0
```

Standard policy normally rejects:
- Outputs below dust threshold (typically ~330 sat).
- Anyone-can-spend outputs without timelock or other anti-spam.

Ephemeral-anchor policy waives these IF AND ONLY IF:
1. The tx creating the anchor is v3 (TRUC).
2. The anchor is spent in the same package as creation.
3. The anchor's spending input has no witness data (just spends
   the OP_TRUE).

**The atomic-spend check:**

```
For each tx T in package:
  for each output O of T with value=0 and script=OP_TRUE:
    require: some other tx in package spends T:O.vout
```

If not satisfied, package admission fails with
"min-fee-not-met-anchor" or similar.

**Why zero-value?** Avoids the dust threshold debate entirely. Zero
sats can never be "dust"; it's just a UTXO that exists to be spent.
The miner who confirms the package gets the parent + child fees;
nobody loses 330 sat of "lock-up" cost as in BOLT3 anchors.

**Why OP_TRUE?** Anyone can spend it. The point isn't authentication
- it's that the owner of the package controls which child appears
in the package. Once mined together, the anchor is gone.

## Worked example

**Lightning commitment with ephemeral anchor:**

```
funding_tx (already confirmed):
  out0: 1_000_000 sat to 2-of-2 multisig

commitment_tx (v3):
  in0:  funding_tx:0 (multisig spend with both sigs)
  out0: 500_000 sat to local balance
  out1: 500_000 sat to remote balance
  out2: 0 sat with scriptPubKey OP_TRUE       # ephemeral anchor

anchor_cpfp (v3):
  in0:  commitment_tx:2 (the OP_TRUE anchor)
  in1:  unrelated 50_000 sat utxo (signed by local)
  out0: 49_000 sat back to local                # fee = 1_000 sat

submitpackage [commitment_tx, anchor_cpfp]
```

Mempool checks:

```
1. commitment_tx is v3 -> good.
2. anchor_cpfp is v3 -> good.
3. anchor output (out2) is OP_TRUE 0-value -> apply ephemeral rules.
4. anchor_cpfp spends commitment_tx:2 -> atomic-spend rule satisfied.
5. Package fee rate = (commitment fee + cpfp fee) / total vsize.
6. If rate >= mempool min, accept.
```

**Comparison to BOLT3 anchor (legacy):**

| Aspect | BOLT3 anchor | Ephemeral anchor |
|--------|--------------|------------------|
| Value | 330 sat (dust threshold) | 0 sat |
| Script | `<key> CHECKSIG IFDUP NOTIF <16> CSV ENDIF` | `OP_TRUE` |
| Lifecycle | Lives on-chain until spent | Must be spent in same package |
| Spendability without commitment | After 16-block delay, anyone | Cannot exist outside package |
| Dust burden | 330 sat per channel | None |
| Pinning surface | Two anchors (one each party); attacker spends theirs to pin | Single anchor; only package author controls |

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

1. **Forgetting atomic-spend.** Submitting commitment_tx alone via
   `sendrawtransaction` fails: ephemeral anchor unsupported outside
   package context.
2. **Two ephemeral anchors per commitment.** BOLT3 had two (one per
   party) for symmetric fee-bumping. Ephemeral anchors typically use
   ONE because both parties can spend OP_TRUE; signing one party's
   own commitment is implicit.
3. **Trying to add witness data to anchor spend.** OP_TRUE doesn't
   consume stack inputs; an anchor-spending input with witness data
   wastes fee. Some implementations refuse non-empty witness for
   anchor spends.
4. **Anchor not in package descendants.** If commitment is in mempool
   and you want to add a CPFP later, you must use `submitpackage`
   with `[anchor_cpfp]` only, BUT this works only if commitment is
   in mempool already and the anchor admission policy already
   accepted it. Standard sendrawtransaction on the cpfp alone
   FAILS because the parent's anchor was admitted under ephemeral
   policy that requires same-package spend.
5. **Fee rate accounting on the parent.** The commitment_tx fee is
   typically zero or negative (after subtracting the funding output's
   total minus channel balance distribution). Effective fee rate is
   only meaningful at package level.
6. **Two children spending same anchor.** Rule 3 of TRUC: max 1
   descendant. Cannot fan out two CPFPs from one anchor.

## References

- BIP431: https://github.com/bitcoin/bips/blob/master/bip-0431.mediawiki
- Bitcoin Core 28 release notes: https://bitcoincore.org/en/releases/28.0/
- BOLT3 anchor history: https://github.com/lightning/bolts/blob/master/03-transactions.md
- Replacement cycling: https://bitcoinops.org/en/newsletters/2023/10/25/
