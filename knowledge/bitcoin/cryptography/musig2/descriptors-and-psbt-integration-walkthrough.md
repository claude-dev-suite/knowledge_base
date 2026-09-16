# MuSig2 Descriptors and PSBT Integration - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/musig2`.
> Canonical source: BIP328 + BIP373 + BIP390, Bitcoin Core `doc/descriptors.md`
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/musig2/SKILL.md

## Concept

BIP327 stops at the signing math: it says how n keys aggregate and how two
rounds of nonces and partial sigs produce a BIP340 signature. It says nothing
about how a wallet *describes* such a key, how it derives fresh addresses from
it, or how the two rounds cross a process boundary. Three companion BIPs,
all authored by Ava Chow and all assigned 2024-06-04, fill that gap:

```
| BIP | Title                                        | Status (Sep 2026) |
|-----|----------------------------------------------|-------------------|
| 328 | Derivation Scheme for MuSig2 Aggregate Keys  | Complete          |
| 373 | MuSig2 PSBT Fields                           | Complete          |
| 390 | musig() Descriptor Key Expression            | Draft (v0.2.0)    |
```

Bitcoin Core has implemented all three since v30.0 (October 2025), with the
restrictions noted below. This article walks the stack from descriptor string
to signed PSBT.

## Walkthrough / mechanics

### BIP328 - the synthetic xpub

A MuSig2 aggregate key is an ordinary-looking 33-byte compressed point, so it
can be dressed up as a BIP32 extended public key and derived from directly,
instead of deriving every participant and re-aggregating per address.

```
synthetic_xpub = serialize_xpub(
    depth        = 0,
    child_number = 0,
    chaincode    = 868087ca02a6f974c4598924c36b57762d32cb45717167e300622c7167e38965,
    key          = agg_pubkey (33 bytes, plain aggregate, not x-only))
```

The chaincode is a fixed constant shared by every BIP328 synthetic xpub: it is
`SHA256("MuSig2MuSig2MuSig2")`. There is no aggregate private key, so only
**unhardened** derivation is possible.

Signing for a derived child is a tweak problem. At each `CKDpub` step the
`I_L` value is the tweak, and it enters the BIP327 session context with
**plain** mode (`is_xonly_t = false`) - contrast the Taproot tweak, which is
x-only (see [tweak-integration-with-taproot.md](tweak-integration-with-taproot.md)).
Every signer must recompute the whole derivation chain; the tweaks are applied
to the partial signatures during `Sign`.

### BIP390 - the musig() key expression

```
musig(KEY, KEY, ..., KEY)
musig(KEY, KEY, ..., KEY)/NUM/.../*
```

Placement rules:

- Legal only inside `tr()`, `rawtr()` or `sp()`.
- Cannot be nested inside another `musig()`.
- Participant public keys may be repeated.
- BIP390's invalid vectors reject it at the top level of `pk()` and `pkh()`,
  and anywhere in `wpkh()`, `combo()`, `sh()`, `wsh()`, `sh(wpkh())` or
  `sh(wsh())`. Inside a `tr()` script leaf, `pk(musig(...))` is valid.

Aggregation rules:

- Perform all per-participant derivation first.
- Then sort with `KeySort`, then aggregate. Because sorting happens after
  derivation and before `KeyAgg`, the order the keys are *written in* is
  irrelevant - the same set of keys always yields the same aggregate. This
  also means a user recovering from an un-backed-up descriptor does not have
  to guess the original ordering.

Rules for the trailing `/NUM/.../*` form (BIP328 derivation on the aggregate):

- Every participant `KEY` must be an xpub or derived from one.
- No participant may itself use `/*` or `/<NUM;NUM;...>` - a ranged or
  multipath `musig()` cannot contain ranged or multipath participants.
- No hardened steps: `/NUMh`, `/NUM'`, `/*h` and `/*'` are all rejected.

### BIP373 - the PSBT fields

```
| Field                                | Key type | Scope  |
|--------------------------------------|----------|--------|
| PSBT_IN_MUSIG2_PARTICIPANT_PUBKEYS   | 0x1a     | input  |
| PSBT_IN_MUSIG2_PUB_NONCE             | 0x1b     | input  |
| PSBT_IN_MUSIG2_PARTIAL_SIG           | 0x1c     | input  |
| PSBT_OUT_MUSIG2_PARTICIPANT_PUBKEYS  | 0x08     | output |
```

Flow across roles:

1. **Updater** adds `PSBT_IN_MUSIG2_PARTICIPANT_PUBKEYS` (and
   `PSBT_OUT_MUSIG2_PARTICIPANT_PUBKEYS`, to aid change detection) mapping
   each aggregate pubkey to its participants, so a signer that only knows its
   own key can discover it is a participant. It should also add the matching
   `PSBT_IN_TAP_BIP32_DERIVATION` entries. If there is no `PSBT_GLOBAL_XPUB`
   giving the synthetic xpub, derivation from the aggregate pubkey is assumed
   to follow BIP328.
2. **Signer, round 1** finds itself in the participant list (directly, or via
   `PSBT_IN_TAP_BIP32_DERIVATION`), and if no `PSBT_IN_MUSIG2_PUB_NONCE` for
   its key exists yet, runs `NonceGen` and writes one.
3. **Signer, round 2** - once every pubnonce is present - runs `NonceAgg` then
   `Sign`, applying any BIP32 derivation tweaks, and writes
   `PSBT_IN_MUSIG2_PARTIAL_SIG`.
4. **Final signer or finalizer** runs `PartialSigAgg` and places the resulting
   64-byte BIP340 signature in `PSBT_IN_TAP_KEY_SIG` or
   `PSBT_IN_TAP_SCRIPT_SIG`.

The two signer steps are two separate passes over the same PSBT, which is
exactly where secnonce state management bites (see
[state-machine-diagram.md](state-machine-diagram.md)).

## Worked example

BIP328 test vector - aggregate pubkey to synthetic xpub:

```
keys      = [02F9308A019258C31049344F85F89D5229B531C845836F99B08601F113BCE036F9,
             03DFF1D77F2A671C5F36183726DB2341BE58FEAE1DA2DECED843240F7B502BA659,
             023590A94E768F8E1815C2F24B4D80A8E3149316C3518CE7B7AD338368D038CA66]
agg_pk    = 0290539eede565f5d054f32cc0c220126889ed1e5d193baf15aef344fe59d4610c
synthetic = xpub661MyMwAqRbcFt6tk3uaczE1y6EvM1TqXvawXcYmFEWijEM4PDBnuCXwwVk5T
            FJk8Tw5WAdV3DhrGfbFA216sE9BsQQiSFTdudkETnKdg8k
```

BIP390 test vector - a ranged `musig()` inside `tr()`, where the two xpubs are
aggregated and then derived at `m/0/*`:

```
tr(musig(xpub6ERApfZwUNrhLCkDtcHTcxd75RbzS1ed54G1LkBUHQVHQKqhMkhgbmJbZRkrgZw4koxb5JaHWkY4ALHY2grBGRjaDMzQLcgJvLJuZZvRcEL,
         xpub68NZiKmJWnxxS6aaHmn81bvJeTESw724CRDs6HbuccFQN9Ku14VQrADWgqbhhTHBaohPX4CjNLf9fq9MYo6oDaPPLPxSb7gwQN3ih19Zm4Y)/0/*)

index 0 -> 51209508c08832f3bb9d5e8baf8cb5cfa3669902e2f2da19acea63ff47b93faa9bfc
index 1 -> 51205ca1102663025a83dd9b5dbc214762c5a6309af00d48167d2d6483808525a298
index 2 -> 51207dbed1b89c338df6a1ae137f133a19cae6e03d481196ee6f1a5c7d1aeb56b166
```

That exact descriptor is the example carried in Bitcoin Core's
`doc/descriptors.md`.

## Implementation status (as of September 2026)

- **Bitcoin Core** - `musig()` descriptor parsing and the BIP373 PSBT fields
  ship from v30.0 (2025-10-13); descriptor support merged in PR #31244
  (2025-07-31). Core accepts `musig()` **only inside `tr()`**, not `rawtr()`
  or `sp()`, and only followed by unhardened `/NUM` steps when all `KEY`
  subexpressions are xpubs and none use `/*` or `/<NUM;NUM;...>`. Follow-up
  fixes through 2026: #35316 (2026-05-21), #35269 (2026-06-02), #35493
  (2026-08-11), #34697 (2026-08-24, milestone 32.0).
- **Ledger Bitcoin app** - `musig()` in taproot wallet policies since v2.4.0
  (CHANGELOG 2025-03-07; GitHub release 2025-03-17). Limits per the app's
  `doc/musig.md`: at most 5 keys per `musig()`, at most 8 parallel signing
  sessions, `musig()` allowed among `multi_a` key expressions but not
  `sortedmulti_a`, and only `musig(...)/**` or `musig(...)/<M;N>/*` - schemes
  that derive each participant before aggregation are unsupported. v2.5.1
  (CHANGELOG 2026-09-09) fixed `musig()` inside `multi_a` fragments.
- **Coldcard** - Mk4 and Q1 both set `NGU_INCL_MUSIG = 0` in their board
  configs to save firmware space, so MuSig is compiled out entirely.

## Common pitfalls

- **Aggregating in written order.** BIP390 mandates `KeySort` *after*
  derivation and *before* `KeyAgg`. Aggregating in the order the descriptor
  spells the keys produces a different `Q` and therefore a different address.
- **Deriving participants and aggregating per index.** Legal in the plain
  `musig(KEY,...)` form, but the `musig(...)/NUM/.../*` form means the
  opposite: aggregate once, then derive the aggregate per BIP328. The two
  produce different scripts. Ledger supports only the latter.
- **Using an x-only tweak for BIP328 derivation.** The `CKDpub` `I_L` tweaks
  are plain (`is_xonly_t = false`). Only the BIP341 TapTweak is x-only.
- **Signing round 2 from a re-sent round-1 PSBT.** The signer must recognise
  that its own `PSBT_IN_MUSIG2_PUB_NONCE` is already present and match it
  against retained (or recomputed) secnonce state rather than generating a
  fresh nonce - regenerating silently invalidates the session, reusing the
  slot risks key leak.
- **Assuming Core takes `rawtr(musig(...))`.** BIP390 allows it; Core's
  descriptor parser, as documented in v31.1 (July 2026), does not.

## References

- BIP328: https://github.com/bitcoin/bips/blob/master/bip-0328.mediawiki
- BIP373: https://github.com/bitcoin/bips/blob/master/bip-0373.mediawiki
- BIP390: https://github.com/bitcoin/bips/blob/master/bip-0390.mediawiki
- Bitcoin Core descriptors doc:
  https://github.com/bitcoin/bitcoin/blob/master/doc/descriptors.md
- Bitcoin Core PR #31244 "descriptors: MuSig2":
  https://github.com/bitcoin/bitcoin/pull/31244
- Ledger Bitcoin app MuSig2 doc:
  https://github.com/LedgerHQ/app-bitcoin-new/blob/develop/doc/musig.md
