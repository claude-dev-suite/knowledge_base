# FROST Signing in Spark - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/spark`.
> Canonical source: FROST RFC draft + https://docs.spark.money/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/spark/SKILL.md

## Concept

FROST (Flexible Round-Optimised Schnorr Threshold) is the threshold-signature scheme
underpinning Spark Operator (SO) cosigning. Compared to MuSig2 (used by ARKADE), FROST
supports k-of-n thresholds with k < n, robust signing under non-responsive participants,
and pre-processing that lets a single user-driven request be signed in one online round.
The output is a normal BIP-340 Schnorr signature -- on-chain, a Spark cooperative spend
is indistinguishable from a single-key Taproot key-path spend.

## Walkthrough / mechanics

### Key generation (DKG)

```
SOs run a Distributed Key Generation:
  1. Each SO_i picks a polynomial f_i of degree k-1.
  2. Broadcast commitments to f_i coefficients.
  3. Send each SO_j the share f_i(j) over an encrypted channel.
  4. Each SO_j sums received shares -> their long-term key share s_j.
  5. Public group key Y = sum_i (f_i(0) * G).
```

After DKG, each SO holds a Shamir share of the group secret. No party ever learns the
full secret; the group can produce signatures with k cooperating shares.

### Two-round signing (online phase)

For each transfer the SOs run:

**Round 1 - Commit nonces**

Each participating SO picks two random nonces (d_i, e_i), publishes commitments
(D_i = d_i*G, E_i = e_i*G). Spark pre-batches nonce commitments so most user-facing
transfers complete in *one* online round on the user's perceived latency.

**Round 2 - Sign**

SOs receive the message m (sighash of the leaf-respend tx) and the binding factor
rho_i = H(i, m, B) where B is the set of all (D_j, E_j). Each SO computes its share:

```
R = sum_j (D_j + rho_j * E_j)
c = H(R || Y || m)
z_i = d_i + rho_i * e_i + lambda_i * s_i * c
```

where `lambda_i` is the Lagrange coefficient over the participating set. A coordinator
sums z_i to get z. The signature is (R, z), a valid Schnorr signature over Y.

### Robustness

If an SO sends a malformed share, the others can detect this (verify the partial
signature equation). FROST then drops the malicious party and re-derives Lagrange
coefficients over the remaining honest k-set.

### Spark's deployment

- **Beta**: 2-of-2 with Lightspark + Flashnet. This is technically a MuSig2-equivalent
  configuration (no threshold benefit) but uses FROST tooling for forward compatibility.
- **Roadmap**: 5-of-7 or 7-of-11 spread across jurisdictions to make collusion
  legally as well as technically hard.
- **Key refresh**: SOs periodically run a proactive secret-resharing protocol so that
  share compromise must occur within a single epoch to be useful.

## Worked example

User Alice requests "send 5,000 sats from leaf_X to Bob's leaf_Y":

```
T+0ms      Alice's wallet POSTs transfer to Spark coordinator (Lightspark API).
T+15ms     Coordinator constructs leaf_X spend tx, computes sighash m.
T+25ms     Coordinator picks pre-committed nonces from SO_1 and SO_2's
           pre-published nonce pool (avoids round 1 latency).
T+30ms     Coordinator sends m + binding info to both SOs.
T+50ms     SO_1 returns z_1.  SO_2 returns z_2.
T+55ms     Coordinator sums to z, verifies (R, z) against Y.
T+60ms     Coordinator updates internal tree state, sends new exit-branch
           pre-signatures to Alice and Bob.
T+80ms     User wallets confirm leaf updated.

End-to-end < 100ms typical.  No on-chain tx.
```

If SO_2 is offline at T+30, the coordinator can wait up to ~5 seconds and either retry
or, in 5-of-7 deployments, route to another SO without restarting from scratch.

## Trade-offs and security

- **k-of-n robustness**: in 5-of-7, up to 2 SOs can be offline or malicious without
  affecting liveness or safety. In 2-of-2 beta, both must be honest *and* online --
  effectively a 2-of-2 multisig with an audit trail.
- **Nonce reuse vulnerability**: if any SO reuses a nonce across two distinct messages,
  its share leaks. Spark uses deterministic-nonce-or-secure-randomness implementations
  with strict pre-commitment caches.
- **DKG trust at setup**: an attacker present during DKG who controls k parties can
  forever sign as the group. Spark publishes DKG transcripts and uses publicly-attestable
  key-ceremony hardware.
- **Equivocation**: a malicious coordinator could ask SOs to sign two different leaf
  updates for the same user. SOs are required to maintain a per-leaf nonce-pinning log;
  equivocation is detectable post hoc but does not automatically prevent a fast attack.
  Spark mitigates with a "1-of-n watchtower" set that monitors for double-signing.
- **Compared to Ark (MuSig2)**: ARKADE chose MuSig2 because rounds are batched and
  liveness is restartable; Spark chose FROST because continuous individual transfers
  must succeed even if one SO is briefly down.

## References

- FROST IRTF draft - https://datatracker.ietf.org/doc/draft-irtf-cfrg-frost/
- Komlo and Goldberg, "FROST: Flexible Round-Optimized Schnorr Threshold Signatures" (2020)
- ZF FROST library - https://github.com/ZcashFoundation/frost
- Spark technical docs - https://docs.spark.money/
