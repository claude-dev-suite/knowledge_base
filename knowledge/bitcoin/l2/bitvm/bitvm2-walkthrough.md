# BitVM2 Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/bitvm`.
> Canonical source: https://bitvm.org/bitvm2.pdf
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/bitvm/SKILL.md

## Concept

BitVM2 (Linus, Aumayr, Maffei 2024) is the second-generation BitVM
protocol: it makes optimistic computation verifiable on Bitcoin with
**permissionless challenge** (anyone can challenge, not just a fixed
verifier) and **single-transaction disprove** in the worst case. This
unlocks practical bridges and rollup verification on Bitcoin without
script changes.

## Walkthrough / mechanics

### Architecture

- **Prover (operator)**: claims a fact (e.g. "rollup state root is X").
- **Verifier (any watcher)**: challenges if claim is wrong.
- **Disprove transaction**: a single Bitcoin tx that, if broadcast,
  destroys operator's collateral and proves the lie.

### Pre-signing ceremony

At setup, prover and a permissionless set of verifiers run a multi-party
pre-signing ceremony to produce the **transaction graph** of all possible
challenge / disprove paths:

- Operator's claim tx.
- Each possible challenge tx.
- Each possible disprove tx (one per gate of the verifier circuit).

These pre-signed transactions are stored off-chain and published only
when needed.

### Bit commitments + Lamport one-time sigs

Operator commits to every wire of the verifier circuit via Lamport
signatures on the wire's value (0 or 1). Lamport signatures are simple:
two random 32-byte preimages per bit; sign 0 -> reveal preimage A,
sign 1 -> reveal preimage B. Reveal both = hash collision = anyone can
slash.

The full circuit is broken into ~10^9 gates for a STARK verifier; that's
~32 GB of pre-signed transactions but only KB of on-chain witness in the
happy path.

### Challenge / disprove

If verifier finds a wrong gate:

1. Verifier broadcasts the **challenge tx** for that gate.
2. Operator must respond with the gate's input bits + output bit.
3. Verifier provides the gate's expected output (computed off-chain).
4. Bitcoin script runs the gate's logic over operator-revealed bits and
   verifies the output. Mismatch -> disprove succeeds, operator's
   collateral goes to verifier.

### Optimistic happy path

If no challenger appears, operator's claim is accepted after the
challenge window. On-chain footprint: claim tx + take tx (~600 vB
total). Cost for honest operation is low.

### Single-tx disprove

Unlike BitVM1, which required on-chain bisection rounds, BitVM2's
disprove is a **single transaction** revealing the contradiction. This
dramatically reduces worst-case latency: once the verifier knows which
gate is wrong, they post one tx and the operator is slashed
immediately.

### Permissionless challenge

Anyone can become a verifier by:

1. Posting a small **challenger bond** (e.g. 0.05 BTC).
2. Identifying a wrong gate.
3. Broadcasting the challenge / disprove tx.

If correct, challenger gets a portion of operator's collateral. If
incorrect, challenger's bond is taken by operator.

## Worked example

Operator Op1 claims rollup state root S_1234 is correct.

```
1. Op1 broadcasts claim tx with Lamport commitments to ~10^9 wires
   (size: ~1 KB witness on Bitcoin via aggregation).
2. Challenger Bob reads operator's reveal, simulates verifier
   off-chain, finds wire #5 678 921 to be wrong.
3. Bob posts challenger bond.
4. Bob publishes challenge tx for wire #5 678 921. Op1 must respond.
5. Op1 reveals input bits to gate 5 678 921; Bob has expected output.
6. Bob broadcasts disprove tx: spends Op1's collateral via the
   gate-verification leaf with revealed inputs + Op1's claimed output.
7. Bitcoin script evaluates: gate(input1, input2) != Op1's output.
   Op1 collateral confiscated; Bob earns reward.
```

Honest case: no challenge over ~14 days, Op1 broadcasts take tx; total
on-chain cost ~few KB.

## Common pitfalls

- **Pre-signing ceremony scale**: signing ~10 GB of pre-built
  transactions requires careful coordination; participants must keep
  state.
- **Lamport key reuse**: signing the same bit value with the same
  Lamport key in two contexts leaks the bit. Each Lamport key MUST
  be used exactly once per claim instance.
- **Permissionless vs. controlled challenge**: original BitVM1 had a
  fixed verifier; BitVM2's permissionless challenge requires public
  challenger bond markets.
- **Witness fee market**: if Bitcoin congestion is high, broadcasting
  challenge txs may be expensive. Challengers must price bond + fee
  together.
- **Operator MEV opportunity**: between challenge tx broadcast and
  operator's response, operator may try to bribe miners to censor
  the challenge. Production protocols use timeouts protecting watchers.

## References

- Linus, Aumayr, Maffei. "BitVM2: Permissionless Verification on
  Bitcoin" arxiv 2024.
- bitvm.org spec + reference implementation.
- Lamport one-time signature original paper (1979).
