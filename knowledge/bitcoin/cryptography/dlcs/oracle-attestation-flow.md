# DLC Oracle Attestation Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/dlcs`.
> Canonical source: dlcspecs Oracle.md + Suredbits / Crypto Garage oracle
>                   reference implementations
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/dlcs/SKILL.md

## Concept

A DLC oracle is a **single-purpose Schnorr signer** that commits in advance
to a nonce `R_event` and later publishes one signature `s_outcome` per
attested outcome. The published `s_outcome` doubles as the adaptor secret
that lets the favored bettor finalize their pre-signed CET. The oracle never
holds funds, never sees the contract, and signs only the outcome label —
yet its signature value is exactly what unlocks the on-chain settlement.
This article details the announcement, attestation, and consumption stages,
including the multi-nonce variant for numerical outcomes and the cryptographic
property that lets bettors **bake the future signature value into a pre-sig
today**.

## Walkthrough / mechanics

### Oracle long-term setup

```
Long-term: oracle samples d_oracle <- {1..n-1}
           P_oracle = d_oracle * G                  (BIP340 even-y pubkey)
           Publishes P_oracle out-of-band (oracle's public profile)
```

Parties using DLCs trust this `P_oracle` as the oracle's identity.

### Per-event announcement

For each upcoming attestation event, the oracle samples a fresh nonce and
publishes an **announcement**:

```
For event with id event_id and possible outcomes [o_1, ..., o_m]:
  k_event   <- {1..n-1}                              (sampled randomly)
  R_event   = k_event * G                            (nonce point, x-only)
  Persist k_event in single-use storage tagged with event_id

  announcement = {
    oracle_pubkey: P_oracle
    event_id, descriptor (text/url/etc.)
    nonce: R_event                                   (32 bytes, x-only)
    outcomes: [o_1, ..., o_m]                        (canonical label list)
    timestamp: maturity_epoch
  }

  // Attestation announcement is itself signed by oracle for authenticity:
  ann_sig = Schnorr.Sign(d_oracle, hash(announcement))

  Publish (announcement, ann_sig) to oracle's mailing list / API.
```

Bettors download this announcement. The pair `(P_oracle, R_event)`
determines the **adaptor points** for each outcome via:

```
e_i        = TaggedHash("DLC/oracle/attestation/v0",
                        R_event || P_oracle || o_i) mod n
T_i        = lift_x(R_event) + e_i * P_oracle
```

Each outcome `o_i` has its own `T_i`. Bettors pre-sign one CET per outcome
with adaptor point `T_i`.

### Per-event attestation

When the event resolves to outcome `o_actual`, the oracle publishes:

```
e_actual = TaggedHash("DLC/oracle/attestation/v0",
                       R_event || P_oracle || o_actual) mod n
s_actual = (k_event + e_actual * d_oracle) mod n      (Schnorr sig scalar)

attestation = {
  event_id
  outcome: o_actual
  signature: s_actual                                  (32 bytes)
}

Wipe k_event from oracle's storage.
```

The attestation is a normal Schnorr signature: `(R_event, s_actual)` is a
valid BIP340 sig under `P_oracle` over message `o_actual`. Anyone can verify:

```
verify: s_actual * G == lift_x(R_event) + e_actual * P_oracle
       = lift_x(R_event) + (s_actual - k_event) * G       // by construction
       = lift_x(R_event) + s_actual G - k_event G
       = R_event_lifted + s_actual G - R_event_lifted
       = s_actual G                                       OK
```

### Why `s_actual` is the adaptor secret

Recall a bettor pre-signed their CET[o_actual] with adaptor point `T_actual
= lift_x(R_event) + e_actual * P_oracle`. The discrete log of `T_actual` is:

```
log_G(T_actual) = log_G(lift_x(R_event)) + e_actual * log_G(P_oracle)
                = k_event + e_actual * d_oracle
                = s_actual                           (by definition)
```

So **publishing the attestation IS publishing the adaptor secret**. Anyone
who saw the pre-sig and the attestation runs `Adapt(pre_sig, s_actual)` and
gets a valid 2-of-2 sig on the CET.

This is the keystone identity of the DLC construction: oracle signing math
and bettor adaptor math line up exactly.

### Multi-nonce attestation (numerical outcomes)

For continuous values (e.g., BTC price), a single nonce per bit is published:

```
For an outcome digit i in 0..K-1 (e.g., K = 20 bits = ~1M-resolution):
  k_i   <- {1..n-1}
  R_i   = k_i * G
  Publish [R_0, R_1, ..., R_{K-1}] in announcement.

When event resolves to numeric value V:
  Decompose V into base-2 digits b_{K-1}...b_1 b_0 (or base-10, base-256 etc.)
  For each digit i:
    e_i  = H(R_i || P_oracle || str(b_i))
    s_i  = (k_i + e_i * d_oracle) mod n
  Publish [s_0, s_1, ..., s_{K-1}] all together.
```

Bettors construct CETs over **ranges** of bit patterns:

```
For range "value in [lo, hi)" expressed as a set of bit patterns
  with shared high-bit prefix (binary tree style),
  the CET's adaptor point is the SUM of T_i for the fixed bits
  in that prefix.

Adapt-time: bettor combines the s_i for the fixed bits to form the secret.
```

The CET tree (covered in cet-tree-numerical-outcomes.md) keeps the CET count
logarithmic in resolution, not linear.

### Multi-oracle (k-of-n)

For high-value contracts, m oracles each publish their own nonces; the CET's
adaptor point is the sum (or threshold combination) of m oracle attestations:

```
T_combined = sum over j of T_{oracle_j, outcome}

s_combined = sum over j of s_{oracle_j, outcome}
```

Verify `s_combined * G == T_combined` mod parity tracking. Trust assumption:
attacker must compromise *all m* oracles in lock-step (or k-of-n quorum
variant via FROST-style threshold attestation).

## Worked example

Toy oracle, two-outcome event "BTC > 50k tomorrow". `n = 23`.

```
Oracle long-term: d_oracle = 13, P_oracle = 13G

Announcement (samples per-event nonce):
  k_event = 5
  R_event = 5G
  outcomes = ["above", "below"]
  Publishes (P_oracle = 13G, R_event = 5G, ["above", "below"])

Bettor computes adaptor points:
  e_above = H(5G || 13G || "above") mod 23, say = 7
  T_above = lift_x(5G) + 7 * 13G = 5G + 91G mod 23 = 5G + 91G
            91 mod 23 = 91 - 3*23 = 22
          = 5G + 22G = 27G mod 23 = 4G
  e_below = H(5G || 13G || "below") mod 23, say = 11
  T_below = 5G + 11 * 13G = 5G + 143G
            143 mod 23 = 143 - 6*23 = 5
          = 5G + 5G = 10G

Bettor pre-signs CET_above with T_above = 4G,
              pre-signs CET_below with T_below = 10G.

Event resolves to "above". Oracle publishes:
  e_above = 7
  s_actual = (k_event + e_above * d_oracle) mod n
           = (5 + 7 * 13) mod 23
           = (5 + 91) mod 23
           = 96 mod 23 = 96 - 4*23 = 4
  Publishes attestation = ("above", s_actual = 4)

Verify: s_actual G ?= T_above
        4 G            4G                  OK

Bettor adapts pre_sig_alice_above (assume s'_alice_above = 18):
  s_full_alice = (s'_alice_above + s_actual) mod n
               = (18 + 4) mod 23
               = 22
  Bettor publishes the CET with the now-finalized witness.
```

The same `s_actual = 4` value used by the bettor as adaptor secret is
identical to the s-value of the BIP340 sig anyone can verify against
`P_oracle` over message "above".

## Common pitfalls

- **Reusing `k_event` across events.** Two events signed with the same
  nonce leak `d_oracle`. The oracle must persist used nonces with a
  permanent flag and refuse to re-sign with one. Standard Schnorr
  nonce-reuse failure mode.
- **Insufficient announcement signing.** Without `ann_sig`, an MITM can
  swap `R_event` and trick bettors into pre-signing CETs that the oracle
  cannot fulfill (because oracle never committed to that R).
- **Outcome label canonicalization.** "Above" vs "above" vs "ABOVE" hash
  to different `e_i`, producing different `T_i`. dlcspecs mandates UTF-8
  NFC encoding with byte-exact match.
- **Late nonce publication.** The R_event must be published *before* the
  event resolves; otherwise the oracle could collude with one bettor by
  picking R_event after-the-fact to favor a specific outcome.
- **Lying about the outcome.** The oracle can sign a false outcome. DLCs
  do not protect against malicious oracle behavior — only against oracle
  *unavailability* (via refund). Mitigation: multi-oracle sums.

## References

- dlcspecs Oracle.md:
  https://github.com/discreetlogcontracts/dlcspecs/blob/master/Oracle.md
- Suredbits oracle reference:
  https://github.com/Suredbits/krystal-ball
- Crypto Garage P2PDerivatives oracle:
  https://github.com/p2pderivatives/oracle-rs
- Numeric outcomes spec:
  https://github.com/discreetlogcontracts/dlcspecs/blob/master/NumericOutcome.md
