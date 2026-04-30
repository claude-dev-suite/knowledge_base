# Ark Round Mechanics Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/ark`.
> Canonical source: https://arkdev.info/docs/learn/intro and ARKADE protocol notes
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/ark/SKILL.md

## Concept

A "round" is the atomic unit of Ark settlement. Inside a round, users submit payment
intents, the ASP computes a balanced VTXO tree, all parties cosign every branch, the ASP
broadcasts the pool tx, and forfeit txs neutralize old VTXOs. A round either completes
fully (all intents settled) or aborts (no on-chain footprint, users retry next round).

## Walkthrough / mechanics

```
TIME ->  T0          T0+regWindow    T0+nonceWindow    T0+sigWindow    T0+broadcast
         |               |                 |                 |               |
         | Registration  |  Nonce share    |  Partial sigs   |  ASP signs    |
         | (intents)     |  (MuSig2/FROST) |  collected      |  pool tx      |
         |               |                 |                 |               |
USER     | submit intent | publish nonce   | partial sig     | receive proofs|
ASP      | aggregate     | merge nonces    | aggregate sigs  | broadcast     |
                                                                + forfeit gather
```

ARKADE targets ~5s round time on signet/mainnet beta. The reference `bark` uses longer
rounds for testing.

### Phase 1 - Registration

Each client opens a websocket to the ASP and POSTs an Intent:

```json
{
  "inputs":  [ {"vtxo_id": "...", "amount": 10000} ],
  "outputs": [ {"recipient": "ark1...", "amount": 7000},
               {"recipient": "ark1...", "amount": 3000} ],
  "boarding_inputs": [ {"txid": "...", "vout": 0} ]   // optional fresh BTC
}
```

`boarding_inputs` are fresh on-chain UTXOs the user is converting into VTXOs ("boarding"
the round). The ASP charges a small fee on boarding.

### Phase 2 - Tree planning and nonce ceremony

Once registration closes, the ASP:

1. Builds the balanced binary tree minimising depth.
2. Allocates outputs at leaves.
3. Computes the set of pre-signed branch txs.
4. Sends each cosigner the list of branch txs they participate in plus a per-tx unsigned
   sighash.
5. Each user generates and broadcasts MuSig2 nonces for their cosigning leaves.

### Phase 3 - Partial signing

Users sign each branch tx they participate in (typically just their direct ancestors in
the tree). They also sign a **forfeit transaction** for every VTXO they brought as input.
The forfeit tx is a pre-signed authorization that allows the ASP to spend the user's
*old* VTXO if they ever try to redeem it on-chain after the new round confirms.

### Phase 4 - ASP aggregation and broadcast

ASP aggregates partial signatures, validates the tree, signs the pool tx, and broadcasts.
The pool tx includes the round's commitment as the Taproot output script tree.

After confirmation, all old VTXOs are now redundant; if a malicious user tries to exit
their *old* VTXO, the ASP broadcasts the forfeit and reclaims it.

## Worked example

Three intents in one round:

```
Intent 1: Alice spends 10000 sat VTXO -> 7000 to Bob, 3000 change to Alice
Intent 2: Carol boards 5000 sat from on-chain -> 5000 VTXO to Carol
Intent 3: Dave spends 8000 sat VTXO -> 8000 to Eve

Tree leaves built by ASP:
   Bob 7000 | Alice 3000 | Carol 5000 | Eve 8000

Pool tx inputs:
   - ASP funding UTXO (covers temporary liquidity)
   - Carol's 5000 sat boarding UTXO
   - (Alice/Dave's old VTXOs do NOT appear; they are off-chain forfeits)

Pool tx outputs:
   - Tree root Taproot output worth 23000 - fees
   - ASP fee output

After confirm:
   - Bob has a fresh VTXO leaf in the new tree.
   - Alice has 3000 sat change VTXO.
   - Carol has 5000 sat VTXO.
   - Eve has 8000 sat VTXO.
   - Old VTXOs of Alice and Dave are forfeited; if either tries to exit
     the prior round's branch, ASP claims it via the forfeit tx.
```

## Trade-offs and security

- **Atomicity**: a single non-responsive participant can stall the cosigning ceremony.
  ARKADE uses short timeouts and re-runs the round without the dropout.
- **Forfeit timing race**: the forfeit must be broadcastable *strictly before* the
  unilateral-exit branch becomes spendable. The ASP enforces this by setting the new
  round's CSV well below the old round's exit timelock.
- **MEV / ordering**: ASP sees all intents before tree construction, allowing potential
  ordering manipulation. Mitigations are an open research area (e.g., PBS-style separated
  builders/operators).
- **Boarding fees**: amortized across the pool tx vbyte budget; cheaper at high N.
- **Privacy**: the pool tx is a single anonymity set per round; observers learn round
  size but not individual transfers.

## References

- ARKADE protocol overview - https://arkdev.info/docs/protocol-spec/round-flow
- Burak, original Ark mailing-list post, 2023
- Ark Labs blog "Round Mechanics" series
- bark reference implementation - https://codeberg.org/ark-network/bark
