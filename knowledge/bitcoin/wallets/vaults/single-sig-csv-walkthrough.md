# Single-Sig CSV Vault Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/vaults`.
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/vaults/SKILL.md

## Concept

A single-sig CSV vault encodes the simplest meaningful vault property:
**every spend has a mandatory waiting period during which a recovery key
can intervene**. There is no quorum, no shamir, no third-party watchtower
required to express the policy — just two keys (hot and cold) and a
relative timelock.

It is the design every Bitcoin engineer should understand before moving on
to multisig or BIP345 OP_VAULT, because the same primitives (CSV, P2WSH or
Tapscript, alternate spending paths) compose into the more elaborate
patterns.

## Walkthrough / mechanics

The output script (P2WSH form):

```
OP_IF
   <DELAY> OP_CHECKSEQUENCEVERIFY OP_DROP
   <hot_pubkey> OP_CHECKSIG
OP_ELSE
   <cold_pubkey> OP_CHECKSIG
OP_ENDIF
```

The two spend paths:

| Path | Witness               | Sequence  | Triggers      |
|------|-----------------------|-----------|---------------|
| Hot  | `<hot_sig> 0x01 ...`  | `>= DELAY` | After waiting |
| Cold | `<cold_sig> 0x00 ...` | any        | Immediate    |

Encoding `DELAY` follows BIP68: low 16 bits = block count if bit 22 is
clear, otherwise low 16 bits × 512 seconds. The `tx.version` field must
be `>= 2` for relative locktimes to be enforced (BIP68 mandate).

For Taproot vaults the same logic lives in a tapleaf (single-key branch
unused, leaf path is the union of conditions). Tapscript replaces P2WSH:

```
# Taproot internal key = NUMS point (so key-path is unspendable)
# Taptree:
#   leaf 1: <DELAY> CSV DROP <hot_xonly> CHECKSIG
#   leaf 2: <cold_xonly> CHECKSIG
```

This shrinks the witness in the common (hot path) case from ~120 bytes
to ~108 bytes thanks to Schnorr 64-byte signatures.

## Worked example

Build a regtest vault with a 144-block hot delay (~1 day) using
`bitcoinutils` in Python:

```python
from bitcoinutils.script import Script
from bitcoinutils.keys import PrivateKey, P2wshAddress
from bitcoinutils.setup import setup
setup("regtest")

hot = PrivateKey()
cold = PrivateKey()
DELAY = 144

vault_script = Script([
    'OP_IF',
        DELAY, 'OP_CHECKSEQUENCEVERIFY', 'OP_DROP',
        hot.get_public_key().to_hex(), 'OP_CHECKSIG',
    'OP_ELSE',
        cold.get_public_key().to_hex(), 'OP_CHECKSIG',
    'OP_ENDIF',
])

vault_addr = P2wshAddress.from_script(vault_script)
print(vault_addr.to_string())   # bcrt1q...
```

Fund it on regtest:

```bash
bitcoin-cli -regtest sendtoaddress bcrt1q... 0.5
bitcoin-cli -regtest -generate 1
```

Hot-path spend (after waiting 144 blocks):

```python
from bitcoinutils.transactions import Transaction, TxInput, TxOutput

# Mine 144 blocks first
# bitcoin-cli -regtest -generate 144

txin = TxInput(funding_txid, 0, sequence=DELAY.to_bytes(4, 'little'))
txout = TxOutput(0.499 * 100_000_000, dest_script_pubkey)
tx = Transaction([txin], [txout], has_segwit=True)
tx.version = 2

sig_hot = hot.sign_segwit_input(tx, 0, vault_script, 0.5 * 100_000_000)
witness = [sig_hot, b'\x01', vault_script.to_hex()]
tx.witnesses.append(TxWitnessInput(witness))

bitcoin-cli -regtest sendrawtransaction tx.serialize()
```

The `sequence=DELAY` plus `tx.version=2` together enable BIP68 enforcement.
If you broadcast before mining 144 confirmations, the mempool rejects
with `non-BIP68-final`.

Cold-path emergency spend (no delay):

```python
txin = TxInput(funding_txid, 0)   # default sequence 0xffffffff is fine
txout = TxOutput(0.499 * 100_000_000, deep_cold_script_pubkey)
tx = Transaction([txin], [txout], has_segwit=True)
tx.version = 2

sig_cold = cold.sign_segwit_input(tx, 0, vault_script, 0.5 * 100_000_000)
witness = [sig_cold, b'', vault_script.to_hex()]   # b'' = OP_FALSE
tx.witnesses.append(TxWitnessInput(witness))
```

The empty bytestring `b''` is OP_FALSE, selecting the ELSE branch.

## Operational flow

```
DAY 0   - Cold key generated airgapped, hot key on phone.
        - Funds sent to vault address.
DAY 30  - User wants to spend 0.1 BTC.
        - Build tx with sequence=144 (1 day).
        - Pre-publish witness (some workflows actually broadcast
          immediately; the timelock just gates inclusion).
DAY 31  - Tx becomes valid; gets mined within minutes.

ATTACK SCENARIO:
DAY 30   - Phone is stolen. Attacker broadcasts a hot-path tx with
           sequence=144.
DAY 30+1 - Owner sees the broadcast on a watcher (mempool.space alert).
        - Owner signs cold-path tx draining funds to deep cold.
DAY 30+2 - Cold-path tx mines first (no delay). Hot-path is now
           double-spent. Attacker tx is invalid.
```

The defining property: the attacker's success requires owner to be offline
for the full DELAY window.

## Common pitfalls

- **Forgetting `tx.version = 2`**: relative locktimes silently disabled.
  The hot path's CSV check passes (because BIP68 isn't engaged) and the
  delay is a fiction.
- **Cold-path key on the same device as hot key**: the entire purpose
  of the vault evaporates. Use airgapped, ideally separate hardware.
- **DELAY too short**: at 1 block, you have ~10 minutes to react.
  Realistic minimums are 144 blocks (1 day) for individuals, 1 008
  (1 week) for high-value.
- **No watcher**: if no one monitors the chain for unauthorized
  unvault, you don't *know* to broadcast the cold-path tx in time.
  Run mempool.space alerts on the vault address, or a custom ZMQ
  listener.
- **Cold-path UTXO reuse**: spending repeatedly from the cold key on
  chain reveals its pubkey, weakening Taproot's privacy benefit.
- **Witness script confusion**: the `OP_IF`/`OP_ELSE` selector is the
  *last item pushed before the script* in segwit witness order. Bugs
  here cause unspendable UTXOs.

## References

- BIP65 (CLTV): https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki
- BIP68 (Sequence): https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki
- BIP112 (CSV): https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki
- See also: [revault-architecture-deep.md](revault-architecture-deep.md), [../timelocks/cltv-vs-csv-deep.md](../timelocks/cltv-vs-csv-deep.md)
