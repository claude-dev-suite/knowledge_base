# rust-miniscript Descriptor Parsing - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/miniscript-rs`.
> Canonical source: https://github.com/rust-bitcoin/rust-miniscript
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/miniscript-rs/SKILL.md

## Concept

rust-miniscript implements two related languages: output descriptors
(BIP380, the wallet-level spec) and Miniscript (the scripting subset
of Bitcoin Script with formal semantics). A descriptor wraps a
miniscript with a checksum, BIP32 paths, and a script context
(`Legacy`, `Segwitv0`, `Tap`).

Parsing produces a `Descriptor<DescriptorPublicKey>` (xpub-based,
no derivation done yet) which you can `at_derivation_index(i)` to
get a concrete descriptor with normal pubkeys, then call
`script_pubkey()` for an address-equivalent script. The same code
path supports `wpkh`, `wsh(multi(...))`, `tr(internal, {leaf_a, leaf_b})`,
multipath `<0;1>`, and inline xpubs.

## API walkthrough

```rust
use miniscript::{Descriptor, DescriptorPublicKey};
use miniscript::bitcoin::secp256k1::Secp256k1;
use miniscript::bitcoin::{Address, Network};
use std::str::FromStr;

let s = "wsh(multi(2,\
    [d34db33f/48h/0h/0h/2h]xpub6E.../0/*,\
    [11223344/48h/0h/0h/2h]xpub6F.../0/*\
))#xxxxxxxx";

// Parse + verify checksum
let desc = Descriptor::<DescriptorPublicKey>::from_str(s)?;
desc.sanity_check()?;          // policy invariants
println!("max sat size: {}", desc.max_weight_to_satisfy()?);

let secp = Secp256k1::verification_only();
for i in 0u32..3 {
    let derived = desc.at_derivation_index(i)?;
    let definite = derived.derived_descriptor(&secp)?;
    let addr: Address = definite.address(Network::Bitcoin)?;
    println!("[{i}] {addr}");
}
```

## Worked example: compile policy then derive scripts

A common pattern: human-friendly policy -> compiled miniscript ->
descriptor wrapper -> per-index scripts.

```rust
use miniscript::policy::Concrete;
use miniscript::{Descriptor, Segwitv0};
use miniscript::bitcoin::PublicKey;
use std::str::FromStr;

let pol_str = "thresh(2,pk(02aaaa...),pk(02bbbb...),and(pk(02cccc...),older(144)))";
let policy: Concrete<PublicKey> = pol_str.parse()?;
let ms = policy.compile::<Segwitv0>()?;    // needs the `compiler` feature
let desc = Descriptor::new_wsh(ms)?;
println!("descriptor: {}", desc);
println!("address:    {}", desc.address(miniscript::bitcoin::Network::Bitcoin)?);
```

`compile::<Segwitv0>` picks the smallest correct script under the
segwit-v0 fragment set. For Tapscript leaves, `compile::<Tap>()`
targets a different fragment set including `multi_a` (BIP342), and
`compile_tr(unspendable_key)` builds a whole `tr()` descriptor with a
Huffman-weighted TapTree. All three are gated on the non-default
`compiler` cargo feature (verified against rust-miniscript 13.1.0,
June 2026).

## Common pitfalls

- Checksum drift: editing a descriptor by hand invalidates the `#xxxx`
  suffix. Recompute with `desc.to_string_with_secret(...)` or
  `Descriptor::to_string` (which adds a fresh checksum).
- Mixing rust-bitcoin versions: each miniscript major pins one
  rust-bitcoin release range, and a type from the wrong one won't
  satisfy the trait bound -- `bitcoin = "0.31"` types will not
  satisfy a miniscript 12 or 13 bound expecting `0.32`. See the
  pairing table below.
- Tapscript miniscript fragment set differs from segwit-v0; copying
  `older(144)` wrappings between contexts can fail to compile.
- `at_derivation_index` requires a wildcard descriptor -- a fixed
  descriptor will return an error rather than ignore the index.
- `multipath` (`<0;1>`) returns a `Vec<Descriptor>`; iterate both
  branches or extract one with `into_single_descriptors()`.

## Version pairing with rust-bitcoin

As of September 2026, from each major's published `Cargo.toml`:

| miniscript | rust-bitcoin dependency |
|------------|-------------------------|
| 13.x (13.1.0, June 2026) | `^0.32.6` |
| 12.x (12.3.7, May 2026)  | `^0.32`   |
| 11.x (11.2.3, July 2025) | `^0.31`   |
| 10.x                     | `^0.30`   |

13.x therefore shares rust-bitcoin 0.32 with 12.x: the 13.0.0 break
(October 2025) was miniscript's own API, not a rust-bitcoin bump. That
release eliminated recursion throughout the library, rewrote the
Taproot API, replaced stringly-typed errors, removed
`Ctx::check_witness`, renamed `Miniscript::parse_insane` to
`decode_consensus` and `parse_with_ext` to `decode_with_ext`. 13.1.0
adds only one change: `Plan`'s `descriptor` field is now public.

The 12.x line is still maintained in parallel (12.3.6 April 2026,
12.3.7 May 2026) for crates that cannot take the 13.0.0 API break.

## References

- Repo: https://github.com/rust-bitcoin/rust-miniscript
- Site: https://bitcoin.sipa.be/miniscript/
- BIP380 (descriptors): https://github.com/bitcoin/bips/blob/master/bip-0380.mediawiki
- Companion: [rust-bitcoin/SKILL.md](../rust-bitcoin/SKILL.md), [bdk/SKILL.md](../bdk/SKILL.md)
