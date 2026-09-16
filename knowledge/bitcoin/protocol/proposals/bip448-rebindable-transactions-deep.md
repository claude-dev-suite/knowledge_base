# BIP448 Taproot-native (Re)bindable Transactions - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/proposals`.
> Canonical source: BIP448 (bundling BIP446, BIP348, BIP349)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/proposals/SKILL.md

## Concept

BIP448 (Gregory Sanders, Antoine Poinsot, Steven Roose; assigned
2026-03-11; **Status: Draft**) is a *deployment bundle*, not an opcode.
It proposes activating three tapscript operations together, each with
its own specification BIP:

| BIP | Operation | Redefines | Assigned | Status |
|-----|-----------|-----------|----------|--------|
| 446 | `OP_TEMPLATEHASH` | `OP_SUCCESS206` (0xce) | 2026-02-06 | Draft |
| 348 | `OP_CHECKSIGFROMSTACK` | `OP_SUCCESS204` (0xcc) | 2024-11-26 | Draft |
| 349 | `OP_INTERNALKEY` | `OP_SUCCESS203` (0xcb) | 2024-11-14 | Draft |

BIP446 and BIP448 are by Sanders, Poinsot and Roose; BIP348 and BIP349
are by Brandon Black and Jeremy Rubin.

The bundle's stated purpose is **rebindable transaction signatures**.
BIP448's motivation: together the three "enable rebindable transaction
signatures, making possible a new type of payment channel: LN-Symmetry
('Eltoo')". This matters for the corpus because it displaces
`SIGHASH_ANYPREVOUT` (BIP118) as the mainstream route to eltoo -
see `apo-bip118-deep.md`, which now carries that framing.

Because all three redefine existing `OP_SUCCESS` opcodes, none of them
needs a new sighash type, a new key version, or a new leaf version.
Pre-activation, an `OP_SUCCESS` in an executed tapscript makes the
script succeed unconditionally, so these outputs are anyone-can-spend
until a soft fork lands.

## Walkthrough / mechanics

**`OP_TEMPLATEHASH` (BIP446).**

Redefines `OP_SUCCESS206` in the tapscript execution context. On
execution it **pushes** the template hash of the transaction in
context onto the stack.

That push-rather-than-verify shape is the key difference from
`OP_CHECKTEMPLATEVERIFY` (BIP119), which pops a 32-byte commitment and
compares. Pushing composes: the hash can be fed to
`OP_CHECKSIGFROMSTACK`, hashed with other data, or compared by the
script itself. CTV's compare-and-fail shape cannot be composed that
way.

BIP446's motivation lists what commitment-to-the-spending-transaction
buys: removing the need to share HTLC signatures in Lightning's
`commitment_signed`, making receipt of an Ark VTXO non-interactive,
reducing roundtrips in LN-Symmetry, and a "significant optimisation"
for discreet log contracts.

**`OP_CHECKSIGFROMSTACK` (BIP348).**

Redefines `OP_SUCCESS204` for taproot script spends with leaf version
0xc0. Semantics, from the BIP:

```
pops 3 elements:  <pubkey (32 bytes)>  <message>  <signature>

if the signature is valid for that pubkey over that message
    push 1
else if the signature is the empty vector
    push 0
else
    script execution fails
```

Only 32-byte keys are constrained. As with BIP342 unknown key types,
other key lengths perform no verification and are considered
successful - the standard upgrade hook.

CSFS is what makes signatures over arbitrary messages possible:
delegation and oracle attestations are the direct applications. It has
been deployed in Blockstream's Elements for years.

**`OP_INTERNALKEY` (BIP349).**

Redefines `OP_SUCCESS203`. Pushes the 32-byte x-only taproot internal
key (BIP341's *p*) to the stack. BIP349's motivation gives the
concrete win: when parties want to sign with an aggregate internal key
but only under extra script restrictions, `OP_INTERNALKEY` saves 8
vBytes over pushing the key literally.

**How the three combine into rebindability.**

`OP_TEMPLATEHASH` produces a digest of the spending transaction;
`OP_CHECKSIGFROMSTACK` checks a BIP340 signature over a message taken
from the stack rather than over the BIP341 sighash. A signature can
therefore be checked against a message the script constructs, instead
of one consensus computes from a fixed prevout - which is exactly the
property BIP118 obtains by deleting the prevout from the sighash
digest. `OP_INTERNALKEY` supplies the key cheaply.

## Worked example

The BIP448 motivation section enumerates what the bundle is claimed to
enable. Quoting and unpacking it, because this list is the actual
argument for the proposal:

```
LN-Symmetry ("Eltoo")     a new type of payment channel; "its
                          simplicity makes advanced constructs like
                          multiparty channels practical"
Daric                     simplifications for 2-party channels
Statechains               "can substantially improve statechains"
Ark variant "Erk"         further interactivity reduction
PTLC upgrade              "dramatic simplification" of moving today's
                          Lightning to Point Time Locked Contracts
```

**Deployment.** BIP448's Deployment section is one sentence: "The
specific activation is left to be determined at a later date." There
is no version bit, no start time and no threshold. The BIP notes that
a CTV+CSFS-flavoured opcode set was previously proposed for activation
under the name "LNHANCE".

**Ecosystem as of September 2026** (Optech, 2026-09-04):

- A **BIP448 GitHub organization** aggregates implementations: Bitcoin
  Inquisition, a Bitcoin Core patch without activation, miniscript and
  PSBT integration, draft LN-Symmetry BOLTs with a Core Lightning
  implementation, and an Ark `OP_TEMPLATEHASH` signet demo. The
  organization says the full bundle will be usable on the default
  signet with the next Bitcoin Inquisition release.
- **covenants.diy** is a browser editor that builds taproot outputs and
  steps through tapscript under selectable opcode sets, with
  permalinked examples including BIP448 rebindable state, BIP119
  vaults and congestion control, and BIP348 delegation.
- A **Covenants Use-Case Atlas** collects more than two dozen
  constructions - vaults, congestion control, Ark issuance,
  LN-Symmetry.

A concrete construction from the same newsletter shows how CSFS gets
used in practice: an Ark out-of-round VTXO **equivocation bond**,
slashable by publishing two CSFS-validated signatures from the
assignment key over distinct BIP341 sighashes. The bond and the
preallocated transaction tree need a next-transaction covenant, which
"can be either `OP_CHECKTEMPLATEVERIFY` (CTV) or `OP_TEMPLATEHASH`" -
a fair statement of how interchangeable the two are for that role.

## Common bugs / pitfalls

1. **Treating BIP448 as an opcode.** It is a bundle. The opcode
   semantics live in BIP446, BIP348 and BIP349, and those can in
   principle be activated separately.
2. **Pre-activation outputs are anyone-can-spend.** All three
   operations redefine `OP_SUCCESS` opcodes. Today, a tapscript
   containing any of them succeeds unconditionally when executed.
   Never fund such an output on mainnet.
3. **Assuming `OP_TEMPLATEHASH` is CTV with a new number.** CTV pops a
   commitment and verifies; `OP_TEMPLATEHASH` pushes the hash. The
   push form composes with other opcodes, which is the whole point.
4. **Assuming eltoo requires BIP118.** As of September 2026 the
   LN-Symmetry implementations in the wild target BIP448. BIP118 is
   still Draft and has not been withdrawn, but it is not the active
   route.
5. **Reading CSFS's empty-signature case as a failure.** An empty
   vector pushes 0 and continues; any other invalid signature fails
   the script. Scripts that branch on the result must handle the
   0 case deliberately.
6. **Forgetting the non-32-byte key upgrade hook.** `OP_CSFS` performs
   no verification and reports success for pubkeys that are not 32
   bytes. A script that accepts caller-supplied keys without
   constraining their length has no signature check at all.
7. **Citing an activation date.** There is none. Neither BIP448 nor
   any of its three component BIPs has activation parameters.

## References

- BIP448: https://github.com/bitcoin/bips/blob/master/bip-0448.md
- BIP446 OP_TEMPLATEHASH: https://github.com/bitcoin/bips/blob/master/bip-0446.md
- BIP348 OP_CHECKSIGFROMSTACK: https://github.com/bitcoin/bips/blob/master/bip-0348.md
- BIP349 OP_INTERNALKEY: https://github.com/bitcoin/bips/blob/master/bip-0349.md
- BIPs index (statuses): https://github.com/bitcoin/bips/blob/master/README.mediawiki
- Optech newsletter #421 (2026-09-04), BIP448 demos and applications: https://bitcoinops.org/en/newsletters/2026/09/04/
- Optech eltoo/LN-Symmetry topic: https://bitcoinops.org/en/topics/eltoo/
- APO, the historical route: `apo-bip118-deep.md`
