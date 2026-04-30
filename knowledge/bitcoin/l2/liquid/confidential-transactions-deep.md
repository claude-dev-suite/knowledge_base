# Liquid Confidential Transactions - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/liquid`.
> Canonical source: https://elementsproject.org/features/confidential-transactions
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/liquid/SKILL.md

## Concept

Liquid's Confidential Transactions (CT) hide both the **amount** and the **asset id** of
each output while preserving public verifiability that no inflation occurred and that all
inputs are accounted for. CT was originally proposed by Adam Back and Greg Maxwell for
Bitcoin and shipped on Liquid in 2018; Liquid extended it with **Confidential Assets** so
the same machinery hides which of L-BTC, L-USDt, or any user-issued token a UTXO holds.

## Walkthrough / mechanics

### Pedersen commitments

Each output amount `v` is committed as:

```
C = v*H + r*G
```

where `G, H` are independent generators and `r` is a blinding factor known to sender
and receiver. Crucially, commitments are *additive*:

```
sum(C_outputs) - sum(C_inputs) - C_fee = 0   <=>  sum(v_outputs) = sum(v_inputs) - fee
```

The chain verifies this equation directly on the EC point sum, so balance is enforced
without seeing values.

### Range proofs

Naive Pedersen lets anyone "commit" to a negative amount that wraps mod n. CT therefore
attaches a **range proof** to each output proving `0 <= v < 2^52` (Liquid uses 52 bits
to leave headroom under the EC group order).

Liquid uses **Bulletproofs** (since 2018) for range proofs, ~700 bytes per output --
much smaller than original Borromean ring sigs. Verification cost is logarithmic in
range size.

### Confidential Assets

Liquid extends CT to hide **which** asset a UTXO holds. Each output has both:

- An amount commitment `Cv = v*H_a + r*G` where `H_a` is the asset-specific generator.
- An asset commitment `Ca = H_a + s*G` where `s` is an asset-blinding factor.

Surjection proofs ensure that each output's asset commitment matches one of the input
asset commitments, proving conservation per asset without revealing which.

### Receiver-side decryption

The sender encrypts `(v, asset_id, r, s)` to the receiver's blinding key (derived from
the receiver's confidential address). Receiver decrypts to learn the amount and asset.

A "view key" can be exported to give a third party (auditor, exchange) read access
without revealing spending control.

```
Liquid confidential address format:
  el1qq <55 bytes blinding pubkey> <bech32m receiving payload>

Receiver hands this to sender; sender uses pubkey to ECDH-derive blinding factors.
```

## Worked example

Alice sends Bob 1000 L-USDt and gets 50 L-BTC change in a single tx:

```
Inputs:
  IN1:  amount commitment Cv_in1, asset commitment Ca_in1   (1500 L-USDt)
  IN2:  amount commitment Cv_in2, asset commitment Ca_in2   (60 L-BTC)

Outputs:
  OUT1 (Bob, 1000 L-USDt): Cv_out1, Ca_out1, range proof, surjection proof
  OUT2 (Alice change 500 L-USDt): Cv_out2, Ca_out2, range proof, surjection proof
  OUT3 (Alice change 50 L-BTC):   Cv_out3, Ca_out3, range proof, surjection proof
  FEE:  explicit fee 10000 sats L-BTC (always public on Liquid)

Verifier checks:
  sum_outputs(L-USDt) commitments == IN1 commitment   (no inflation in USDt)
  sum_outputs(L-BTC) commitments + fee == IN2 commitment   (no inflation in L-BTC)
  All range proofs valid.
  All surjection proofs valid (each output asset commits to one of {IN1.asset, IN2.asset}).

Public observer sees:
  - 2 inputs, 3 outputs + fee
  - fee amount in L-BTC
  - addresses (blinded)
  Does NOT see: which output is L-USDt vs L-BTC, individual amounts.

Bob sees (via decryption with his blinding key): output 1 is 1000 L-USDt, asset id Tether.
```

## Trade-offs and security

- **Computational overhead**: ~6 KB per CT output in worst case (range + surjection
  proofs). Liquid blocks are larger than equivalent BTC blocks for the same tx count.
- **Verification cost**: Bulletproofs verify in tens of ms per output; full nodes feel it
  on initial sync but it is amortised per block.
- **Compromise of blinding factor**: anyone who learns `r` for an output learns the
  amount. View keys exist precisely to share this controlledly.
- **Auditability**: Liquid Federation members hold view keys for their own services;
  regulators can require reporting via per-business view-key disclosure rather than chain
  surveillance.
- **No covenants on commitments**: a covenant script that wants to enforce "amount < X"
  cannot read `v` directly; CT and OP_CTV-style covenants are largely incompatible without
  expensive zero-knowledge add-ons.
- **Privacy is per-tx, not per-graph**: a confidential tx hides amounts but on-chain
  graph analysis (which UTXO spends which) is still public. Combined with peg-out
  whitelisting, this means CT in practice obscures amounts more than counterparties.

## References

- Greg Maxwell, "Confidential Transactions" investigation page (2015)
- Andrew Poelstra et al., "Confidential Assets" (2017) - https://blockstream.com/bitcoin17-final41.pdf
- Bulletproofs - https://web.stanford.edu/~buenz/pubs/bulletproofs.pdf
- Elements Project - https://elementsproject.org/features/confidential-transactions
