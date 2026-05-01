# TRUC v3 Replacement-Cycling Mitigation - Walkthrough Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/replacement-cycling`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0431.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/replacement-cycling/SKILL.md

## Concept

**TRUC** (Topologically Restricted Until Confirmation, BIP-431) introduces
a new mempool topology — version 3 (`nVersion=3`) transactions — designed
to be **non-cycleable**. It enforces strict packaging rules so that an
HTLC-success transaction's child can be reliably bumped via CPFP without
the parent being replaceable in ways that allow replacement-cycling
attacks.

Combined with **package relay** (BIP-329 family), TRUC v3 lets Lightning
nodes guarantee their HTLC claim transactions confirm in time, mitigating
the October 2023 attack class.

## Walkthrough / mechanics

### TRUC rules

A v3 transaction:

1. Must have only v3 ancestors (no mixing v2/v3 in mempool ancestry).
2. Has restricted topology: at most 1 unconfirmed v3 ancestor; at most
   1 unconfirmed v3 descendant.
3. Total package size <= 10 KvB.
4. Replacement (RBF) follows v3 rules: full RBF allowed regardless of
   `nSequence` (full-RBF policy is built-in for v3).
5. Replacement of a v3 ancestor requires also replacing its v3 descendant
   (cannot orphan the chain).

### Why this prevents cycling

The cycling attack depends on the attacker repeatedly replacing the
victim's package. Under TRUC:

- Victim's HTLC-success tx is v3.
- Victim's CPFP child (anchor output spend with high fee) is the v3
  descendant.
- Attacker tries to replace the parent: but TRUC requires they also
  replace the descendant atomically; their tx package must include the
  descendant.
- Attacker cannot construct a valid descendant unless they have the
  preimage / signing keys; they don't.

So the attacker's only attack path is to replace the parent + descendant
with a valid alternative — but they can't sign the victim's anchor
output. Cycling is broken.

### Migration path for Lightning

Channels using TRUC need:

1. New commitment tx version: v3.
2. New HTLC-success / HTLC-timeout tx version: v3.
3. Anchor outputs that only spend v3 children.
4. CPFP packaging: client always broadcasts a parent + child pair (using
   package relay, BIP-329).

This is a significant channel-format change; Lightning implementations
have been gradually rolling out support since 2024.

### Package relay

Mempool package relay (BIP-329 family) lets nodes broadcast a parent
+ child as a single package, evaluated for fee-bump eligibility against
the package as a whole. Without it, a low-fee parent might be
prematurely rejected.

### Full-RBF policy

TRUC mandates full-RBF for v3, removing the BIP-125 opt-in signaling
requirement. This simplifies the attack model: replacement is always
allowed if it follows TRUC rules.

## Worked example

Bob enforces an HTLC after force-close, with TRUC v3 channels:

```
Bob's commitment tx (v3):
  Output 0: Bob's main balance (1000 sat reserved)
  Output 1: HTLC output (100,000 sat)
  Output 2: anchor (Bob's, 330 sat)
  Output 3: anchor (Carol's, 330 sat)

Bob broadcasts:
  HTLC-success-tx (v3, ancestor of commitment):
    Input: Output 1 of commitment.
    Output: 100,000 sat - fee -> Bob.

Bob broadcasts (atomically as v3 package):
  CPFP-child-tx (v3, descendant):
    Input: anchor (Output 2).
    Output: 330 - extra_fee -> Bob (just bumps fee).

Total package: parent fee + CPFP fee, 2 txs.
```

If Carol attempts to replace the parent:

```
Carol crafts an HTLC-timeout-tx (v3) with higher fee.
TRUC rule: must replace BOTH parent and descendant atomically.
Carol cannot construct Bob's CPFP-child (lacks Bob's signing key).
Carol's replacement attempt fails -> cycling attack broken.
```

## Common pitfalls

- **Mixed-version channels**: until all peers + relays are TRUC-aware,
  some channels remain vulnerable. Migration is gradual.
- **Package relay deployment**: requires Bitcoin Core 27+ and updated
  mempool policies. Older nodes may reject TRUC packages.
- **Larger commitment tx size**: TRUC adds CPFP anchors for both peers,
  slightly increasing on-chain footprint per channel.
- **Ephemeral anchors** (BIP-431 part of v3) require both peers to use
  v3-compliant signing; misconfiguration breaks fee bumping.
- **Force-close still required**: TRUC mitigates ATTACK on force-close
  state but doesn't eliminate the need for force-close in adversarial
  scenarios.

## References

- BIP-431 — TRUC.
- BIP-329 — Package relay.
- "Mitigating Replacement Cycling Attacks" — Lightning-dev list, late 2023 / 2024.
- Bitcoin Core 27.0+ release notes (TRUC implementation).
