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
use miniscript::Descriptor;
use miniscript::bitcoin::PublicKey;
use std::str::FromStr;

let pol_str = "thresh(2,pk(02aaaa...),pk(02bbbb...),and(pk(02cccc...),older(144)))";
let policy: Concrete<PublicKey> = pol_str.parse()?;
let ms = policy.compile_to_miniscript_segwit_v0()?;
let desc = Descriptor::new_wsh(ms)?;
println!("descriptor: {}", desc);
println!("address:    {}", desc.address(miniscript::bitcoin::Network::Bitcoin)?);
```

`compile_to_miniscript_segwit_v0` picks the smallest correct script
under the segwit-v0 fragment set. For Tapscript leaves, use
`compile_to_miniscript_tap()` which exposes a different fragment
set including `multi_a` (BIP342).

## Common pitfalls

- Checksum drift: editing a descriptor by hand invalidates the `#xxxx`
  suffix. Recompute with `desc.to_string_with_secret(...)` or
  `Descriptor::to_string` (which adds a fresh checksum).
- Mixing rust-bitcoin versions: miniscript 11 -> 12 each pin
  rust-bitcoin majors. A type from `bitcoin = "0.31"` won't satisfy a
  miniscript 12 trait bound expecting `0.32`.
- Tapscript miniscript fragment set differs from segwit-v0; copying
  `older(144)` wrappings between contexts can fail to compile.
- `at_derivation_index` requires a wildcard descriptor -- a fixed
  descriptor will return an error rather than ignore the index.
- `multipath` (`<0;1>`) returns a `Vec<Descriptor>`; iterate both
  branches or extract one with `into_single_descriptors()`.

## References

- Repo: https://github.com/rust-bitcoin/rust-miniscript
- Site: https://bitcoin.sipa.be/miniscript/
- BIP380 (descriptors): https://github.com/bitcoin/bips/blob/master/bip-0380.mediawiki
- Companion: [rust-bitcoin/SKILL.md](../rust-bitcoin/SKILL.md), [bdk/SKILL.md](../bdk/SKILL.md)
