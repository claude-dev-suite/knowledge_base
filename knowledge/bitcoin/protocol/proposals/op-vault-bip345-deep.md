# OP_VAULT (BIP345) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/proposals`.
> Canonical source: BIP345
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/proposals/SKILL.md

## Concept

`OP_VAULT` and `OP_VAULT_RECOVER` are two new opcodes for first-class
vault primitives without complex script gymnastics. The skill summarizes
the two-stage flow; this article walks the opcode semantics, the
Tapscript encoding, the deferred-check mechanism that lets unvault txs
target arbitrary destinations, and the comparison to CTV-based vaults.

## Walkthrough / mechanics

**Two new opcodes (proposed for Tapscript):**

```
OP_VAULT          (uses an OP_SUCCESS-replaced opcode allocation)
OP_VAULT_RECOVER  (separate opcode)
```

**OP_VAULT script structure:**

```
<recovery-script-path-hash> <unvault-spk-template> <delay> OP_VAULT
```

When this script is the leaf of a Taproot output, the spender provides
a "trigger" tx that:
1. Spends the OP_VAULT input.
2. Creates an output with scriptPubKey matching `unvault-spk-template`
   (parameterized by the input's value and an additional hash).
3. Has a sequence number >= `delay`.

The opcode validates the spending tx structure matches and accepts.
Once confirmed, the output becomes a "trigger" output with its own
script that allows:
- Spending to the **target destination** (committed in the trigger
  output's script) after `delay` blocks.
- Spending via `OP_VAULT_RECOVER` to the recovery address at any
  time.

**OP_VAULT_RECOVER script:**

```
<recovery-spk> OP_VAULT_RECOVER
```

When triggered, the spending tx must:
1. Send all the output value to `recovery-spk`.
2. Have no other "OP_VAULT_RECOVER-protected" inputs that go elsewhere.

This lets the cold-key holder sweep ALL vaults to recovery in one tx
without per-vault signing overhead.

**Deferred-check mechanism:**

OP_VAULT uses "deferred validation": the opcode itself doesn't
fully validate the spending tx; it pushes a deferred check onto a
list. After all inputs are processed, Bitcoin Core reconciles the
deferred checks against actual outputs. This lets a single recover
tx sweep many vaults: each input pushes a deferred check, and the
combined output set is validated once.

## Worked example

**Vault setup (Taproot output with a single OP_VAULT leaf):**

```
internal_key   = P (Taproot internal key, can be NUMS for "no keypath spend")
recovery_path  = ScriptHash of "<recovery_pubkey> CHECKSIG"
unvault_template = <Q> for some intermediate output script
delay          = 144 blocks (1 day)

leaf_script = <recovery_path> <unvault_template> <144> OP_VAULT
taproot output: P2TR(NUMS, [leaf_script])
```

**Trigger / unvault flow:**

User wants to send to `dest_pubkey`:

```
trigger_tx:
  inputs: [vault_outpoint with script-path spending leaf_script]
  outputs:
    out0: full vault value to script:
        <144> OP_CSV DROP <dest_pubkey> OP_CHECKSIG   # delayed, locked to dest
        OR via OP_VAULT_RECOVER -> recovery_path
  witness: [<dest_commitment>, <leaf_script>, <control_block>]
```

The spending witness includes a hash committing to the destination,
which OP_VAULT validates is matched in the output's script.

After 144 confirmations, user signs:

```
final_tx:
  inputs: [trigger_outpoint]
  outputs: [dest_pubkey -> any address chosen now]
```

**Recovery flow:**

If user detects unauthorized unvault attempt during the 144-block
delay, cold-key holder broadcasts:

```
recover_tx:
  inputs: [trigger_outpoint]            # spends via recovery branch
  outputs: [recovery_address -> all funds]
  witness: [<recover_witness>, <recover_script>, <control_block>]
```

Or, sweeping multiple vaults:

```
recover_all_tx:
  inputs:
    [trigger_outpoint_1]
    [trigger_outpoint_2]
    [vault_outpoint_3 (still in OP_VAULT state)]
  outputs:
    [recovery_address -> sum of all input values]
```

The deferred-check mechanism validates that each input's recovery
path agrees with the single output destination.

## Common bugs / pitfalls

1. **Mistaking OP_VAULT for a covenant constraining destinations.**
   OP_VAULT does NOT lock the final destination on-chain. The
   trigger commits to a destination via a hash, but ANYTHING can be
   in the trigger output's destination commitment. This is by design
   - users want flexibility per-spend.
2. **No protection without monitoring.** Vaults work because the
   user MONITORS for trigger txs and races to recover during the
   delay. A user not watching the chain gains nothing over plain
   single-sig.
3. **Recovery key compromise = total compromise.** OP_VAULT_RECOVER
   uses a single key to bypass the delay. If that key is stolen,
   the attacker can recover-then-spend without delay. The recovery
   key must be cold and ideally multisig.
4. **Deferred-check ordering with mixed inputs.** A tx with OP_VAULT
   inputs and OP_VAULT_RECOVER inputs and normal inputs has multiple
   deferred check sets. Implementations must reconcile carefully;
   incorrect order produces false acceptance/rejection.
5. **OP_VAULT vs CTV for vaults.** CTV vaults pre-commit to specific
   destinations at unvault time; OP_VAULT defers this. CTV is rigid
   but requires only template hashing; OP_VAULT is flexible but
   requires more script logic and deferred validation.
6. **Tapscript-only.** OP_VAULT is proposed for Tapscript only; no
   legacy P2WSH version. Vaults must be P2TR.
7. **Activation pending.** Like CTV/APO, OP_VAULT is not active on
   mainnet. Test deployments exist on signet but mainnet status
   remains "proposed".

## References

- BIP345: https://github.com/bitcoin/bips/blob/master/bip-0345.mediawiki
- James O'Beirne's vault impl: https://github.com/jamesob/bitcoin/tree/2023-opvault
- Vaults design history: https://utxos.org/uses/vaults/
- Bitcoin OS: https://bitcoinos.tech/
- CTV-based vault comparison: https://utxos.org/blog/2022-bitcoin-vaults/
