# Miniscript Fragment Types and Modifiers - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/miniscript`.
> Canonical source: https://bitcoin.sipa.be/miniscript/, BIP379 draft
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/miniscript/SKILL.md

## Concept

Every miniscript expression has a TYPE (one of B, V, K, W) plus a set
of MODIFIER properties (z, o, n, d, u, e, f, m, x). The type system is
how miniscript guarantees correctness, non-malleability, and
satisfiability. Wrappers (`a:`, `s:`, `c:`, `t:`, `d:`, `v:`, `j:`,
`n:`, `l:`, `u:`) convert between types/modifiers and are part of the
fragment grammar. The skill lists the basics; this article enumerates
each type, each modifier, and shows how the type system rules out
ambiguous or malleable scripts.

## Walkthrough / mechanics

**Types (basic categories):**

| Type | Stack behaviour | Where used |
|------|-----------------|------------|
| B | Boolean, leaves 0/1 on stack. | Top-level scripts, OR branches. |
| V | Verify, leaves nothing. Fails if condition false. | Inside `and_v`. |
| K | Key. Leaves a public key on stack (used before CHECKSIG). | Inside CHECKSIG fragments. |
| W | Wrapped. Acts on stack item one position back. | Inside `or_b`, `thresh`. |

A B fragment can be top-level. A V fragment cannot. To finalize a
script, the outermost expression must be B (or convertible to B via
wrapper).

**Modifiers (property bits):**

| Mod | Name | Meaning |
|-----|------|---------|
| z | zero-arg | Consumes no stack inputs (e.g., `older(n)`). |
| o | one-arg | Consumes exactly one stack input. |
| n | nonzero | Always pushes a nonzero value when satisfied. |
| d | dissatisfiable | Has a non-malleable dissatisfaction. |
| u | unique-sat | Has exactly one satisfaction that pushes 1. |
| e | expression | Dissatisfaction is "easy" (cheap and free of sigs). |
| f | forced | Dissatisfaction is impossible (any non-failing exec is sat). |
| m | non-malleable | No third-party can alter the witness without invalidating. |
| s | safe | All sigs are bound to the tx, no replay surface. |
| x | non-canonical-sat | Satisfaction allows nonstandard pushes. |

**Wrapper grammar:**

```
a:X  -> ALTSTACK wrapping (B -> W)
s:X  -> SWAP wrapping (B -> W with swap)
c:X  -> CHECKSIG wrapping (K -> B)
t:X  -> append <1> (V -> B)
d:X  -> IF/ELSE/ENDIF dissat (Vu -> B)
v:X  -> VERIFY wrapping (B -> V)
j:X  -> "If 0NOTEQUAL else..." (Bn -> B)
n:X  -> 0NOTEQUAL (B -> B with type fix)
l:X  -> if-else with literal 0 left
u:X  -> if-else with literal 0 right
```

Wrappers are stacked: `t:and_v(v:pk(A), pk(B))` means apply `t:` to
`and_v(v:pk(A), pk(B))`. Modifiers track through wrappers.

## Worked example

**Building 2-of-2 + timelock recovery:**

```
policy = or(and(pk(A), pk(B)), and(older(1008), pk(Recovery)))
```

Compiler emits (one possible canonical form):

```
or_d(
  multi(2, A, B),           # K type wrapped to B via implicit
  and_v(
    v:older(1008),          # B -> V via v:
    pk(Recovery)            # K -> B via implicit c:
  )
)
```

Type analysis:

```
multi(2, A, B): type B, modifiers ndu  -- non-zero, dissatisfiable, unique sat
older(1008):     type B, modifiers fz   -- forced, zero-arg
v:older(1008):  type V (from B via v:)
pk(Recovery):   type K, modifiers ndu (raw key)
                wrapped to B by surrounding context (CHECKSIG)
and_v(V, B) -> type B (carries forward modifiers)

or_d(B, B) -> type B
```

Final type: B (good for top-level).

**Counter-example: type-incorrect script.**

```
or_b(older(1008), pk(A))                 # WRONG: older is z (no stack input)
                                         # but or_b wants its left arg
                                         # to consume a stack item
```

The compiler refuses; you must wrap:

```
or_b(after(1008), s:pk(A))               # correct: s:pk(A) is W type
```

Or restructure with `or_d` which doesn't have the W requirement on
the right.

**Modifier-driven optimizations:**

```
thresh(2, pk(A), pk(B), pk(C))
```

Each branch in thresh must be Bdu (dissatisfiable, unique-sat) so the
satisfier can dissatisfy the unused branches without ambiguity. If
`pk(A)` weren't `d`, you couldn't form `thresh` over it.

`pk(A)` IS `Bdu` because:
- Sat: `[sig]` (unique).
- Dsat: `[empty]` (cheap).
- Boolean: 0 or 1.

A fragment like `older(1008)` is NOT `d` (you can't dissatisfy a
timelock cheaply), so `thresh(k, ..., older(...), ...)` is a type
error. Wrap with `j:` if you need optional timelock semantics.

## Common bugs / pitfalls

1. **Confusing V and B.** Top-level must be B. A bare V at the top
   leaves the stack empty, which is a script failure.
2. **Wrapper order matters.** `c:t:pk_k(A)` is different from
   `t:c:pk_k(A)`. The inner-most wrapper applies first.
3. **Forgetting the `c:` for raw keys in legacy.** `pk(A)` is
   `c:pk_k(A)` under the hood; if you build manually, you must include
   the CHECKSIG.
4. **Tapscript `multi_a` type mismatch.** In Tapscript context,
   `multi_a` produces B directly (not K), because there's no
   CHECKMULTISIG-equivalent. Mixing legacy `multi` in `tr()` is a
   type error.
5. **Missing `m` (non-malleable) flag.** A miniscript that lacks
   `m` allows a third-party observer with mempool access to mutate
   the witness without invalidating - a malleability attack. The
   compiler refuses to emit non-`m` scripts; if you assemble manually,
   check the property table.
6. **`u` (unique sat) violation in OR.** `or_b(A, A)` is type-correct
   but has TWO satisfactions (left or right). Without `u`, the
   satisfier may produce different witnesses on retries, breaking
   PSBT idempotence.
7. **Custom contexts with weird limits.** Tapscript miniscript has
   different size limits than legacy (no 520-byte stack item limit
   on initial pushes; CHECKSIG budget). Same fragment may be
   acceptable in one context and not the other.

## References

- Miniscript site (full type table): https://bitcoin.sipa.be/miniscript/
- rust-miniscript types module: https://docs.rs/miniscript/latest/miniscript/types/
- BIP379 draft (Miniscript spec): https://github.com/bitcoin/bips
- Pieter Wuille's correctness proof slides
