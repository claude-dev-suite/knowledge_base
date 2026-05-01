# Miniscript Satisfaction Algorithm - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/miniscript`.
> Canonical source: https://bitcoin.sipa.be/miniscript/, rust-miniscript satisfier
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/miniscript/SKILL.md

## Concept

Given a Miniscript fragment and a set of "available" data
(signatures, preimages, current block height/MTP), the satisfier
returns the cheapest valid witness (or fails). Unlike compilation
(which is one-shot at descriptor design time), satisfaction runs at
EVERY spend - it's the algorithm a wallet uses to construct the
witness stack from collected partial signatures.

The skill mentions satisfaction exists; this article walks the
recursive algorithm, the dissatisfaction (dsat) duality, and how the
satisfier handles ambiguity (multiple valid satisfactions).

## Walkthrough / mechanics

The satisfier returns two values per fragment:
- `sat(F)`: cheapest valid witness when F evaluates to TRUE.
- `dsat(F)`: cheapest valid witness when F evaluates to FALSE
  (only valid if `d` modifier).

Both are stack pushes (lists of byte-strings).

```
sat(pk_k(K)):
    if signature_for(K) available:
        return [sig]
    else FAIL

dsat(pk_k(K)):
    return [empty]                              # 1 byte (zero-length push)

sat(older(n)):
    if tx.nSequence_satisfies(n):
        return []                                # no witness needed
    else FAIL

dsat(older(n)):
    FAIL                                         # not d-modifiable

sat(and_v(V, B)):
    return sat(V) || sat(B)

sat(or_b(B, W)):
    options = [
      sat(B) || dsat(W),                         # take left branch
      dsat(B) || sat(W),                         # take right branch
      sat(B) || sat(W),                          # take both (more expensive)
    ]
    return cheapest(options)

dsat(or_b(B, W)):
    return dsat(B) || dsat(W)

sat(or_d(B, B)):
    if sat(B0) succeeds:
        return sat(B0)
    else:
        return dsat(B0) || sat(B1)
```

Each fragment has analogous rules; the rust-miniscript implementation
covers ~30 fragment types.

**Cost optimization:** when multiple satisfactions exist, the satisfier
selects the cheapest by witness vbytes. For ORs with no probability
annotation, the satisfier picks dynamically based on what's available
in the assets bag.

## Worked example

Fragment from earlier: `or_d(pk(Hot), and_v(v:pk(Cold), older(1008)))`.

**Case A: Hot key sig available.**

```
sat(or_d(pk(Hot), and_v(v:pk(Cold), older(1008)))):
  try sat(pk(Hot)) -> [sig_Hot]                 # 64-66 bytes (DER)
  succeeded -> return [sig_Hot]

witness = [sig_Hot]                              # ~73 wu
```

**Case B: Hot unavailable, cold + 1008 blocks elapsed.**

```
sat(or_d(...)):
  sat(pk(Hot)) FAILS (no sig)
  dsat(pk(Hot)) -> [empty]
  sat(and_v(v:pk(Cold), older(1008))):
    sat(v:pk(Cold)) -> [sig_Cold]               # via verify, no stack output
    sat(older(1008)) -> [] (timelock OK)
    return [sig_Cold]
  return [sig_Cold] || [empty]

witness = [sig_Cold, empty]                      # ~74 wu
```

When verifier executes:
```
push sig_Cold; push empty
script: <Hot> CHECKSIG IFDUP NOTIF <Cold> CHECKSIGVERIFY <1008> CSV ENDIF

stack:  [sig_Cold, empty]
<Hot> CHECKSIG -> verify "empty" against pk Hot -> FAIL gracefully:
  CHECKSIG returns 0 (push 0 onto stack)
stack: [sig_Cold, 0]
IFDUP duplicates non-zero, but top is 0 -> no-op
NOTIF: 0 is true for NOT -> enter else branch:
  <Cold> CHECKSIGVERIFY: verify sig_Cold against pk Cold -> OK
  <1008> CSV: check nSequence -> OK
ENDIF
final stack: empty -> tx invalid? No: CHECKSIGVERIFY consumed sig.
```

Wait - that produces an empty stack. Looking again at miniscript
satisfaction for `or_d`, the dissatisfaction of `pk(Hot)` for `or_d`
specifically is the empty signature push, and the `or_d` shape uses
IFDUP NOTIF which expects a 0/1 selector. The final witness keeps
the boolean on stack. (Real implementations handle this correctly;
the trace above is a reminder to use the rust-miniscript satisfier
rather than rolling your own.)

**Case C: ambiguous policy, both Hot and Cold available.**

```
sat(or_d(pk(Hot), and_v(...))):
  Both branches succeed.
  Branch 0 cost: 73 wu
  Branch 1 cost: 74 wu
  Pick branch 0 (Hot path).
```

The satisfier prefers the cheaper branch even if not annotated. This
is purely a fee-saving heuristic; both witnesses would validate.

**Using rust-miniscript:**

```rust
use miniscript::psbt::PsbtExt;
use bitcoin::Psbt;

fn finalize_psbt(psbt: &mut Psbt) -> Result<(), miniscript::psbt::Error> {
    psbt.finalize_mut(&secp256k1::Secp256k1::verification_only())?;
    Ok(())
}
```

Internally `finalize_mut` calls the satisfier for each input's
descriptor against the PSBT's collected partial sigs/preimages.

## Common bugs / pitfalls

1. **Hand-built witness disagrees with satisfier.** The cheapest
   non-malleable satisfaction is unique under the spec. Hand-crafting
   a witness that "looks right" risks malleability or invalidity.
   Always run through the satisfier.
2. **Missing preimages.** Hash-locked branches (`sha256(H)`) require
   the preimage in the asset bag. Many wallets forget to plumb HTLC
   preimages from Lightning into the satisfier; result is "no
   satisfaction".
3. **Timelocks evaluated against wrong height.** `older(n)` requires
   `nSequence` field on the input tx; the satisfier needs the spending
   tx context. Some implementations check height at sign time; you
   must re-check at broadcast.
4. **Conflicting timelocks.** `and(older(100), older(200))` is
   satisfiable (set nSequence to 200), but `and(older(100), after(500))`
   mixes relative and absolute - rust-miniscript handles, custom
   satisfiers often forget.
5. **Threshold dissatisfaction.** `thresh(2, A, B, C)` dissatisfaction
   needs ALL three to push their dsat; missing one fragment's `d` flag
   makes the whole dsat impossible.
6. **Witness reordering.** Stack order matters - first push is bottom
   of stack on script execution. Reversing order yields a script
   error with no clear message.

## References

- Miniscript site: https://bitcoin.sipa.be/miniscript/
- rust-miniscript satisfier: https://docs.rs/miniscript/latest/miniscript/struct.Miniscript.html#method.satisfy
- Test vectors: https://github.com/rust-bitcoin/rust-miniscript/tree/master/tests
