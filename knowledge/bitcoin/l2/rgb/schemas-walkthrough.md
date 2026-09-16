# RGB Schemas Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/rgb`.
> Canonical source: RGB-WG specifications and AluVM docs
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/rgb/SKILL.md

> **Superseded on the RGB-WG v0.12 line (September 2026).** RGB v0.12 (`rgb-core`
> 0.12.0, 10 July 2025) removed schemata from the wallet-facing model entirely:
> instead of importing schemata, interfaces and other components, wallets consume
> *issuers* published by contract developers plus a contract *API*. Interfaces now
> live only inside the **Contractum** language, whose compiler emits those APIs.
> AluVM was also replaced by zk-AluVM. Contracts issued with pre-v0.12 RGB-WG
> releases are not compatible with v0.12 and RGB-WG recommends re-issuing them.
>
> Everything below still describes the live model of the `rgb-protocol` **v0.11.1
> line** (`0.11.1-rc.11`, 15 July 2026), which keeps schemata and classic AluVM and
> is what `rgb-lib` and `rgb-lightning-node` build on -- so it remains the model you
> will meet in RGB Lightning today. Read it as the v0.11.1 model, not as v0.12.

## Concept

A **schema** in RGB is the contract type-definition: it specifies what state fields a
contract has, what operations are allowed, and what AluVM scripts validate transitions.
The schema is part of the contract's genesis -- once issued, it cannot be changed.

Two naming systems are easy to confuse:
- **Schemata** are the contract templates. `rgb-schemas` (v0.11.1 line) ships **NIA**
  (non-inflatable fungible asset), **UDA** (unique digital asset / single-token NFT
  with attachment) and **CFA** (collectible fungible asset), plus **PFA**
  (permissioned fungible asset) and **IFA** (inflatable fungible asset), both marked
  *not production-ready* in the repo as of `0.11.1-rc.11` (July 2026).
- **RGB20 / RGB21 / RGB25** are *interface* names -- the wallet-facing ABI layered
  over a schema, from `RGB-WG/rgb-interfaces` (RGB21 is the one still carried as a
  named library in that repo's tree). This article uses the RGB2x interface names
  for readability; the underlying schemata are the NIA/UDA/CFA set above.

Issuers can also write custom schemas for use cases like supply-chain tracking,
medical records, and structured financial products.

## Walkthrough / mechanics

### Schema structure

```
Schema {
  schema_id: hash of canonical encoding,
  metadata_types:    { ticker: AsciiPrintable, name: Unicode, ... },
  global_state_types: { circulating_supply: U64, ... },
  owned_state_types:  { fungible: AmountU64, attachment: BlobHash, ... },
  transitions:        { transfer: TransitionSchema, issue: ..., burn: ... },
  genesis:            GenesisSchema { required_globals, required_metadata },
  scripts:            { transfer: AluVM bytecode, ... }
}
```

Illustrative pseudo-structure, not the wire format: the real `Schema` on the v0.11.1
line carries `ffv`, `name`, `meta_types`, `global_types`, `owned_types`, `genesis`,
`transitions` and `default_assignment`. *Valencies* -- the public-rights state type
RGB-WG's v0.11.0 betas carried as `valency_types: TinyOrdSet<ValencyType>` -- were
dropped before v0.11.1 and are in neither line's `Schema` as of September 2026.

Each transition references a schema-defined operation (e.g., "transfer") and is checked
by the schema's AluVM script before being accepted by validating wallets.

### RGB20 interface / NIA schema (fungible)

The most-used pairing: the RGB20 interface over the NIA schema. State:

- `globals`: `circulating_supply: u64`, `nominal_unit: AsciiString`, `precision: u8`.
- `owned_state`: `assignments` -> `Amount: u64` per output seal.
- `transitions`:
  - `genesis`: issue total `circulating_supply`, set globals.
  - `transfer`: spend N inputs, create N' outputs preserving `sum(inputs) == sum(outputs)`.
  - `burn` (optional): destroy supply by spending without re-creating.

AluVM script for `transfer`:

```
sum_of_input_amounts == sum_of_output_amounts
all_amounts >= 0
```

(The actual bytecode is more involved; this is the high-level invariant.)

### RGB21 interface / UDA schema (collectibles)

NFT-like. State:

- `globals`: collection metadata, attachment hashes (off-chain media).
- `owned_state`: each token is an enumerated identifier with optional attachment.
- `transitions`: `transfer` (1 token in -> 1 token out, no fractional split).

### RGB25 interface / CFA schema (collectible fungible assets)

The interface over the **CFA** schema (`CollectibleFungibleAsset`): a fungible asset
carrying collectible metadata (article, name, details, precision, terms). It is *not* a
reissuance schema -- at `0.11.1-rc.11` (July 2026) `cfa.rs` declares a single
`TS_TRANSFER` transition, so supply is fixed at genesis exactly as in NIA.

Post-genesis inflation lives in the separate **IFA** schema, whose `ifa.rs` adds
`TS_INFLATION`, `TS_BURN` and `TS_LINK` transitions; `rgb-schemas` marks IFA
*not production-ready* as of `0.11.1-rc.11` (July 2026). This is the schema a
supply-tracks-fiat-reserves stablecoin would need.

### Custom schemas

Developers write schemas via the `rgb-schemata` Rust toolchain. Steps:

1. Define types in Strict-encoded TOML/CBOR.
2. Author AluVM scripts for each transition operation.
3. Compile to a schema artifact + register in a schema universe.
4. Issuers can now create contracts referencing the schema_id.

Custom schemas have been demoed for:
- Real-estate fractional ownership (RGB-RE).
- Carbon credits with retirement (RGB-Carbon).
- Encrypted health records with consent transitions.

### AluVM

RGB's virtual machine on the v0.11.1 line (`rgb-protocol/rgb-aluvm`). Register-machine
with:
- Strict resource limits (no unbounded loops).
- Cryptographic primitives as opcodes (hash, sig verify).
- Decidable termination (designed to halt).

This is comparable in spirit to Stacks' Clarity (which is also decidable and LISP-like)
rather than EVM's general-purpose Solidity.

RGB v0.12 replaced it with **zk-AluVM**: a Turing-complete zk-VM with a
non-von-Neumann architecture, 40 instructions and read-once memory matching single-use
seal semantics. Contract state there is finite-field elements, replacing the three
state variants of RGB's pre-v0.12 design (fungible via Pedersen commitments plus
Bulletproofs, structured non-fungible, binary attachments) that the 10 July 2025
release post describes; v0.12 dropped that design in favour of zk-STARKs. Note the
v0.11.1 code itself stores fungible state as a revealed `u64`, not as a Pedersen
commitment.

## Worked example

Issuing an RGB20 stablecoin pegged to USD reserves:

```
Step 1: Issuer (BankCo) authors genesis
  schema_id = NIA   // the RGB20 interface is the wallet-facing ABI over it
  metadata: { ticker: "BANKCOUSD", name: "BankCo USD Stablecoin", precision: 6 }
  globals:  { circulating_supply: 1_000_000_000_000 }   // 1M tokens, 6-decimals
  owned:    Assignment(seal=tx:0, amount=1_000_000_000_000)
  Issuer signs genesis. Stored at https://bankco-rgb.example/genesis.

Step 2: BankCo distributes 100 USD to Alice
  Genesis tx:0 spent in transfer_1.
  AluVM check: sum_in (1B) == sum_out (1B) -> pass.
  Outputs:
    Alice_seal: 100_000_000  (100 tokens at 6 decimals)
    BankCo_seal: 999_999_900_000_000

Step 3: Alice transfers 10 USD to Bob
  transfer_2 spends Alice's transfer_1 output.
  Outputs:
    Bob_seal:  10_000_000
    Alice_seal: 90_000_000

Step 4: Audit
  Anyone with the schema_id and BankCo's public key can validate:
  - Trace any UTXO back to genesis.
  - Confirm total supply unchanged (1B tokens).
  - Confirm no minting is possible at all: NIA/CFA have only a transfer
    transition, so supply is whatever genesis declared.
```

## Trade-offs and security

- **Schema immutability**: bug in schema script = contract is bricked or exploitable.
  Fix requires migrating to a new contract (off-protocol coordination). The v0.12
  cutover is the same problem at protocol scale: RGB-WG recommends re-issuing every
  pre-v0.12 contract.
- **Line divergence**: a contract issued against a v0.11.1 schema cannot be read by a
  v0.12 wallet, and vice versa. Pick a line before issuing.
- **AluVM tooling immaturity**: writing custom schemas is currently expert-level. Most
  ecosystem still uses the stock NIA / UDA schemata behind the RGB20 / RGB21
  interfaces.
- **Validator divergence**: different RGB wallet versions may implement AluVM
  semantics with subtle differences. RGB-WG provides a conformance suite.
- **Schema governance**: on the v0.11.1 line the stock schemata (NIA, UDA, CFA, plus
  the not-production-ready PFA and IFA) ship from `rgb-protocol/rgb-schemas`, while the
  interface definitions layered over them live in `RGB-WG/rgb-interfaces` -- where, as
  of September 2026, RGB21 is the only one still carried as a named library
  (`LIB_NAME_RGB21`). Wide adoption depends on issuer trust in the stock schemata.
- **Cross-schema interaction**: contracts under different schemas cannot directly
  exchange state; bridging requires off-protocol agreement (atomic swaps, custodied
  conversion).
- **Compared to ERC-20**: an RGB20 transfer is fundamentally private and scales without
  consensus, but loses the global-discoverability of ERC-20. Buyers of an RGB20 must
  obtain the full proof chain off-chain rather than reading a public ledger.

## References

- RGB Working Group specs - https://github.com/RGB-WG/specs
- AluVM docs - https://www.aluvm.org/
- RGB schemata, v0.11.1 line - https://github.com/rgb-protocol/rgb-schemas
- RGB interfaces - https://github.com/RGB-WG/rgb-interfaces
- RGB v0.12 consensus release (10 July 2025) - https://rgb.tech/blog/release-v0-12-consensus/
- rgb-core v0.12.0 release - https://github.com/RGB-WG/rgb-core/releases/tag/v0.12.0
