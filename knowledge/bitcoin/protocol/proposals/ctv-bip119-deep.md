# CTV (BIP119) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/proposals`.
> Canonical source: BIP119
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/proposals/SKILL.md

## Concept

`OP_CHECKTEMPLATEVERIFY` (CTV) is a covenant opcode that compares a
hash on the stack to a "template hash" computed from the spending
transaction's structure. If they match, validation passes; otherwise
fail. The template commits to OUTPUTS, locktimes, sequences, and
input count - but NOT specific input prevouts. This makes it possible
to pre-commit to a payment graph before knowing which UTXO will fund
it.

The skill summarizes CTV's purpose; this article walks the template
hash computation, the deployment plan and its political history, and
the practical patterns CTV enables (vaults, congestion control, simple
Ark variants).

## Walkthrough / mechanics

**Opcode encoding (proposed):**

CTV repurposes opcode `OP_NOP4` (0xb3). Pre-activation, it is a no-op
that keeps the witness valid. Post-activation, it pops the top stack
item (must be a 32-byte hash) and verifies it equals the
StandardTemplateHash of the current spending transaction.

**StandardTemplateHash computation:**

```
H = SHA256(
    nVersion              (4 bytes LE)
    nLockTime             (4 bytes LE)
    [optional] sha256(ser(scriptSigs)) IF scriptSig is non-empty for ANY input
    nInputCount           (4 bytes LE)
    sha256(ser(sequences))
    nOutputCount          (4 bytes LE)
    sha256(ser(outputs))
    nIn                   (4 bytes LE)   # input index of THIS input
)
```

`outputs` are serialized as concatenated `(value || scriptPubKey)`
pairs, then SHA256-summed. Each output is fully committed.

`nIn` is the position of the currently-evaluated input - so different
inputs of the same tx commit to different template hashes. This lets
multi-input txs differentiate their constraints per input.

Notably absent: prevouts (txid:vout). The funder doesn't need to know
in advance which UTXO will satisfy CTV, only that the eventual tx
matches the structure.

## Worked example

**Vault construction:**

```
unvault output (locked):
  scriptPubKey = <H_unvault> OP_CTV
  value = 1 BTC

H_unvault commits to a tx that:
  - has 1 input (whatever it is)
  - has 2 outputs:
    - output 0: 1.0 BTC to <H_cooldown> OP_CTV (cooldown locked)
    - output 1: 0 sat OP_RETURN (anchor for fee bump)
  - locktime = 0
  - sequence = 0xffffffff
```

When user wants to spend:

```
unvault_tx:
  inputs:  [unvault_outpoint]                  # the 1 BTC
  outputs:
    out0: 1.0 BTC to (P2WSH wrapping the cooldown CTV)
    out1: 0 sat OP_RETURN
```

Compute StandardTemplateHash of `unvault_tx` -> matches `<H_unvault>`,
so OP_CTV passes. Tx confirms.

After cooldown:

```
cooldown_tx:
  inputs:  [unvault_outpoint with sequence >= cooldown_blocks]
  outputs:
    out0: 1.0 BTC to user_destination_address
```

Cooldown spends use a relative timelock (`OP_CSV`) plus another CTV
template, or just `<pubkey> CHECKSIG <cooldown> CSV`.

If user detects theft attempt during cooldown:

```
recover_tx:
  inputs:  [unvault_outpoint]
  outputs: [cold_storage_address]
  scriptSig: signature against <hot> CHECKSIG path of inner script
```

The cold-storage path doesn't need to wait for CTV cooldown - it's
the recovery branch.

**Congestion control:**

A merchant with 1000 customers awaiting payment:

```
funding_tx:
  outputs:
    out0: 1000 * avg_payment to <H_payouts> OP_CTV
```

`H_payouts` commits to a tx producing 1000 outputs (one per customer).
The funding tx confirms cheaply when fees are low; the actual fan-out
to 1000 outputs happens later, when fees are higher, OR can be
deferred indefinitely.

**Simple Ark variant:**

A coordinator pre-builds a tree of CTV templates representing a
2^N-leaf payment tree. The root output commits to the next level via
CTV; participants spend down the tree only when they need to, with
miners only seeing the path actually taken.

## Common bugs / pitfalls

1. **Forgetting nIn in template hash.** Each input has its OWN
   template hash because `nIn` differs. A two-input tx where both
   inputs use the SAME template hash on stack would fail (one of the
   two has a different commit).
2. **Computing template hash before signing other inputs.** In the
   funding direction, you build the template-hash from the OUTPUT
   structure of the SPENDING tx. The spending tx's inputs and
   sequences are part of the hash; if you change inputs later, the
   hash changes.
3. **Pre-activation use is unreachable.** CTV only activates after a
   soft fork. Until then, OP_NOP4 is a no-op; using `<hash> OP_CTV`
   in a script today produces an output anyone can spend after
   trivially providing a 32-byte item on the stack.
4. **Mixing CTV and Tapscript paths.** CTV in Tapscript would use a
   different opcode allocation (in script-path it's `OP_NOP4` still
   but the validation hooks differ from legacy). Current proposal
   focuses on legacy Script first.
5. **scriptSig commitment.** The template hash includes
   `sha256(ser(scriptSigs))` only IF any input has non-empty
   scriptSig. SegWit inputs have empty scriptSig, so this hash is
   often empty. P2SH inputs require committing to the redeem-script
   push.
6. **Witness commitment.** Witness data is NOT in the template hash.
   So sigs and witness_scripts can vary between the template's
   "committed-to" tx and the actual confirmed tx. Funder doesn't
   need to predict witness sizes (good for fee accounting).

## References

- BIP119: https://github.com/bitcoin/bips/blob/master/bip-0119.mediawiki
- Reference implementation in Bitcoin Core (PR): https://github.com/bitcoin/bitcoin/pull/21702
- Jeremy Rubin's CTV site: https://utxos.org/
- CTV signet: https://signet.utxos.org/
- OP_VAULT (similar territory): BIP345
