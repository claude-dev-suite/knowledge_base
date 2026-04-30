# Tapscript vs Legacy Script - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/scripts`.
> Canonical source: https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/scripts/SKILL.md

## Concept

Tapscript (BIP342) is the script-execution rule set used inside taproot script-path spends. While syntactically similar to legacy script, it changes 1) opcode semantics (notably CHECKMULTISIG removal), 2) signature opcode rules, 3) sigops accounting (budget instead of count), 4) extension surface (OP_SUCCESS opcodes). This article enumerates each delta with the concrete byte-level differences.

## Walkthrough / mechanics

### Opcode set differences

| Opcode | Legacy / segwit v0 | Tapscript (BIP342) |
|--------|--------------------|--------------------|
| `OP_CHECKMULTISIG` (0xae) | Verifies M-of-N | **OP_SUCCESS** (script always succeeds!) |
| `OP_CHECKMULTISIGVERIFY` (0xaf) | Like above + VERIFY | **OP_SUCCESS** |
| `OP_CHECKSIGADD` (0xba) | undefined / OP_NOP | **New**: pops `<sig> <num> <pubkey>`, pushes `num + 1` if sig valid else `num` |
| 80, 98, 126-129, 131-134, 137-138, 141-142, 149-153, 187-254 | various OP_NOPs / undefined | **OP_SUCCESS** (anyone-can-spend if reached) |
| `OP_CODESEPARATOR` | costly behavior | retained but with new semantics: position is committed to in sighash |

### Stack / push limits

| Limit | Legacy | Tapscript |
|-------|--------|-----------|
| Max script size | 10_000 bytes | unlimited (constrained by witness weight) |
| Max stack item size | 520 bytes | unlimited (constrained by witness weight) |
| Max stack + altstack items | 1000 | 1000 |
| `MINIMALDATA` | flag-controlled | **always required** |
| `MINIMALIF` | flag-controlled | **always required** |
| Sigops | counted per-script (max 80k per block) | **budget** = floor(witness_weight / 50); each CHECKSIG/CHECKSIGADD costs 50 |

### Signature opcode rules

In Tapscript:

- Signatures must be 64 or 65 bytes (Schnorr). Any other length: invalid.
- Empty signature for `OP_CHECKSIG`/`OP_CHECKSIGADD`: stack top remains unchanged or is consumed? **Empty sig = "false branch"**, no key/sig validation, no budget consumed. This is essential for K-of-N CHECKSIGADD chains.
- `OP_CHECKMULTISIG` is gone - replaced by chained `OP_CHECKSIGADD`.
- A signature with `SIGHASH_DEFAULT` (omitted byte) is 64 bytes; with explicit hash type, 65 bytes. The byte must be one of: 0x01 ALL, 0x02 NONE, 0x03 SINGLE, 0x81 ALL|ANYONECANPAY, 0x82 NONE|ANYONECANPAY, 0x83 SINGLE|ANYONECANPAY.

### Leaf versions and OP_SUCCESS

Tapscript leaves carry a 1-byte version tag. Currently defined: `0xc0` (with parity bit in low bit of control block first byte).

If during execution the interpreter encounters an `OP_SUCCESS` opcode (any in the set above), the script **succeeds immediately**, regardless of stack state - no signature verification needed. This is the soft-fork upgrade lane: a future BIP can redefine an OP_SUCCESS opcode to a real check; old nodes (still seeing OP_SUCCESS) accept the spend, new nodes verify properly.

### CODESEPARATOR

Legacy: when present, modifies the `scriptCode` covered by signing - useful but rarely correct.

Tapscript: scriptCode is no longer the concept (BIP341 sighash commits to `tapleaf_hash`). Instead, OP_CODESEPARATOR records its **opcode position** (`codeseparator_position`, 4 bytes) in the sighash. So a signature can selectively bind to "the script after the most recent CODESEPARATOR I've seen." Useful only with extreme care.

### Sigops budget worked

```
witness for input i = [<args> <leaf_script> <control_block>]
witness_weight  = sum of (1 + len(item)) for each item, length-prefixed varint counted
budget          = floor(witness_weight / 50)
each CHECKSIG / CHECKSIGADD: budget -= 50
if budget < 0: script fails
```

A 200-byte witness gives budget ~4, so at most 4 CHECKSIGs. A 1000-byte witness gives 20. This makes large script-path spends self-fund their verification cost.

## Worked example

3-of-5 Tapscript leaf, sorted public keys `P1..P5`:

```
script:
  20 <P1> ac           # PUSH32 P1, OP_CHECKSIG
  20 <P2> ba           # PUSH32 P2, OP_CHECKSIGADD
  20 <P3> ba
  20 <P4> ba
  20 <P5> ba
  53                   # OP_3
  9c                   # OP_NUMEQUAL
total leaf bytes: 5*(1+32+1) + 1 + 1 = 172

leaf_hash = TaggedHash("TapLeaf", 0xc0 || compact_size(172) || script)
```

Witness for spend by signers 1, 3, 5:

```
[<sig5>, <empty>, <sig3>, <empty>, <sig1>, <leaf_script>, <control_block>]

control_block = (0xc0 | parity) || <internal_pubkey> || <merkle_path...>
```

Note ordering: stack top is consumed first by `<P1> CHECKSIG`, so sig1 must be at the bottom of the stack of args. `OP_CHECKSIG` consumes `<P1>` and `<sig1>`, pushes 1. Then `<P2> CHECKSIGADD` pops `<P2> 1 <empty_sig>`, pushes `1 + 0 = 1`. And so on.

Sigops budget: 5 CHECKSIG-class ops -> 250 budget needed -> witness must be >= 12500 weight units. The 5-of-5 case here with 5 actual signatures (~64 each) easily clears it.

## Common bugs / anti-patterns

- Using `OP_CHECKMULTISIG` in a Tapscript leaf - it is OP_SUCCESS, anyone can drain the output.
- Sigs in wrong order: Tapscript requires sigs in stack order matching pubkey order in script (last pubkey first on stack).
- Missing the empty-sig placeholder for absent signers: stack underflow.
- Encoding `OP_TRUE`/`OP_FALSE` non-minimally: `OP_PUSHBYTES_1 0x01` instead of `OP_1` is fail under MINIMALDATA (always-on in Tapscript).
- Forgetting the leaf-version byte parity in the control block: validators reconstruct the wrong tweak and reject.
- Designing a script-path that needs more sigops than its own witness funds.
- Triggering an OP_SUCCESS during testing and not realising the script "works" - because OP_SUCCESS makes the spend always succeed; confused as a passing test.
- Using `OP_IF` without minimal `OP_TRUE`/`OP_FALSE` push: MINIMALIF fails.

## References

- BIP342 (Tapscript): https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki
- BIP341 (sighash + control block): https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
- Bitcoin Core `src/script/interpreter.cpp` - `EvalScript` with `SIGVERSION_TAPSCRIPT`
- Optech topic: https://bitcoinops.org/en/topics/tapscript/
