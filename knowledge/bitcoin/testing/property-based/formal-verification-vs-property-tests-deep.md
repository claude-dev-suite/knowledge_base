# Formal Verification vs Property Tests - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/testing/property-based`.
> Canonical source: https://github.com/ProofOfKeags/btc-verified and
> https://delvingbitcoin.org/t/btc-verified-formalizing-the-bitcoin-protocol/2684
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/testing/property-based/SKILL.md

## Concept

A property test and a theorem state the same shape of claim - "for all
x, P(x)" - and differ only in how the "for all" is discharged. proptest
and hypothesis discharge it by sampling: a few thousand draws from a
strategy, plus shrinking to make the counterexample readable. A proof
assistant discharges it by construction, over the whole domain, with a
kernel re-checking the argument on every build.

The gap matters most exactly where Bitcoin lives. Consensus code must
have every node agree on every byte; a divergence is a chain split; and
the inputs that cause divergence are the ones nobody wrote a test for.
Sampling is a poor fit for a space that adversarial.

Three tiers are worth distinguishing, cheapest first:

1. Property tests - random draws against an invariant. Cheap, catches
   round-trip and parser bugs, never exhaustive.
2. Declarative executable specifications - the rules written once in a
   form that both runs and reads as a spec, then differentially tested
   against implementations. Stronger as documentation and as a
   differential oracle; still not proved.
3. Machine-checked proofs - the invariant as a theorem in a proof
   assistant, with an explicit, auditable list of assumptions.

None replaces the one below it. A proof is about a model; a property
test is about the code you actually ship.

## Walkthrough / mechanics

### btc-verified (Lean 4)

`btc-verified` (Keagan McClelland, Apache-2.0) is the Bitcoin-specific
instance of tier 3. It was announced on Delving Bitcoin on 2026-07-03
and covered by Optech on 2026-07-17; the repository pins Lean
`v4.30.0-rc2` as of September 2026.

Its structure is worth copying even if you never write Lean. The repo
grows as small "proof leaves": each module builds cleanly, states
exactly what it proves in its own README, and makes the next claim
easier to state. Two mechanisms keep the claims honest:

- Golden vectors at build time. The verified decoders and the computable
  SHA-256 are ordinary reducible definitions, not opaque ones, so a
  digest *computes*. `lake build` runs them over real mainnet bytes -
  the first Bitcoin payment, the genesis block, block 170, the SegWit
  activation coinbase, the first SegWit spend - and `lake test` decodes
  the full SegWit activation block (481824).
- An axiom audit. The build fails if any headline theorem depends on
  `sorry` or an unexpected axiom. This is the formal-methods analogue of
  committing `proptest-regressions/`: it stops the guarantee from
  quietly eroding.

Proof leaves in place as of September 2026, per the repository's own
READMEs: serialization (decoding inverts encoding, and each value has
exactly one accepted encoding, CompactSize included), a computable
FIPS 180-4 SHA-256 checked against published vectors, Bitcoin's merkle
tree, transactions and binding txids, block codecs plus the coinbase
witness commitment and chain-tip commitment, Script as the raw bytes
consensus validates, the chainstate and its accounting identity,
transcriptions of Bitcoin Core functions under `Impl/`, a fast packed
serializer proved equal to the spec one, and an abstract BitVM bit
commitment whose equivocation yields a hash collision.

Announced future work is tracked as GitHub milestones: consensus guards,
the 21-million supply-limit theorem, proof-of-work and cumulative work,
block tree and fork choice, the Core transcription track, a verified
Script interpreter, and a full-chain demo. None of those are proved yet;
do not cite them as results.

### Hornet Node's declarative specification (tier 2)

Toby Sharp reported (Optech #402, 2026-04-24, via Delving Bitcoin and
the bitcoin-dev list) completing a declarative specification of
non-script block validation rules for Hornet Node: 34 semantic
invariants composed using a simple algebra, with stated future work
extending it to script validation and comparing against implementations
such as libbitcoin. This is the cheapest step up from property tests -
the invariants become a single artifact you can both run and read,
usable as a differential oracle against Core.

### Verified implementations of crypto primitives

`libshrincs` is a proof-of-concept (as of September 2026) for the
SHRINCS post-quantum signature draft: one machine-checked theorem
connecting a C implementation of the WOTS+C leaf one-time signature, via
Rocq/VST against CompCert's Clight semantics, to a game-based
one-time-unforgeability bound expressed in SSProve. Its trust surface is
published explicitly - six real-valued advantage-bound parameters and
six axioms asserting truncated SHA-256 meets them - and an audit script
fails the build if the proved statement drifts from a hand-pasted copy
of it. Note the honesty caveat its own README makes: SSProve has no
resource model, so a hardness assumption about a fixed function is
formally weak however it is phrased.

## Worked example

The merkle-root mutation check (CVE-2012-2459) is the clearest
comparison, because the corpus already describes it informally in
`protocol/consensus/block-validation-flow.md`.

As a property test, you would generate leaf lists, compute Core's
root-and-`mutated` pair, and assert that two accepted lists with equal
roots are equal. Neither of the interesting lists is a counterexample
to that property: `[1, 2, 3, 3]`, the textbook attack, is rejected by
the `mutated` scan, and `[1, 2, 2]` - the scan-before-pad
discriminator, where Core scans each level for an adjacent duplicate
*before* it pads, so a duplicate that padding itself synthesized is
never compared to its twin - is accepted and canonical. Both are
discriminating cases for the behavior rather than failing cases, and
both have to be constructed by hand; btc-verified runs them as
concrete build-time vectors. A strategy that draws leaves as
independent 32-byte hashes synthesizes duplicate or padding-shaped
leaf lists with negligible probability, so it exercises neither.

As a proof, btc-verified factors the problem apart. `Crypto/` defines
the merkle root as a structural tree fold, with procedural duplication
an explicit `pad` constructor rather than an artifact of iteration
order, and makes canonicality a first-class decidable property of the
leaf list. `Impl/BitcoinCore` then transcribes Core's
`ComputeMerkleRoot` from `src/consensus/merkle.cpp` - the fused variant
that also computes the `mutated` flag - rendering its `while` loop as a
recursion and its sticky `bool mutation` as the OR of each level's scan.
The checked claims, by name:

| Theorem | What it establishes |
|---------|---------------------|
| `computeRoot_eq_root` | the bottom-up vector fold computes the spec's tree root |
| `canonical_of_not_mutated` | a leaf list Core accepts (`mutated = false`) is canonical - unconditionally, no distinctness or collision caveat |
| `eq_of_computeMerkleRoot_eq_of_not_mutated` | two nonempty equal-width lists Core accepts with equal roots are equal, or a concrete double-SHA-256 collision exists |
| `computeMerkleRoot_fst` | the root Core returns is exactly the spec root on every input |

Two details that a prose description of CVE-2012-2459 usually loses and
the proof makes explicit:

- The converse of `canonical_of_not_mutated` is false: `[a, a]` is
  canonical yet sets `mutated`. Core's scan is strictly stronger than
  canonicality requires, which is why no transaction-distinctness
  hypothesis is needed anywhere in the argument.
- Collision resistance is deliberately *not* asserted of the concrete
  SHA-256. For a fixed function it is provably false by pigeonhole, so
  it lives as a hypothesis over an abstract hash wherever a soundness
  proof needs it, and the binding statements come out as "the lists are
  equal, or here are two colliding byte strings".

Reproducing the build:

```bash
git clone https://github.com/ProofOfKeags/btc-verified
cd btc-verified
lake exe cache get   # mathlib cache, first build only
lake build           # library + golden vectors + axiom audit
lake test            # block 481824 through the block codec
```

## Common pitfalls

- Treating a proof as a statement about Bitcoin Core. The author is
  explicit that the repository "is structurally incapable of giving
  guarantees about Bitcoin Core's behavior". `Impl/` transcribes Core
  functions; a transcription checked against a spec is evidence about an
  algorithm, not about the compiled binary.
- Citing the roadmap as a result. The supply-limit and fork-choice
  theorems are milestones, not proofs, as of September 2026.
- Assuming the model is the protocol. btc-verified's `Consensus/` is a
  reasoning substrate heavily informed by Core without being
  definitionally identical to it; the intended bridge to `Impl/` is a
  refinement theorem over accepted block histories, which is future
  work.
- Dropping the property tests. The proof covers the model. The property
  tests are what keep the shipped Rust or C++ honest about matching it,
  and stay the cheapest way to catch a serializer regression.
- Ignoring the trust surface. Both btc-verified and libshrincs publish an
  allowlist of axioms and fail the build when the cone of assumptions
  grows. A verification effort without that gate can lose its guarantee
  to a single `sorry` / `Admitted` introduced in a refactor.
- Over-reading AI-assisted proofs. btc-verified's author flags heavy AI
  use in the repository and limited proof readability; the kernel still
  checks the proof, but the *statement* of each theorem is what a human
  must review, and a statement can be weaker than it looks.

## References

- btc-verified: `https://github.com/ProofOfKeags/btc-verified`
- Delving Bitcoin announcement (2026-07-03):
  `https://delvingbitcoin.org/t/btc-verified-formalizing-the-bitcoin-protocol/2684`
- Optech newsletter, 2026-07-17:
  `https://bitcoinops.org/en/newsletters/2026/07/17/`
- Optech #402 (2026-04-24), Hornet Node declarative specification:
  `https://bitcoinops.org/en/newsletters/2026/04/24/`
- libshrincs: `https://github.com/remix7531/libshrincs`
- Lean 4: `https://github.com/leanprover/lean4`
