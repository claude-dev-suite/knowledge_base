# OAuth 2.0 Provider Integration

> Official Documentation: https://openid.net/specs/openid-connect-discovery-1_0.html

## Overview

Integrating with an OAuth 2.0 / OpenID Connect (OIDC) provider means: registering
a client, discovering the provider's endpoints, redirecting the user through the
authorization flow, and validating the tokens you receive. OIDC layers
authentication (an ID token identifying the user) on top of OAuth 2.0's
authorization. This document uses a generic provider and notes the patterns for
Google, GitHub, and Auth0.

## Registering a Client

Before any flow, register your application with the provider to obtain a
`client_id` and (for confidential clients) a `client_secret`. During
registration you declare:

- **Redirect URIs** — the exact URLs the provider may redirect back to.
- **Client type** — public (SPA, mobile, no secret) or confidential (server-side).
- **Scopes / permissions** the app may request.
- **Grant types / response types** the app will use.

### Redirect URIs

- Registered and validated by **exact string match** (OAuth 2.1). `https://app.example.com/callback` and `https://app.example.com/callback/` are different.
- Use HTTPS in production. `http://localhost` (or a loopback IP) is generally allowed for native/dev clients.
- Never use wildcards or open redirects; an attacker-controllable redirect URI leaks authorization codes and tokens.

## OIDC Discovery

OIDC providers publish metadata at a well-known path formed by appending
`/.well-known/openid-configuration` to the issuer URL. This lets you avoid
hard-coding endpoints.

```http
GET /.well-known/openid-configuration HTTP/1.1
Host: accounts.example.com
```

```json
{
  "issuer": "https://accounts.example.com",
  "authorization_endpoint": "https://accounts.example.com/authorize",
  "token_endpoint": "https://accounts.example.com/oauth/token",
  "userinfo_endpoint": "https://accounts.example.com/userinfo",
  "jwks_uri": "https://accounts.example.com/.well-known/jwks.json",
  "registration_endpoint": "https://accounts.example.com/register",
  "scopes_supported": ["openid", "profile", "email"],
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token", "client_credentials"],
  "id_token_signing_alg_values_supported": ["RS256", "ES256"]
}
```

## Endpoints

| Endpoint | Metadata field | Purpose |
|----------|----------------|---------|
| Authorization | `authorization_endpoint` | Front-channel; user authenticates and consents, returns a code |
| Token | `token_endpoint` | Back-channel; exchange code/refresh token for access & ID tokens |
| UserInfo | `userinfo_endpoint` | Returns claims about the authenticated user (OIDC) |
| JWKS | `jwks_uri` | Public keys (JSON Web Key Set) to verify token signatures |

### UserInfo request

Call UserInfo with the access token to retrieve profile claims:

```http
GET /userinfo HTTP/1.1
Host: accounts.example.com
Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
```

```json
{ "sub": "248289761001", "name": "Jane Doe", "email": "jane@example.com" }
```

## Scopes

Scopes limit what an access token can do. Request the minimum you need.

- **OIDC standard scopes**: `openid` (required to get an ID token), `profile`, `email`, `address`, `phone`, `offline_access` (requests a refresh token).
- **Provider-specific scopes** control API access, e.g. `https://www.googleapis.com/auth/drive.readonly` (Google) or `repo`, `read:user` (GitHub).

## Generic Authorization Code Flow

```http
GET /authorize?response_type=code
  &client_id=CLIENT_ID
  &redirect_uri=https%3A%2F%2Fapp.example.com%2Fcallback
  &scope=openid%20profile%20email
  &state=RANDOM_CSRF_TOKEN
  &code_challenge=CODE_CHALLENGE
  &code_challenge_method=S256 HTTP/1.1
Host: accounts.example.com
```

The provider redirects back with `?code=...&state=...`. Verify `state`, then
exchange the code at the token endpoint (see `oauth2/flows.md`). With the
`openid` scope the token response also includes an `id_token` (a JWT).

## Provider Patterns

### Google

- Discovery: `https://accounts.google.com/.well-known/openid-configuration`.
- Fully OIDC-compliant; issues ID tokens (RS256) and supports refresh tokens
  when you request `access_type=offline` (Google-specific parameter). Configure
  the client in Google Cloud Console → APIs & Services → Credentials.

### GitHub

- OAuth 2.0 but **not** OIDC — there is no ID token or discovery document.
  Endpoints are fixed: `https://github.com/login/oauth/authorize` and
  `https://github.com/login/oauth/access_token`. Identify the user via the REST
  API `GET https://api.github.com/user` with the access token. By default the
  token response is form-encoded; send `Accept: application/json` to get JSON.

### Auth0

- Discovery: `https://YOUR_TENANT.auth0.com/.well-known/openid-configuration`.
  Full OIDC. Use the `audience` parameter on the authorization request to obtain
  an access token for a specific API, and `offline_access` scope for refresh
  tokens. Rotates refresh tokens by default.

## Token Validation

How you validate depends on the token format the provider issues.

### ID tokens and JWT access tokens

ID tokens are always JWTs; many providers (Google, Auth0) also issue JWT access
tokens. Validate them **locally**:

1. Fetch the signing keys from `jwks_uri` (cache them; refresh on key rotation).
2. Verify the signature using the key whose `kid` matches the JWT header.
3. Check `iss` equals the provider's issuer, `aud` includes your `client_id`
   (ID token) or API identifier (access token), and `exp`/`nbf` are valid.
4. For login, verify the `nonce` you sent matches the `nonce` claim.

See `jwt/implementation.md` and `jwt/security.md` for details.

### Opaque access tokens (e.g. GitHub)

Opaque tokens carry no verifiable structure. The resource server validates them
by introspection (RFC 7662, `POST /introspect`) or a provider API call. Do not
attempt to parse them.
