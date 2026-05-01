# FROST DKG Protocol Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/frost`.
> Canonical source: Pedersen DKG (1991) + draft-irtf-cfrg-frost / ZF FROST spec
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/frost/SKILL.md

## Concept

A Distributed Key Generation (DKG) sets up a t-of-n FROST signing group with no
single party ever holding the master secret `d`. The construction in current
FROST drafts is a Pedersen-style two-round DKG with verifiable secret sharing
(VSS). Each party picks a random degree-(t-1) polynomial whose constant term is
their secret contribution; the group's master secret ends up being the sum of
those constant terms, distributed via Shamir shares of the sum-polynomial. The
delicate parts are: (1) every share must be verifiable from broadcast
commitments before being accepted, and (2) any complainant must be resolvable
without leaking shares of honest parties.

## Walkthrough / mechanics

### Setup parameters

```
t = threshold (any t can sign)
n = total parties
P_i = participant i for i in 1..n   (use small integer "identifiers")
G = secp256k1 generator
q = secp256k1 group order
```

### Round 1 - polynomial commitment broadcast

Each P_i:

```
1. Sample random polynomial f_i of degree t-1:
     f_i(x) = a_{i,0} + a_{i,1} x + ... + a_{i,t-1} x^{t-1}
   where each a_{i,j} <- {1..q-1}

2. Compute commitment vector (Feldman VSS):
     C_{i,j} = a_{i,j} * G    for j in 0..t-1

3. Compute proof of knowledge of a_{i,0} (Schnorr POK):
     k_i  <- {1..q-1}
     R_i  = k_i * G
     mu_i = H(i, ctx, C_{i,0}, R_i)
     z_i  = (k_i + a_{i,0} * mu_i) mod q
   POK_i = (R_i, z_i)

4. Broadcast (C_{i,0}, ..., C_{i,t-1}, POK_i) to all peers.
```

Verifier on receipt of P_j's broadcast:
```
mu_j  = H(j, ctx, C_{j,0}, R_j)
check: z_j * G == R_j + mu_j * C_{j,0}
```

If POK fails, P_j is excluded from the protocol.

### Round 2 - share distribution

Each P_i computes share for each other P_l (l != i):

```
share_{i->l} = f_i(l) mod q
```

and sends it over a **secure private channel** to P_l. Each P_l verifies:

```
expected = sum over j in 0..t-1 of (l^j) * C_{i,j}     // EC point math
check: share_{i->l} * G == expected
```

If the equality fails, P_l broadcasts a complaint against P_i with
share_{i->l} as evidence (P_i is now publicly malicious and excluded).

### Final share computation

After Round 2 completes for all honest pairs:

```
P_l's long-term secret share:
  s_l = sum over i (i in surviving set) of share_{i->l}   mod q

P_l's verification key:
  S_l = sum over i (i in surviving set) of (sum over j of l^j * C_{i,j})
      = s_l * G

Group public key:
  P = sum over i of C_{i,0}
    = sum over i of a_{i,0} * G
    = (sum a_{i,0}) * G
    = d * G    where d = sum a_{i,0} is the implicit master secret
```

No party ever computes `d` directly. Each P_l ends up with `(s_l, S_l, P)`,
which is exactly Shamir's-secret-sharing form for d under polynomial
`F(x) = sum_i f_i(x)`, since `s_l = F(l)` and `d = F(0)`.

### Identifiable abort

Modern FROST DKG variants (chillDKG, ZF FROST 1.0) embed each share in a
non-interactive zero-knowledge encryption (e.g., ECIES) so the complaint
itself can be verified by any third party — no he-said-she-said. The naive
Pedersen flow above leaks the share in a complaint, but the surviving group
reuses only honest shares so this is acceptable for one-shot DKG.

## Worked example

3-of-5 group (t=3, n=5), participants P_1..P_5.

```
P_1 picks f_1(x) = 7 + 11x + 4x^2  (toy values; real ones are 256-bit)
        C_{1,0} = 7G, C_{1,1} = 11G, C_{1,2} = 4G
        POK_1   = (R_1, z_1) for log_G(7G)=7

P_2 picks f_2(x) = 3 + 9x  + 6x^2
P_3 picks f_3(x) = 1 + 5x  + 8x^2
P_4 picks f_4(x) = 2 + 13x + 10x^2
P_5 picks f_5(x) = 4 + 7x  + 12x^2

Shares to P_3 (l=3):
  share_{1->3} = f_1(3) = 7 + 33 + 36 = 76
  share_{2->3} = f_2(3) = 3 + 27 + 54 = 84
  share_{4->3} = f_4(3) = 2 + 39 + 90 = 131
  share_{5->3} = f_5(3) = 4 + 21 + 108 = 133
  (P_3's own contribution f_3(3) = 1 + 15 + 72 = 88)

P_3's verification (e.g., for P_1's share):
  l=3, t=3
  expected = 1*C_{1,0} + 3*C_{1,1} + 9*C_{1,2}
           = (7 + 33 + 36)G = 76G
  Check: 76 G == expected. OK.

P_3's final share:
  s_3 = (76 + 84 + 88 + 131 + 133) mod q = 512

Master secret (no party computes this directly):
  d = a_{1,0}+a_{2,0}+a_{3,0}+a_{4,0}+a_{5,0} = 7+3+1+2+4 = 17
  Group pubkey: P = 17 G
```

Lagrange-reconstructed at any t=3 subset (say {1, 3, 5}):
```
F(0) = sum over l in {1,3,5} of (s_l * lambda_l)
where lambda_l = product over m != l of m / (m - l)   mod q

lambda_1 = (3*5)/((3-1)*(5-1)) = 15/8
lambda_3 = (1*5)/((1-3)*(5-3)) = 5/(-4)
lambda_5 = (1*3)/((1-5)*(3-5)) = 3/8
```

The actual scalars are computed mod q in production. A FROST signing session
plugs each `s_l * lambda_l` into the partial-signature aggregation.

## Common pitfalls

- **Skipping the POK on `a_{i,0}`.** Without it, a malicious P_i can pick
  `a_{i,0}` after seeing others' commitments, mounting a rogue-key-style
  attack on the group public key.
- **Assuming all P_i survive.** A robust DKG must terminate with a *subset*
  S of surviving participants, and shares are summed only over S. Hard-coding
  `range(1..n)` in the share-sum is a real bug shipped in early FROST libs.
- **Reusing the DKG context across runs.** The transcript hash `ctx` must
  bind run identifier, group parameters, and round labels. Replay across runs
  enables cross-run forgery.
- **Insecure share transport.** Pedersen DKG assumes pairwise authenticated
  encrypted channels. Sending shares in plaintext over a coordinator is a
  total break.

## References

- Gennaro, Jarecki, Krawczyk, Rabin (1999) "Secure Distributed Key Generation":
  https://link.springer.com/chapter/10.1007/3-540-48910-X_21
- ZF FROST DKG spec: https://github.com/ZcashFoundation/frost/blob/main/book/src/dkg.md
- chillDKG (Ruffing): https://github.com/BlockstreamResearch/bip-frost-dkg
- draft-irtf-cfrg-frost: https://datatracker.ietf.org/doc/draft-irtf-cfrg-frost/
