# MuSig2 Tweak Integration with Taproot - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/musig2`.
> Canonical source: BIP327 sec. "Tweaking the Aggregate Public Key" + BIP341
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/musig2/SKILL.md

## Concept

A MuSig2 aggregate key Q on its own is a perfectly valid 32-byte BIP340
public key. To use it in a Taproot output it must absorb the **TapTweak** from
BIP341, which folds in the (possibly empty) script-tree commitment. BIP327
distinguishes two tweak modes — **plain** (additive only) and **x-only**
(parity-aware) — and tracks an accumulator `(gacc, tacc)` so partial sigs land
on the final tweaked key. Misordering or mis-typing a tweak produces a sig that
verifies against the wrong point. This article works through the math of both
tweak types and the canonical Taproot binding.

## Walkthrough / mechanics

### Per-tweak update rule (BIP327 sec. "ApplyTweak")

State: `(Q, gacc, tacc)` from `KeyAgg`. Initially `gacc=1`, `tacc=0`.

For each tweak `t` with mode `is_xonly_t`:

```
if is_xonly_t and not has_even_y(Q):
    g = -1 mod n          // need to flip sign on aggregated x-only point
else:
    g = 1
Q     = g * Q + t * G
gacc  = (g * gacc) mod n
tacc  = (g * tacc + t) mod n
```

`gacc` is the cumulative parity factor that each signer's secret eats during
`Sign`. `tacc` is the cumulative additive scalar that the **coordinator** eats
during `PartialSigAgg`. The split is what makes per-signer signing key-agnostic
of the additive tweak — only one party needs to know it for aggregation.

### Sign-time application

In `Sign`:
```
d_i' = a_i * d_i               // KeyAgg coefficient absorbed
if not has_even_y(Q): d_i' = n - d_i'
d_i' = (gacc * d_i') mod n     // <-- the tweak's parity factor
s_i  = (k1' + b*k2' + e * d_i') mod n
```

In `PartialSigAgg`:
```
s = (sum(s_i) + e * tacc) mod n   // <-- the tweak's additive factor
```

Verifier checks `s*G == R + e*Q_final`. Because `Q_final = gacc*Q_agg + tacc*G`
and each `d_i'` is scaled by `gacc`, the cross-multiplication works out.

### Mapping to Taproot (BIP341)

A Taproot output key is:
```
P_out = lift_x(Q_internal) + t_tap * G
       where t_tap = int(tagged_hash("TapTweak", xbytes(Q_internal) || merkle_root)) mod n
```

If the script tree is empty, `merkle_root` is an empty byte string, but the
tagged hash still includes the tag — `t_tap` is non-zero in practice.

In MuSig2:
```
1. KeyAgg(X) -> Q_internal, KeyAggCtx with (gacc=1, tacc=0)
2. Compute t_tap from xbytes(Q_internal) and merkle_root
3. ApplyTweak(t=t_tap, is_xonly_t=True)
   -> KeyAggCtx now holds (Q_out, gacc, tacc)
   xbytes(Q_out) IS the Taproot output key (after even-y normalization)
4. Sign / aggregate using updated KeyAggCtx
```

The output script is `OP_1 <xbytes(Q_out)>` — indistinguishable from a
single-key Taproot output.

### x-only vs plain mode

```
| Mode      | Use case                                     | gacc behavior |
|-----------|----------------------------------------------|---------------|
| x-only    | TapTweak (BIP341) -- output is x-only        | flips on odd  |
| plain     | BIP32 child derivation, additive blinding    | unchanged     |
```

Plain tweaks (e.g., adding a BIP32 chain code derivative) do **not** flip
parity. x-only tweaks must, because the BIP340 verifier only accepts even-y
points and the verifier re-lifts `xbytes(Q_out)` as even-y.

## Worked example

Two-of-two MuSig2 with key-path-only Taproot (no script tree):

```
sk_A = 0x...AA  -> P_A = sk_A * G
sk_B = 0x...BB  -> P_B = sk_B * G

X = sorted([xbytes(P_A), xbytes(P_B)])
KeyAgg(X) -> Q_int, KeyAggCtx(Q_int, gacc=1, tacc=0)

merkle_root = b""    // key-path-only
t_tap = int(tagged_hash("TapTweak", xbytes(Q_int))) mod n

ApplyTweak(t_tap, is_xonly=True)
  if not has_even_y(Q_int): g = n-1 else g = 1
  Q_out = g * Q_int + t_tap * G
  gacc  = g
  tacc  = t_tap   (because g*0 + t_tap = t_tap when g=1; -t_tap when g=-1)

scriptPubKey = OP_1 || xbytes(Q_out)
```

When spending, signers run NonceGen / NonceAgg as usual against `KeyAggCtx`.
Signer A's partial sig:

```
d_A_eff = a_A * sk_A
if not has_even_y(Q_out): d_A_eff = n - d_A_eff
d_A_eff = gacc * d_A_eff mod n
s_A = (k1 + b*k2 + e * d_A_eff) mod n
```

Coordinator:

```
s = (s_A + s_B + e * tacc) mod n
final = xbytes(R) || sbytes(s)
```

`final` is a vanilla 64-byte BIP340 sig and goes into the witness as a single
push. No annex, no script, no control block.

## Common pitfalls

- **Tweaking before KeyAgg.** Cannot work — `Q_internal` doesn't exist yet.
  Must be `KeyAgg -> ApplyTweak`.
- **Using `is_xonly_t=False` for TapTweak.** Produces a sig that verifies
  against `Q_internal + t*G` but not against the even-y-lifted Taproot key.
- **Forgetting `gacc` in `Sign`.** The partial sigs are then over the
  untweaked Q, and PartialSigAgg's `e*tacc` term cannot rescue them — the
  Sign-time `gacc` factor is multiplicative, the Agg-time `tacc` is additive,
  and they are not interchangeable.
- **Stacking tweaks in the wrong order.** BIP32 derivation tweak is plain;
  Taproot tweak is x-only. If you BIP32-derive after Taproot-tweaking, the
  parity tracking double-flips.

## References

- BIP327 sec. "Tweaking the Aggregate Public Key":
  https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki#tweaking-the-aggregate-public-key
- BIP341: https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
- libsecp256k1 musig tweak tests: `src/modules/musig/tests_impl.h`
- rust-bitcoin Taproot crate: https://docs.rs/bitcoin/latest/bitcoin/taproot/
