# Liquid Peg Mechanics Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/liquid`.
> Canonical source: https://docs.liquid.net/docs/technical-overview
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/liquid/SKILL.md

## Concept

Liquid's peg is a federated 11-of-15 multisig held by ~15 functionaries (Blockstream and
exchange partners). Peg-in (BTC -> L-BTC) is permissionless; peg-out (L-BTC -> BTC) is
whitelisted and runs in queued batches. The peg is the trust foundation of Liquid: if
11 functionaries collude or leak keys, all L-BTC is at risk. The federation also produces
Liquid blocks via a round-robin signer schedule using Strong Federations consensus.

## Walkthrough / mechanics

### Peg-in (BTC -> L-BTC)

```
USER on Bitcoin                      FEDERATION                     USER on Liquid
   |                                     |                                |
   | 1. Generate Liquid receiving        |                                |
   |    address.                         |                                |
   |                                     |                                |
   | 2. Compute claim script             |                                |
   |    (P2WSH-wrapped peg-in script).   |                                |
   |                                     |                                |
   | 3. Send BTC to deposit address      |                                |
   |    derived from claim script +      |                                |
   |    federation pubkeys.              |                                |
   |                                     |                                |
   |  ---- 102 confirmations ---->                                        |
   |                                     |                                |
   | 4. Submit "claim tx" on Liquid      |                                |
   |    referencing the BTC tx +         |                                |
   |    Merkle proof.                    |                                |
   |                                     |  5. Functionaries verify:     |
   |                                     |     - 102 confs                |
   |                                     |     - claim script matches    |
   |                                     |     - amount conservation     |
   |                                     |  6. L-BTC issued to recipient |
   |                                     |     on Liquid in next block.  |
   |                                     |                                |
   |                                     |                                |--- L-BTC arrives
```

The 102-confirmation rule provides Bitcoin-side reorg safety; smaller would risk a Liquid
issuance for a BTC tx that later disappears.

### Peg-out (L-BTC -> BTC)

Peg-out is **gated**:
- The Liquid Federation maintains a whitelist of known peg-out addresses (typically
  exchange withdrawal addresses).
- Retail users effectively peg out by depositing L-BTC at an exchange and withdrawing
  BTC; the exchange sits on the whitelist.

Mechanically:

1. User submits a peg-out tx on Liquid sending L-BTC to a special peg-out address with
   the destination BTC scriptPubKey embedded.
2. Functionaries verify the BTC scriptPubKey is on the whitelist.
3. After confirmations on Liquid (typically 2-3 blocks ~ 4-6 minutes), functionaries
   batch the peg-out into a federated BTC withdrawal tx.
4. 11 of 15 functionaries sign; tx broadcast on Bitcoin.
5. User receives BTC at the destination address.

### Block production

15 functionaries sign blocks in a round-robin schedule. A block requires 11 signatures.
Liquid uses Strong Federations: a fixed signer set with no PoW; safety holds while < 5
of 15 are byzantine (since 11/15 must sign). Block time is 1 minute (target).

### HSM-backed signing

Functionaries operate **purpose-built HSMs** ("Liquid HSM"):
- Tamper-evident hardware enforcing per-block signing rules.
- HSM refuses to sign if the proposed block re-spends its predecessor or violates
  L-BTC conservation.
- HSM also handles peg-out signing; refuses to sign peg-outs not from whitelisted
  addresses.

This means a compromised functionary OS cannot easily exfiltrate keys or sign rogue
peg-outs without physical HSM defeat.

## Worked example

Bitfinex withdraws 5 L-BTC to a whitelisted BTC address:

```
1. Bitfinex creates a Liquid tx:
     OUT 1: 5 L-BTC -> peg-out script committing to BTC address bc1qxxx
     OUT 2: change to Bitfinex hot wallet
     OUT 3: explicit fee
2. Tx confirms in Liquid block N.
3. Functionaries detect peg-out, batch with other peg-outs in queue.
4. After Liquid block N+2, batched BTC withdrawal tx assembled:
     IN: federation multisig UTXO
     OUT 1: 5 BTC -> bc1qxxx (Bitfinex)
     OUT 2: 12 BTC -> bc1qyyy (other exchange whitelisted addr)
     OUT 3: federation multisig change
5. 11 functionaries' HSMs sign; tx broadcast.
6. After 1-2 BTC confs, Bitfinex sees withdrawal arrived.
```

End-to-end peg-out latency: ~15-30 minutes typical.

## Trade-offs and security

- **Federation trust**: 11-of-15 is the fundamental assumption. A coordinated 11-party
  compromise (legal seizure, hardware exploit, insider collusion) drains the peg.
- **No on-chain enforcement**: unlike BitVM2, Liquid's peg is *not* trust-minimised;
  there is no challenge protocol that lets outsiders punish federation misbehaviour.
- **Retail peg-out friction**: whitelisting effectively means retail users go through
  exchanges, which is a regulatory feature for institutional Liquid users but a UX loss
  for sovereign users.
- **102-confirmation peg-in delay**: ~17 hours is much longer than Lightning or Spark
  but is required for Bitcoin reorg safety.
- **HSM dependency**: bug or backdoor in the Liquid HSM is a systemic risk. The HSM is
  closed-source firmware; Blockstream audits and rotates.
- **Block-signer liveness**: if 5+ functionaries are simultaneously offline, Liquid
  halts. This has happened twice in production for short windows (hours).
- **Compared to RSK merge mining**: RSK inherits Bitcoin PoW security but has a slower
  peg via federation+merge-mining. Liquid is faster (1-min blocks) but more centralised.

## References

- Liquid technical paper - https://blockstream.com/strong-federations.pdf
- Liquid docs - https://docs.liquid.net/docs/technical-overview
- Elements Project - https://github.com/ElementsProject/elements
- "Strong Federations" - Dickson et al. (2016)
