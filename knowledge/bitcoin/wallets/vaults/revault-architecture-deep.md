# Revault Architecture - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/wallets/vaults`.
> Canonical source: Revault paper (Cohen, 2020) https://github.com/revault/practical-revault/blob/master/revault.pdf
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/wallets/vaults/SKILL.md

## Concept

Revault is a multi-party vault protocol designed for **organisational
custody**. Where a single-sig CSV vault optimises for one user, Revault
splits roles across stakeholders, managers, cosigners, and watchtowers,
producing pre-signed transaction sets that bind the institution's policy
into Bitcoin script. The key insight is that *all* possible spending
transactions are pre-signed at deposit time; runtime decisions only
choose which pre-signed transaction to broadcast.

Revault was deprecated as an active product in 2022, but the architecture
is still studied because it remains the most thorough analysis of
multi-party Bitcoin custody and most modern proposals (Liana, Cake
multisig vaults) are simplifications of it.

## Roles

| Role | Holds | Online expectations |
|------|-------|---------------------|
| Stakeholder   | Long-term cold key                  | Online only at deposit + emergency |
| Manager       | Short-term operational key          | Daily |
| Cosigner      | Anti-replay key (no policy check)  | Real-time signing service |
| Watchtower    | Pre-signed Cancel + Emergency txs  | 24/7 monitoring |

A typical setup has e.g., 3 stakeholders, 2 managers, 1 cosigner per
manager, and 1 watchtower per stakeholder.

## Pre-signed transaction set

For each deposit UTXO, the participants pre-sign:

```
Vault tx:    user → Vault P2WSH(N-of-N stakeholder multisig)

Unvault tx:  Vault → Unvault P2WSH(
                   IF: cooldown=144 + (M-of-M managers + cosigner)
                   ELSE: N-of-N stakeholders )
              [pre-signed by all stakeholders + cosigner upfront]

Spend tx:    Unvault (cooldown branch) → recipient(s)
              [pre-signed by managers; broadcast after CSV expires]

Cancel tx:   Unvault (immediate stakeholder branch) → fresh Vault
              [pre-signed by all stakeholders; broadcast if Unvault is unauthorized]

Emergency tx: Vault → deep-cold P2WSH
              [pre-signed by all stakeholders; nuclear option]

Emergency Unvault tx: Unvault → deep-cold P2WSH
              [pre-signed by all stakeholders; for in-flight unvaults]
```

The architecture's invariant: at any moment, every UTXO has a complete
set of "what can happen next" transactions already signed and stored
encrypted at watchtowers. No live signing key is needed for emergency
response.

## Walkthrough / mechanics

### Deposit ceremony

1. Stakeholders generate Vault address (N-of-N taproot or P2WSH multisig).
2. User deposits to Vault address.
3. After 6 confirmations, all stakeholders sign Cancel, Emergency, and
   Unvault-emergency transactions. The Spend transaction is pre-signed
   only by managers (who can sign new Spend txs at runtime).
4. Signed transactions are encrypted and shipped to watchtowers.

### Spend ceremony

1. Manager builds an Unvault tx (already pre-signed by stakeholders).
2. Manager + cosigner co-sign the Spend tx (Unvault → recipient).
3. Manager broadcasts Unvault tx.
4. Watchtowers see Unvault on-chain. They check it against the
   policy database:
   - Recipient on whitelist?
   - Manager-cosigner pair authorised?
   - Spend tx hash matches an authorised one?
5. If checks fail, watchtowers broadcast Cancel within the 144-block
   window, redirecting funds to a fresh Vault.
6. If checks pass, watchtowers do nothing. After cooldown, the Spend
   tx becomes valid and is broadcast by the manager.

### Emergency

If a stakeholder's monitoring detects systemic compromise (e.g.,
manager keys leaked), they broadcast the pre-signed Emergency tx.
This drains all current Vault UTXOs to deep cold, requiring a full
ceremony to recover.

## Why pre-signed?

Pre-signing relies on **input non-malleability**. Before SegWit, a tx
could be malleated post-signing (third-party reorders witness data),
invalidating descendants. SegWit (and later Taproot) made input txids
deterministic regardless of witness data, so pre-signed descendants
always reference the right parent.

This is why Revault requires SegWit deposits. A legacy P2SH deposit
could be malleated and the entire pre-signed tree would be invalidated.

## Worked example: building Cancel tx flow (sketch)

```python
# At deposit time, stakeholders compute:
unvault_psbt = build_unvault(vault_outpoint)
for s in stakeholders:
    s.sign(unvault_psbt)

cancel_psbt = build_cancel(unvault_outpoint, fresh_vault_addr)
for s in stakeholders:
    s.sign(cancel_psbt)

# Encrypt and ship to watchtowers
for w in watchtowers:
    w.store(deposit_id, encrypt(unvault_psbt + cancel_psbt + emergency_psbt))

# At runtime, manager wants to spend:
spend_psbt = build_spend(unvault_outpoint, recipient, amount)
manager.sign(spend_psbt)
cosigner.sign(spend_psbt)   # cosigner check: have I signed this Spend before?

# Manager broadcasts Unvault
broadcast(finalize(unvault_psbt))

# Watchtower observes; if policy fails:
broadcast(finalize(cancel_psbt))
```

## Common pitfalls

- **No cosigner anti-replay**: without a cosigner that refuses to sign the
  same Spend twice, a manager can pre-sign many Spends, broadcast Unvault,
  then race the cooldown picking whichever Spend they want. The cosigner
  ensures one Unvault → one Spend.
- **Watchtower censorship**: a watchtower colluding with a malicious
  manager could refuse to broadcast Cancel. Mitigation: N watchtowers
  per stakeholder, any-of-N can broadcast.
- **Encrypted blob theft is harmless**: the watchtower's encrypted PSBTs
  are only useful with the decryption key (held by stakeholders). Stolen
  blobs cannot move funds.
- **Reorg through the cooldown**: a 144-block reorg can invalidate the
  Cancel race. Use longer cooldowns for very high value.
- **Pre-signed feerate is fixed**: in a fee spike, your Cancel tx may
  not confirm in time. Mitigations: use anchor outputs on Cancel, or
  pre-sign multiple Cancel variants at different fee rates.
- **Key compromise is catastrophic**: if N-of-N stakeholders is the
  Emergency policy and one stakeholder loses keys permanently, the
  Emergency path is broken.

## References

- Revault paper: https://github.com/revault/practical-revault/blob/master/revault.pdf
- Liana wallet (modern simplification): https://wizardsardine.com/liana/
- See also: [single-sig-csv-walkthrough.md](single-sig-csv-walkthrough.md), [../timelocks/cltv-vs-csv-deep.md](../timelocks/cltv-vs-csv-deep.md)
