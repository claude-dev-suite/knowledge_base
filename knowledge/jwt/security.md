# JWT Security Best Practices

> Official Documentation: https://datatracker.ietf.org/doc/html/rfc8725

## Overview

JWTs are frequently misused in ways that undermine their security. RFC 8725
(JSON Web Token Best Current Practices) catalogs the pitfalls; this document
summarizes the practical rules. The recurring theme: **the verifier, not the
token, decides how a token is validated.** Never trust values in the token
(especially the `alg` header) to drive your verification logic.

## Algorithm Attacks

### The `none` algorithm attack

JWS defines an `alg` value of `none` for unsecured tokens (no signature). An
attacker can take a valid token, change the header to `{"alg":"none"}`, strip the
signature, and modify the payload. A naive verifier that honors the header's
`alg` accepts it.

- **Mitigation:** Always pass an explicit allow-list of algorithms to your verify
  function (e.g. `algorithms: ["RS256"]`) and never include `none`. Reject any
  token whose `alg` is not in the list.

### Algorithm confusion (RS256 → HS256)

If your code selects the verification algorithm from the token header, an
attacker can change `alg` from `RS256` to `HS256` and sign the token using your
**public RSA key** as the HMAC secret. Since the public key is, by definition,
public, the forged signature verifies.

- **Mitigation:** Pin the expected algorithm(s) explicitly. Never let the token's
  header choose between symmetric and asymmetric verification. Use key objects
  typed to a single algorithm where the library supports it.

## Validate Every Security-Relevant Claim

A valid signature only proves integrity — you must still check the claims:

- **`exp`** — reject expired tokens. Allow only a small clock-skew leeway (e.g. 30–60s).
- **`nbf`** — reject tokens used before their not-before time.
- **`iss`** — must equal the exact issuer you expect.
- **`aud`** — must include *your* service. This stops a token minted for another
  audience from being replayed against you (the "confused deputy" problem).
- **`sub` / `scope`** — enforce authorization *after* validating the token.

Verifying only the signature and expiry, while ignoring `iss`/`aud`, is a common
and serious mistake.

## Key Management and Rotation

- Store signing keys in a secret manager or HSM/KMS, never in source control.
- HMAC secrets must be long and high-entropy (≥ 256 bits for HS256).
- Publish public keys via a JWKS endpoint and tag each with a `kid`. Verifiers
  select the key by the token's `kid` header.
- **Rotate regularly.** Introduce a new key, sign new tokens with it, but keep the
  old public key in the JWKS until all tokens signed by it have expired. Then
  retire it. This overlap avoids breaking in-flight tokens.
- Verifiers should cache JWKS and refresh on an unknown `kid` (with rate limiting
  to avoid a fetch storm).

## Short Expiry + Refresh

Because signed JWTs are self-contained and hard to revoke, limit the blast radius
of a leaked token with a short lifetime.

- **Access tokens:** short-lived (5–15 minutes).
- **Refresh tokens:** longer-lived, opaque, stored server-side, and **rotated** on
  each use. If a rotated (already-used) refresh token is presented again, treat it
  as theft: revoke the whole token family and force re-authentication.

## Token Storage (Browser)

| Storage | Pros | Cons |
|---------|------|------|
| `httpOnly` + `Secure` + `SameSite` cookie | Not readable by JS → resists XSS token theft; sent automatically | Needs CSRF defense (`SameSite`, CSRF tokens) |
| `localStorage` / `sessionStorage` | Simple; immune to CSRF | Readable by any JS → any XSS steals the token |

- Prefer **`httpOnly`, `Secure`, `SameSite=Lax` (or `Strict`) cookies** for tokens
  used by browser apps, and pair with CSRF protection.
- Avoid `localStorage` for tokens; a single XSS flaw exfiltrates them.
- Never place tokens in URLs or the query string — they leak via logs, history,
  and `Referer` headers.

## Revocation Strategies

Self-contained JWTs remain valid until `exp`. To revoke earlier:

- **Short expiry (primary defense):** small windows make revocation rarely urgent.
- **Denylist by `jti`:** store revoked token IDs (in Redis, TTL = remaining
  lifetime) and check on each request. Reintroduces a lookup, trading away the
  stateless benefit.
- **Refresh-token revocation:** revoke the refresh token so no new access tokens
  can be minted; existing access tokens expire naturally.
- **Token versioning:** include a per-user `token_version` claim; bump it on
  logout/password-change to invalidate all outstanding tokens for that user.
- **Introspection (RFC 7662):** use opaque, server-validated access tokens when
  immediate revocation is a hard requirement.

## Common Pitfalls Checklist

- [ ] Verify algorithm is pinned; `none` rejected.
- [ ] Signature verified against the correct key (no RS256→HS256 confusion).
- [ ] `exp`, `nbf`, `iss`, and `aud` all validated.
- [ ] No secrets or PII in the (readable) payload.
- [ ] Access tokens short-lived; refresh tokens rotated.
- [ ] Tokens in `httpOnly` cookies, not `localStorage`; never in URLs.
- [ ] Signing keys in a secret store; rotation with `kid` and JWKS in place.
- [ ] HTTPS enforced end to end.
