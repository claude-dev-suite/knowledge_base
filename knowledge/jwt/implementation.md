# JWT Implementation

> Official Documentation: https://datatracker.ietf.org/doc/html/rfc7519

## Overview

A JSON Web Token (JWT, RFC 7519) is a compact, URL-safe way to represent claims
between two parties. The most common form is a **JWS** (JSON Web Signature,
https://datatracker.ietf.org/doc/html/rfc7515): a signed token whose integrity
the recipient can verify. A signed JWT is *not* encrypted — anyone can read the
payload — so never put secrets in it.

## Structure

A JWS-based JWT is three Base64URL-encoded parts joined by dots:

```
header.payload.signature
```

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0IiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKx...
└────────── header ──────────┘ └──────────── payload ───────────┘ └ signature ┘
```

### Header

Describes the token type and signing algorithm.

```json
{ "alg": "RS256", "typ": "JWT", "kid": "2024-key-01" }
```

- `alg` — signing algorithm (RFC 7518). Required.
- `typ` — token type, usually `JWT`.
- `kid` — key identifier; tells the verifier which key to use (essential for rotation).

### Payload

A JSON object of **claims**. Base64URL-decoding it reveals the data in plaintext.

```json
{
  "iss": "https://auth.example.com",
  "sub": "1234567890",
  "aud": "https://api.example.com",
  "exp": 1735689600,
  "iat": 1735686000,
  "nbf": 1735686000,
  "jti": "a1b2c3d4",
  "scope": "read write"
}
```

### Signature

Computed over `base64url(header) + "." + base64url(payload)` using the algorithm
in `alg`. It proves the token was issued by a holder of the signing key and has
not been tampered with.

## Registered Claims

RFC 7519 §4.1 defines these standard claims (all optional but recommended):

| Claim | Name | Meaning |
|-------|------|---------|
| `iss` | Issuer | Who issued the token |
| `sub` | Subject | The principal (e.g. user ID) the token is about |
| `aud` | Audience | Recipient(s) the token is intended for |
| `exp` | Expiration Time | Reject at/after this time (NumericDate, seconds since epoch) |
| `nbf` | Not Before | Reject before this time |
| `iat` | Issued At | When the token was issued |
| `jti` | JWT ID | Unique identifier; enables replay/revocation tracking |

All time claims are **NumericDate**: seconds since 1970-01-01 UTC.

## Signing Algorithms

| Algorithm | Type | Keys | Notes |
|-----------|------|------|-------|
| HS256 | HMAC + SHA-256 | One shared secret | Symmetric; signer and verifier share the key. Simple, fast. |
| RS256 | RSA + SHA-256 | Private key signs, public key verifies | Asymmetric; verifiers only need the public key. |
| ES256 | ECDSA P-256 + SHA-256 | Private/public key pair | Asymmetric; smaller keys and signatures than RSA. |

Guidance:

- Use **HS256** only when the same trusted party issues and verifies (e.g. one
  monolith). The secret must be long and random; anyone with it can forge tokens.
- Use **RS256** or **ES256** when third parties must verify without being able to
  sign — the standard for OIDC ID tokens. Publish the public key via a JWKS
  endpoint (`jwks_uri`) and identify it with `kid`.
- Always pin the expected algorithm(s) on verification. See `jwt/security.md`
  for the `none` and algorithm-confusion attacks.

## Creating and Verifying (Node.js)

The examples below use **third-party libraries** — `jsonwebtoken` and `jose` are
npm packages, not part of the spec or Node core. `jose` is modern and
promise-based; `jsonwebtoken` is widely used. Choose one.

### With `jsonwebtoken` (third-party library)

```js
// npm install jsonwebtoken
const jwt = require("jsonwebtoken");

// Sign (HS256 with a shared secret)
const token = jwt.sign(
  { sub: "1234567890", scope: "read write" },
  process.env.JWT_SECRET,
  { algorithm: "HS256", issuer: "https://auth.example.com",
    audience: "https://api.example.com", expiresIn: "15m" }
);

// Verify — always pin algorithms, issuer, and audience
try {
  const payload = jwt.verify(token, process.env.JWT_SECRET, {
    algorithms: ["HS256"],
    issuer: "https://auth.example.com",
    audience: "https://api.example.com",
  });
  console.log(payload.sub);
} catch (err) {
  // TokenExpiredError, JsonWebTokenError, NotBeforeError
}
```

### With `jose` (third-party library, RS256)

```js
// npm install jose
import { SignJWT, jwtVerify, importPKCS8, createRemoteJWKSet } from "jose";

// Sign with a private key
const privateKey = await importPKCS8(process.env.PRIVATE_KEY_PEM, "RS256");
const token = await new SignJWT({ scope: "read write" })
  .setProtectedHeader({ alg: "RS256", kid: "2024-key-01" })
  .setIssuer("https://auth.example.com")
  .setAudience("https://api.example.com")
  .setSubject("1234567890")
  .setIssuedAt()
  .setExpirationTime("15m")
  .sign(privateKey);

// Verify against a remote JWKS (fetches & caches the public keys)
const JWKS = createRemoteJWKSet(new URL("https://auth.example.com/.well-known/jwks.json"));
const { payload } = await jwtVerify(token, JWKS, {
  issuer: "https://auth.example.com",
  audience: "https://api.example.com",
});
```

Frameworks such as **passport** (with `passport-jwt`) wrap this verification for
Express; they are also third-party libraries.

## Access vs Refresh Tokens

| | Access token | Refresh token |
|---|--------------|---------------|
| Purpose | Authorize API calls | Obtain new access tokens |
| Sent to | Resource server (every request) | Authorization server only |
| Lifetime | Short (e.g. 5–15 min) | Long (hours–days), often rotated |
| Format | Often a JWT (self-contained) | Usually opaque; validated by the auth server |

A JWT access token is validated locally by verifying its signature and claims,
so the resource server needs no database lookup. This is also its main drawback:
a valid JWT cannot be revoked before `exp` without extra machinery. Keep access
tokens short-lived and use refresh tokens (with rotation) for longevity — see
`jwt/security.md` for revocation strategies.
