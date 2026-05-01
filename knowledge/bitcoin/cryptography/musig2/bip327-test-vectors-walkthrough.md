# BIP327 Test Vectors Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/musig2`.
> Canonical source: BIP327 reference vectors (`bip-0327/vectors/`)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/musig2/SKILL.md

## Concept

BIP327 ships five JSON vector files that exercise every callable in the reference
spec: `key_agg_vectors.json`, `nonce_gen_vectors.json`, `nonce_agg_vectors.json`,
`sign_verify_vectors.json`, and `tweak_vectors.json`. Each one isolates a single
algorithm so a divergent implementation can be located without running the full
two-round flow. This article walks the data path of one canonical vector through
all five files, naming the exact fields a verifier must reproduce byte-for-byte.

## Walkthrough / mechanics

The vectors share a fixture: three signer pubkeys `X[0..2]`, message `msg`, and
secnonces. Reproducing them follows BIP327 sections in order.

### Step 1 - KeyAgg (sec. "Key Aggregation")

```
X = [pk[0], pk[1], pk[2]]
KeyAggCtx = KeyAgg(X)
  L          = HashKeys(X)              // tagged hash "KeyAgg list"
  pk2        = GetSecondKey(X)          // second-unique key, special case a=1
  for i:
    if X[i] == pk2: a[i] = 1
    else: a[i] = int(tagged_hash("KeyAgg coefficient", L || X[i])) mod n
  Q' = sum(a[i] * cpoint(X[i]))         // cpoint = lift_x with even-y normalization
KeyAggCtx = (Q', gacc=1, tacc=0)
agg_pk    = xbytes(Q')                  // 32-byte x-only
```

`key_agg_vectors.json` asserts `agg_pk == expected`. A common mismatch is
forgetting `cpoint` returns the implicit even-y lift — feeding raw bytes into
addition produces a Q on a different parity branch.

### Step 2 - NonceGen / NonceAgg (sec. "Nonce Generation"/"Nonce Aggregation")

```
secnonce = NonceGen(sk, agg_pk?, msg?, extra_in?, rand)
  k1 = int(tagged_hash("MuSig/aux", rand)) mod n
  k2 = ... (independent extraction with extra_in)
  pubnonce = cbytes(k1*G) || cbytes(k2*G)   // 66 bytes, two 33-byte compressed points

aggnonce = NonceAgg([pubnonce_i for i in signers])
  R1 = sum(point(pubnonce_i[0:33]))
  R2 = sum(point(pubnonce_i[33:66]))
  if R1 == infinity: R1 = G   // BIP327 fix-up to keep aggnonce non-empty
  aggnonce = cbytes_ext(R1) || cbytes_ext(R2)
```

`cbytes_ext` is the 33-byte serialization that allows `0x00...00` for infinity.
The infinity-substitution for `R1` is the silent edge case most non-spec
implementations miss.

### Step 3 - Session and partial sig (sec. "Signing")

```
SessionCtx = (aggnonce, X, [tweaks], [is_xonly_t], msg)
GetSessionValues(SessionCtx):
  (Q, gacc, tacc) = ApplyTweaks(KeyAgg(X), tweaks, is_xonly_t)
  b = int(tagged_hash("MuSig/noncecoef", aggnonce || xbytes(Q) || msg)) mod n
  R = R1 + b*R2
  if R == infinity: R = G
  e = int(tagged_hash("BIP0340/challenge", xbytes(R) || xbytes(Q) || msg)) mod n

Sign(secnonce, sk, SessionCtx):
  k1' = k1; k2' = k2
  if not has_even_y(R): k1' = n - k1; k2' = n - k2
  d = sk; if Q.y odd: d = n - d
  d *= gacc                                  // accumulated parity from tweaks
  s = (k1' + b*k2' + e * a[i] * d) mod n
  return (R.x_bytes, s)                      // partial sig is just s
```

`sign_verify_vectors.json` carries the expected partial-sig hex. Off-by-one in
parity tracking (`gacc`) is the most common failure here.

### Step 4 - PartialSigAgg (sec. "Partial Signature Aggregation")

```
s = (sum(s_i) + e * tacc) mod n              // tacc is accumulated additive tweak
final_sig = xbytes(R) || sbytes(s)
```

The `e*tacc` term is what shifts a "fresh aggregate" sig into one valid for the
**tweaked** key. Forgetting it produces a 64-byte sig that verifies under the
untweaked Q but fails under Q+t*G.

## Worked example

From `key_agg_vectors.json` (vector 0), three pubkeys whose hex begins:

```
X[0] = F9308A019258C31049344F85F89D5229B531C845836F99B08601F113BCE036F9
X[1] = DFF1D77F2A671C5F36183726DB2341BE58FEAE1DA2DECED843240F7B502BA659
X[2] = 3590A94E768F8E1815C2F24B4D80A8E3149316C3518CE7B7AD338368D038CA66
```

Sorted-list canonical hash `L = SHA256(KeyAgg list || X[0] || X[1] || X[2])`.

Coefficient computation (with `pk2 = X[2]` as the second-unique key in this
fixture):

```
a[0] = int(SHA256("KeyAgg coefficient" || L || X[0])) mod n
a[1] = int(SHA256("KeyAgg coefficient" || L || X[1])) mod n
a[2] = 1
Q'   = a[0]*P[0] + a[1]*P[1] + 1*P[2]
agg_pk = xbytes(Q')
```

The vector expects `agg_pk = 90539EEDE565F5D054F32CC0C220126889ED1E5D193BAF15
AEF345FE69D34F34`.

For the signing branch, `nonce_agg_vectors.json` vector 0 supplies two pubnonces
`pnonce[0]`, `pnonce[1]` and asserts `aggnonce`. Then `sign_verify_vectors.json`
vector 0 takes that `aggnonce`, the message `msg = 0x00...01`, and asserts the
first signer's partial sig as a 32-byte hex string.

## Common pitfalls

- **Skipping fix-ups for infinity.** BIP327 requires `R1 = G` if `R1 = O` after
  nonce aggregation, and `R = G` if `R = O` after applying `b`. Reference vectors
  do not all hit these branches — write extra unit tests.
- **Wrong endianness on `s`.** Partial sig is big-endian 32-byte `s`; many
  off-the-shelf math libraries return little-endian.
- **Tweak ordering.** Vectors in `tweak_vectors.json` apply both x-only and
  plain tweaks in sequence; `gacc` flips with x-only tweaks but not plain ones.
- **Confusing `cbytes` (33-byte) with `xbytes` (32-byte x-only).** Pubnonces are
  cbytes, agg_pk is xbytes.

## References

- BIP327: https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki
- Reference vectors: https://github.com/bitcoin/bips/tree/master/bip-0327/vectors
- Reference implementation: `bip-0327/reference.py`
- libsecp256k1 musig module tests: `src/modules/musig/tests_impl.h`
