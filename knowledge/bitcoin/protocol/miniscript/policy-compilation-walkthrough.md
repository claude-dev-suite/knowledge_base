# Policy to Miniscript Compilation - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/miniscript`.
> Canonical source: https://bitcoin.sipa.be/miniscript/, rust-miniscript docs
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/miniscript/SKILL.md

## Concept

The Miniscript compiler turns a high-level policy (declarative spending
conditions with optional probability annotations) into the cheapest
miniscript that implements it. The compiler is non-trivial because
miniscript fragments have different cost characteristics depending on
type (B/V/K/W) and modifiers (z/o/n/d/u). The skill mentions the
compiler exists; this article walks the algorithm and shows the cost
matrix it minimizes against.

## Walkthrough / mechanics

The compiler operates as dynamic programming over policy subexpressions:

```
compile(policy) -> { type -> cheapest_miniscript_of_that_type }

For each fragment of each subpolicy:
  for each (modifier set, type) reachable:
    cost = static_size + p_satisfy * sat_size + p_dissatisfy * dsat_size
    keep cheapest entry per (type, modifier_set) tuple
```

**Cost components per fragment:**

| Component | Definition |
|-----------|------------|
| static_size | Size of script bytes (ignoring witness). |
| sat_size | Witness bytes when path is taken (signatures, preimages). |
| dsat_size | Witness bytes when path is NOT taken (still must satisfy boolean). Only meaningful for `d`-modifier (dissatisfiable) fragments. |
| sigops | Static + dynamic signature-op cost. |
| has_free_dsat | Dissatisfaction is free (zero witness). |

**Probability annotations:**

```
or(99@A, 1@B)     # A is taken 99% of the time; weight cost accordingly
or(A, B)          # equal probabilities by default
```

Annotations propagate through nested ORs to weight branch ordering.

The compiler chooses among alternative shapes for each policy:
- `or(A, B)` can compile to `or_b(A, W:B)`, `or_d(A, B)`, `or_c(A, V:B)`,
  `or_i(A, B)` -- each with different witness cost depending on
  satisfaction probabilities.
- `and(A, B)` -> `and_v(V:A, B)` or `and_b(A, W:B)`.

## Worked example

Policy: hot key OR (cold key AND 1-week timelock).

```
policy = or(99@pk(Hot), and(pk(Cold), older(1008)))
```

**Step 1: enumerate sub-policies.**

```
S1 = pk(Hot)                                  # type B, dissatisfiable
S2 = pk(Cold)                                 # type B, dissatisfiable
S3 = older(1008)                              # type B, NOT dissatisfiable
S4 = and(S2, S3)                              # type B, NOT dissatisfiable
S5 = or(99@S1, S4)                            # type B, NOT dissatisfiable (under standard or_d)
```

**Step 2: enumerate candidates for each.**

For `S4 = and(pk(Cold), older(1008))`:

| Candidate | Script | sat (likely=1%) |
|-----------|--------|-----------------|
| `and_v(v:pk(Cold), older(1008))` | `<Cold> CHECKSIGVERIFY <1008> CSV` | 73 (sig) + 0 = 73 |
| `and_b(pk(Cold), s:older(1008))` | `<Cold> CHECKSIG SWAP <1008> CSV BOOLAND` | 73 + 1 = 74 |

`and_v` wins (smaller script, same sat).

For `S5`:

| Candidate | Selected? |
|-----------|-----------|
| `or_d(pk(Hot), and_v(v:pk(Cold), older(1008)))` | yes -- best for 99/1 weight |
| `or_b(pk(Hot), s:and_b(pk(Cold), s:older(1008)))` | no -- needs dissatisfaction of cold branch |
| `or_i(pk(Hot), and_v(v:pk(Cold), older(1008)))` | no -- always pays 1-byte branch selector |

**Step 3: emit Script.**

```
or_d(pk(Hot), and_v(v:pk(Cold), older(1008)))

translates to:

<Hot> CHECKSIG IFDUP NOTIF <Cold> CHECKSIGVERIFY <1008> CSV ENDIF
```

Witness analysis:

```
hot path  (99%): [<sig_Hot>]                      # 73 wu
cold path (1%):  [<sig_Cold>, <empty>]            # 73 + 1 = 74 wu (empty dissatisfies pk(Hot))
                                                  # then NOTIF takes the cold branch
```

Expected witness: `0.99 * 73 + 0.01 * 74 = 73.01` weight units.

**Using `rust-miniscript`:**

```rust
use miniscript::policy::Concrete;
use miniscript::Miniscript;
use miniscript::Segwitv0;

fn main() {
    let policy: Concrete<bitcoin::PublicKey> =
        "or(99@pk(Hot),and(pk(Cold),older(1008)))".parse().unwrap();
    let ms: Miniscript<_, Segwitv0> = policy.compile().unwrap();
    println!("{}", ms);
    // or_d(pk(Hot),and_v(v:pk(Cold),older(1008)))
    println!("{}", ms.encode());            // raw Script bytes
}
```

## Common bugs / pitfalls

1. **Forgetting probability annotations.** Without `99@`/`1@`, the
   compiler assumes 50/50 and may pick a suboptimal shape (e.g.,
   `or_i` adds 1 byte for branch selection, costly when one branch
   dominates).
2. **Compiling for the wrong context.** `compile::<Segwitv0>()` and
   `compile::<Tap>()` produce different miniscripts because Tapscript
   uses `multi_a` instead of `multi`, and 32-byte x-only keys vs
   33-byte. Mixing them yields scripts that fail consensus.
3. **Policy that exceeds resource limits.** A `thresh(20, ...20 keys)`
   compiles fine but exceeds P2WSH's 3600-byte witness limit. The
   compiler returns an error; you must restructure (e.g., move to
   Taproot script paths).
4. **Asymmetric ORs that should be ANDs.** `or(pk(A), pk(B))` allows
   A or B alone; `and(pk(A), pk(B))` requires both. Misusing the
   policy verbs is the most common semantic bug.
5. **Hash preimages without `older` lock.** A policy like
   `or(pk(A), sha256(H))` lets anyone reveal H to spend. Fine if H
   is bound to an HTLC, dangerous otherwise.
6. **Reusing fragments across BRANCHes hoping for sharing.** The
   compiler emits each leaf independently; no common subexpression
   elimination at Script level.

## References

- Miniscript site: https://bitcoin.sipa.be/miniscript/
- rust-miniscript: https://docs.rs/miniscript/latest/miniscript/
- Pieter Wuille talk "Miniscript": https://www.youtube.com/watch?v=LxsuP9NPtNI
- BIP379 (formal grammar, draft): https://github.com/bitcoin/bips
