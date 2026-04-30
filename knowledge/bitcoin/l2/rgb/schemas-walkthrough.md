# RGB Schemas Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/rgb`.
> Canonical source: RGB-WG specifications and AluVM docs
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/rgb/SKILL.md

## Concept

A **schema** in RGB is the contract type-definition: it specifies what state fields a
contract has, what operations are allowed, and what AluVM scripts validate transitions.
RGB ships standard schemas (RGB20 fungible, RGB21 non-fungible/collectible, RGB25 for
contracts with reissuance, RGB30+ for complex DeFi primitives) and lets issuers write
custom schemas for use cases like supply-chain tracking, medical records, and structured
financial products. The schema is part of the contract's genesis -- once issued, it
cannot be changed.

## Walkthrough / mechanics

### Schema structure

```
Schema {
  schema_id: hash of canonical encoding,
  metadata_types:    { ticker: AsciiPrintable, name: Unicode, ... },
  global_state_types: { circulating_supply: U64, ... },
  owned_state_types:  { fungible: AmountU64, attachment: BlobHash, ... },
  valencies:          { renewable: ..., burnable: ... },
  transitions:        { transfer: TransitionSchema, issue: ..., burn: ... },
  genesis:            GenesisSchema { required_globals, required_metadata },
  scripts:            { transfer: AluVM bytecode, ... }
}
```

Each transition references a schema-defined operation (e.g., "transfer") and is checked
by the schema's AluVM script before being accepted by validating wallets.

### RGB20 (fungible)

The most-used schema. State:

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

### RGB21 (collectibles)

NFT-like schema. State:

- `globals`: collection metadata, attachment hashes (off-chain media).
- `owned_state`: each token is an enumerated identifier with optional attachment.
- `transitions`: `transfer` (1 token in -> 1 token out, no fractional split).

### RGB25 (with reissuance)

Adds a "renewable" valency: the issuer can mint additional supply post-genesis if they
hold a special "renewal right" UTXO (themselves a single-use seal). Used for stablecoins
where supply tracks fiat reserves.

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

RGB's virtual machine. Stack-based with:
- Strict resource limits (no unbounded loops).
- Cryptographic primitives as opcodes (hash, sig verify).
- Decidable termination (designed to halt).

This is comparable in spirit to Stacks' Clarity (which is also decidable and LISP-like)
rather than EVM's general-purpose Solidity.

## Worked example

Issuing an RGB20 stablecoin pegged to USD reserves:

```
Step 1: Issuer (BankCo) authors genesis
  schema_id = RGB20 v1
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
  - Confirm only BankCo can mint (renewable valency restricted to BankCo's seal).
```

## Trade-offs and security

- **Schema immutability**: bug in schema script = contract is bricked or exploitable.
  Fix requires migrating to a new contract (off-protocol coordination).
- **AluVM tooling immaturity**: writing custom schemas is currently expert-level. Most
  ecosystem still uses standard RGB20/21.
- **Validator divergence**: different RGB wallet versions may implement AluVM
  semantics with subtle differences. RGB-WG provides a conformance suite.
- **Schema governance**: standard schemas (RGB20, RGB21) are governed by the RGB
  Working Group. Wide adoption depends on issuer trust in standard schemas.
- **Cross-schema interaction**: contracts under different schemas cannot directly
  exchange state; bridging requires off-protocol agreement (atomic swaps, custodied
  conversion).
- **Compared to ERC-20**: an RGB20 transfer is fundamentally private and scales without
  consensus, but loses the global-discoverability of ERC-20. Buyers of an RGB20 must
  obtain the full proof chain off-chain rather than reading a public ledger.

## References

- RGB Working Group specs - https://github.com/RGB-WG/specs
- AluVM docs - https://www.aluvm.org/
- RGB schemata - https://github.com/RGB-WG/rgb-schemata
- "RGB v0.11" release notes - https://github.com/RGB-WG/rgb/releases
