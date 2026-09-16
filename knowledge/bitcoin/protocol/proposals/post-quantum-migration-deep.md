# Post-Quantum Migration at the Protocol Layer - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/proposals`.
> Canonical source: BIP360, BIP361
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/proposals/SKILL.md

## Concept

Post-quantum work is, as of September 2026, a dominant consensus-track
discussion. The corpus's older framing - "quantum-resistant signature
schemes (Lamport / Winternitz one-time)" as a use case for OP_CAT -
is not wrong, but it is one line about one dimension. The live work
has four distinct dimensions, and conflating them is the usual
mistake:

1. **Output types** - what a quantum-resistant scriptPubKey looks like
   (BIP360 P2MR, the P2TRv2 discussion).
2. **Migration policy** - how and whether to force coins out of
   quantum-vulnerable outputs (BIP361).
3. **Signature schemes** - what actually signs (SHRINCS and the
   NIST-ratified families). Covered by the cryptography domain.
4. **Rescue mechanics** - what happens to coins nobody moved before
   Q-day (DropKick, Lifeboat).

Plus transport: a post-quantum path for BIP324 P2P encryption.

None of this is activated. None of it has activation parameters. The
Optech quantum-resistance topic page still frames the baseline
position: quantum-resistant alternatives to ECDSA exist but "involve
much larger key and signature sizes, so most developers seem to prefer
to delay upgrading until it's necessary".

## Walkthrough / mechanics

**Output types.**

`BIP360 "Pay-to-Merkle-Root (P2MR)"` (Hunter Beast, Ethan Heilman,
Isabel Foxen Duke; assigned 2024-12-18; Draft, v0.12.1) proposes an
output type that behaves like P2TR "but with the key path spend
removed". With no key path there is no EC point in the output, so the
output resists **long exposure attacks** - attacks on keys exposed for
longer than it takes to confirm a spending transaction.

The BIP is explicit about what it does *not* do: P2MR gives no
protection against **short exposure attacks**, meaning private-key
recovery from a public key sitting in the mempool. That needs
post-quantum signatures, which BIP360 defers to a separate proposal.
Note the title change - BIP360 was previously the
pay-to-quantum-resistant-hash (P2QRH) proposal, so older references to
"BIP360 P2QRH" are describing an earlier revision of the same number.

`P2TRv2` is the other candidate output type. It is a mailing-list and
Delving Bitcoin discussion, not a BIP. Pieter Wuille's position
(Optech, 2026-09-04) is P2TRv2 as the default for casual users, with
P2MR for more sophisticated users who want to hide EC points.

**The CISA bundling question.**

The live design argument is whether cross-input signature aggregation
should ship *with* the post-quantum output type:

- **Conduition (for bundling).** Pairing CISA with P2TRv2 would
  strongly incentivise migration.
- **Wuille (against).** Max weight reduction is about 28%, and only
  for many-input transactions; wallet and custodian support is the
  real bottleneck; CISA adds specification and implementation
  complexity that may delay a P2TRv2 soft fork; entities might
  postpone all PQC work to ship both together.
- **Conduition (rebuttal).** A CISA-supporting output type can be
  adopted first with ordinary BIP340 signatures, aggregation added
  later.
- **Adam Gibson.** Agreed with Wuille: bundling is a poor fit for
  P2TRv2's adoption-first goal.

Wuille added two structural cautions: post-Q-day hash-based signatures
likely need a new witness costing rule that weighs CPU more and
serialized size less; and having third-party relay nodes incrementally
aggregate signatures would hide the real bandwidth cost inside the
consensus layer and could entrench existing mining pools by
incentivising direct submission to miners.

**Migration policy.**

`BIP361 "Post Quantum Migration and Legacy Signature Sunset"` (Jameson
Lopp, Christian Papathanasiou, Ian Smith, Joe Ross, Steve Vaile,
Pierre-Luc Dallaire-Demers; assigned 2026-02-11; published via BIPs
#1895 on 2026-04-24) is **Informational**, not a Specification BIP,
and states it "follows the implementation of any post-quantum (PQ)
output type". Two phases:

```
Phase A   Disallows sending any funds to quantum-vulnerable
          addresses, hastening adoption of PQ address types.

Phase B   Restricts ECDSA/Schnorr spends by encumbering them with a
          quantum-safe rescue protocol, preventing theft of funds in
          quantum-vulnerable UTXOs. Triggered by a well-publicized
          flag day five years after activation.
```

Its framing: "It turns quantum security into a private incentive: fail
to upgrade and you will encounter additional friction to access your
funds." The BIP cites NIST's 2024 ratification of three production-grade
PQ signature schemes and academic roadmaps estimating a
cryptographically-relevant quantum computer "as early as 2027-2030".

**Signature schemes: SHRINCS.**

A semi-stateful hash-based scheme, with a first draft BIP posted
2026-09-04 by Conduition on behalf of the SHRINCS working group.
Parameters as specified in that draft:

```
public keys              48 bytes
stateful signatures      548 bytes at the smallest
stateless fallback       5,777 bytes
stateless budget         2^40 signatures (raised so high-frequency
                         protocols such as LN can use the fallback)
verification             4x-16x faster per byte than BIP340 schnorr
                         with SHA256 hardware acceleration; at worst
                         2,792 SHA256 compressions for a stateless
                         signature
```

Changes since the original proposal: black-box SLH-DSA (FIPS-205)
compatibility, flexible XMSS trees of any structure, and faster
(larger) stateful parameters. The draft specifies only a signature
scheme - deploying it via new opcodes or a new output type would be a
separate proposal. `libshrincs`, a C library from Jonas Nick and
remix7531 with machine-checked WOTS+C proofs, was released alongside.

Two caveats from the same discussion: **reusing a stateful counter
lets an observer forge signatures**, and Antoine Riard noted that
5,777-byte stateless signatures would be roughly 90x the onchain cost
of today's transactions unless those fields are discounted.

**Rescue mechanics: DropKick.**

Posted by Conduition to the Bitcoin-Dev list on 2026-09-04. A
commit/reveal protocol for users who have not moved coins to PQC
outputs by Q-day:

```
1. Commit   Hide a commitment to your post-quantum public key and an
            ownership witness (proof of knowledge asymmetry) somewhere
            in a block - e.g. an OP_RETURN or a taproot tweak.
            Users without a PQ-safe UTXO of their own can hand
            commitments to untrusted aggregators, who merkle-commit
            many users' commitments under one onchain root, optionally
            for a fee paid from the rescued coins.

2. Wait     A delay, ~100 blocks in the author's sketch.

3. Reveal   Publish the proof, a signature from the post-quantum
            public key, and an SPV-style opening proof that the
            commitment appeared in an earlier block.
```

The soft fork is **non-confiscatory only if it encumbers exactly the
UTXOs with decidable knowledge asymmetries** - those where a validator
can tell from the output alone that hidden data such as a hashed
pubkey exists. Covering undecidable cases such as BIP32 key derivation
would rescue more coins but could confiscate some. P2PK coins cannot
be covered at all.

Compared with Tadge Dryja's Lifeboat, which requires each user to
already hold a PQ-secure UTXO to post a commitment, DropKick drops the
requirement to index and order every onchain commitment, at the cost
of miner-censorship risk on the reveal. Conduition argues a long delay
plus a value-proportional fee (about 1% of the UTXO to honest miners)
makes censorship unprofitable, assuming censors will not reorg out
blocks that undermine the attempt.

**Adjacent proposals.**

- A segwit commitment to post-quantum witness data (Optech,
  2026-08-07).
- A post-quantum path for BIP324 v2 P2P transport encryption (Optech,
  2026-06-05). Relevant because the Optech topic page notes Lightning's
  use of Noise appears to depend on ECDSA security, so recorded LN
  traffic may be decryptable after Q-day.

## Common bugs / pitfalls

1. **Collapsing the four dimensions.** "Bitcoin's quantum proposal" is
   not one thing. BIP360 is an output type, BIP361 is a migration
   policy, SHRINCS is a signature scheme, DropKick is a rescue
   protocol. They have different authors, statuses and preconditions.
2. **Citing "BIP360 P2QRH".** BIP360 is now titled Pay-to-Merkle-Root
   (P2MR). Older material describing it as pay-to-quantum-resistant-
   hash is describing an earlier revision.
3. **Presenting BIP361 as a consensus specification.** Its Type is
   **Informational** and it explicitly presupposes some other PQ
   output type BIP that does not exist yet ("Requires: TBD Post
   Quantum Signature BIP").
4. **Treating P2MR as full quantum resistance.** It defends against
   long exposure only. A P2MR spend still reveals whatever the
   tapscript reveals, and short-exposure mempool attacks are out of
   scope by the BIP's own statement.
5. **Quoting SHRINCS sizes without the stateful/stateless split.** 548
   bytes and 5,777 bytes are two different regimes with two different
   safety properties, and counter reuse in the stateful regime is
   catastrophic.
6. **Assuming a rescue soft fork is automatically non-confiscatory.**
   DropKick is non-confiscatory only over decidable knowledge
   asymmetries. Extending coverage trades rescued coins against
   confiscated ones, and P2PK is unreachable either way.
7. **Attaching a date.** Nothing here has activation parameters. The
   "2027-2030" figure in BIP361 is a cited estimate of when a
   cryptographically-relevant quantum computer might exist, not a
   Bitcoin schedule.

## References

- BIP360 Pay-to-Merkle-Root: https://github.com/bitcoin/bips/blob/master/bip-0360.mediawiki
- BIP361 Post Quantum Migration and Legacy Signature Sunset: https://github.com/bitcoin/bips/blob/master/bip-0361.mediawiki
- Optech quantum resistance topic: https://bitcoinops.org/en/topics/quantum-resistance/
- Optech newsletter #421 (2026-09-04) - PQC output types, DropKick, SHRINCS draft BIP: https://bitcoinops.org/en/newsletters/2026/09/04/
- Optech newsletter (2026-08-07) - segwit commitment to post-quantum witness data: https://bitcoinops.org/en/newsletters/2026/08/07/
- Optech newsletter (2026-06-05) - a post-quantum path for BIP324: https://bitcoinops.org/en/newsletters/2026/06/05/
