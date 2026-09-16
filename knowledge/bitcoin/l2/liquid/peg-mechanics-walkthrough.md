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

Note the limit of that guarantee: the HSM checks a proposed block against the consensus
rules of the Elements build it is fed. A consensus bug therefore makes the conservation
check vacuous - the HSM sees a block that is valid under the (broken) rules and signs it.
That is exactly what happened in September 2026 (see the incident section below).

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

## Incident: the September 2026 range-proof cache exploit

On 6 September 2026, at Liquid block 4,050,336, an attacker exploited a cache-key
collision in Elements' range-proof verification cache. Range-proof verification is
expensive, so Elements memoises positive results; because a cache hit *is* a positive
verification result, a key collision bypasses verification outright.

Two distinct defects in that key are visible in the Elements history:

- Releases up to and including elements-23.3.3 (13 April 2026) computed the key as
  `salted_hash(proof || commitment)` only - the asset generator and the scriptPubKey
  were not committed to at all.
- The commit that bound those in (`212c43f475`, landed on the `elements-23.3.x` branch
  on 3 September 2026) concatenated `proof || commitment || asset_commitment ||
  scriptPubKey` with no length prefixes, so bytes can be shifted across the boundary
  between the two variable-length fields to make distinct tuples hash to the same key.

Public analyses differ on which of the two the attacker actually used; elements-23.3.4
fixes both. The outcome is not in dispute, as of 15 September 2026:

- ~3,998.5 unbacked L-BTC were minted and pegged out for ~4,000 BTC (~$320M). The
  federation's Bitcoin reserve fell from ~4,200 BTC to ~197 BTC (~95%).
- **No key was compromised.** Blockstream's status page records that the funds moved via
  the SideSwap peg-out authorisation key, "but that key was not compromised, nor were any
  others". The HSMs signed the fraudulent peg-out because it was valid under the
  consensus rules they were running.
- Reports describe a node split - some Liquid nodes accepted the attack block, others
  rejected it - i.e. a consensus failure, not a custody failure.
- Blockstream disabled bridge nodes on 7 September 2026 (00:14 UTC), so no new
  transactions could be submitted.
- The attackers returned ~3,400 BTC on 7 September and kept 598.5 BTC (~$47M) as a
  self-declared bounty. Blockstream refused to pay it ("We will not pay for the return
  of stolen property", 11 September 2026).
- Block production resumed transaction-free for monitoring by 10 September 2026 (status
  page, as of 10:00 UTC); peg operations, including PAK-authorised peg-outs, remained
  suspended while the reserve was restored.

**Minimum safe version: elements-23.3.4** (released 9 September 2026). PR #1600,
"sigcache: harden range proof cache keys and add -norangeproofcache option", switches the
range-proof and surjection-proof cache hashers from raw `CSHA256` concatenation to
`CHashWriter` (`SER_GETHASH`), which length-prefixes every serialised field; it also adds
`vTags` to the surjection-proof key, which had been missing from it entirely. The same PR
adds a `-norangeproofcache` startup option so the range-proof cache can be disabled
without recompiling.

## Trade-offs and security

- **Federation trust**: 11-of-15 is the fundamental assumption. A coordinated 11-party
  compromise (legal seizure, hardware exploit, insider collusion) drains the peg. That is
  not the only way to lose the peg, and it is not how it was lost in September 2026: a
  consensus-rule bug in Elements got eleven honest functionaries to sign a fraudulent
  peg-out with no key compromise at all. Validation-code correctness is as load-bearing
  here as key custody.
- **No on-chain enforcement**: unlike BitVM2, Liquid's peg is *not* trust-minimised;
  there is no challenge protocol that lets outsiders punish federation misbehaviour.
  September 2026 showed the cost: once the federation had signed the inflationary
  peg-out, nothing on Bitcoin could reverse it, and recovery depended on the attackers
  choosing to send most of the funds back.
- **Retail peg-out friction**: whitelisting effectively means retail users go through
  exchanges, which is a regulatory feature for institutional Liquid users but a UX loss
  for sovereign users.
- **102-confirmation peg-in delay**: ~17 hours is much longer than Lightning or Spark
  but is required for Bitcoin reorg safety.
- **HSM dependency**: bug or backdoor in the Liquid HSM is a systemic risk. The HSM is
  closed-source firmware; Blockstream audits and rotates. The deeper dependency is the
  consensus code the HSM validates against: in September 2026 the HSMs behaved exactly as
  specified and still signed away ~95% of the reserve.
- **Block-signer liveness**: if 5+ functionaries are simultaneously offline, Liquid
  halts. Before 2026 this had happened twice in production for short windows (hours).
  The September 2026 halt was of a different order: bridge nodes were disabled on
  7 September, block production only resumed (transaction-free) on 10 September, and peg
  operations stayed suspended beyond that - days, not hours.
- **Compared to RSK merge mining**: RSK inherits Bitcoin PoW security but has a slower
  peg via federation+merge-mining. Liquid is faster (1-min blocks) but more centralised.

## References

- Liquid technical paper - https://blockstream.com/strong-federations.pdf
- Liquid docs - https://docs.liquid.net/docs/technical-overview
- Elements Project - https://github.com/ElementsProject/elements
- elements-23.3.4 release (9 Sept 2026) - https://github.com/ElementsProject/elements/releases/tag/elements-23.3.4
- Elements PR #1600, cache-key hardening - https://github.com/ElementsProject/elements/pull/1600
- Blockstream status, Liquid incident (Sept 2026) - https://status.blockstream.com/incidents/b8b719f3-db70-4487-9cff-946e69509228
- Liquid Network incident report (8 Sept 2026) - https://x.com/Liquid_BTC/status/2097404704028545175
- SlowMist analysis of the cache-key collision (11 Sept 2026) - https://cryptobriefing.com/liquid-network-exploit-slowmist-analysis/
- Reserve drain and partial return, figures (Sept 2026) - https://crypto.news/liquid-network-320-million-drain-cache-bug-unbacked-bitcoin/
- Blockstream refuses the retained-bounty demand (11 Sept 2026) - https://www.theblock.co/news/ecosystems/2026-09-11-return-the-bitcoin-blockstream-refuses-ransom-demand-for-remaining-600-btc-from-liquid-exploit-414247
- "Strong Federations" - Dickson et al. (2016)
