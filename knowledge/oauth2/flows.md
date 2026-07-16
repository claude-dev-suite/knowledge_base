# OAuth 2.0 Flows and Grant Types

> Official Documentation: https://datatracker.ietf.org/doc/html/rfc6749

## Overview

OAuth 2.0 is an authorization framework that lets a client application obtain
limited access to a resource server on behalf of a resource owner, without
handling the owner's credentials directly. Access is granted by exchanging a
**grant** for an **access token** at the authorization server's token endpoint.

This document covers the grant types (flows) defined by RFC 6749 and its
extensions, and follows the guidance of the OAuth 2.1 draft
(https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1), which requires
PKCE for all authorization code flows and removes the implicit and password
grants.

### Roles

| Role | Description |
|------|-------------|
| Resource owner | The user who owns the data |
| Client | The application requesting access |
| Authorization server | Issues tokens after authenticating the owner/client |
| Resource server | Hosts the protected resources; accepts access tokens |

### Choosing a grant type

| Scenario | Grant type |
|----------|-----------|
| Web app, SPA, mobile/native app (user present) | Authorization Code + PKCE |
| Machine-to-machine, no user | Client Credentials |
| Input-constrained device (TV, CLI) | Device Authorization |
| Renew an access token without re-auth | Refresh Token |
| ~~Legacy browser app~~ | ~~Implicit~~ (removed in 2.1) |
| ~~First-party credential exchange~~ | ~~Password / ROPC~~ (removed in 2.1) |

## Authorization Code + PKCE

The recommended flow for any client acting on behalf of a user. PKCE (Proof Key
for Code Exchange, https://datatracker.ietf.org/doc/html/rfc7636) binds the
authorization request to the token request, preventing authorization code
interception. Under OAuth 2.1, PKCE is required for **both public and
confidential** clients.

### Step 1: Client generates a PKCE pair

- `code_verifier`: a high-entropy random string, 43–128 characters from the
  unreserved set `[A-Z] [a-z] [0-9] - . _ ~`.
- `code_challenge` = `BASE64URL-ENCODE(SHA256(ASCII(code_verifier)))` when
  `code_challenge_method=S256` (mandatory to implement; use it over `plain`).

### Step 2: Authorization request (browser redirect)

```http
GET /authorize?response_type=code
  &client_id=s6BhdRkqt3
  &redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb
  &scope=openid%20profile
  &state=xyz
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256 HTTP/1.1
Host: authorization-server.example.com
```

The user authenticates and consents. The server redirects back with a code:

```http
HTTP/1.1 302 Found
Location: https://client.example.com/cb?code=SplxlOBeZQQ&state=xyz
```

Always verify that `state` matches the value you sent (CSRF protection).

### Step 3: Token request (back channel)

```http
POST /token HTTP/1.1
Host: authorization-server.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=SplxlOBeZQQ
&redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb
&client_id=s6BhdRkqt3
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

Confidential clients also authenticate here (e.g. HTTP Basic with the client
secret). The server validates the verifier against the earlier challenge:

```json
{
  "access_token": "2YotnFZFEjr1zCsicMWpAA",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
  "scope": "openid profile"
}
```

## Client Credentials

For machine-to-machine access where the client acts as itself, with no user.
Requires a confidential client (it must authenticate). RFC 6749 §4.4.

```http
POST /token HTTP/1.1
Host: authorization-server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&scope=reports:read
```

```json
{ "access_token": "2YotnFZFEjr1zCsicMWpAA", "token_type": "Bearer", "expires_in": 3600 }
```

No refresh token is issued; the client simply requests a new token when needed.

## Device Authorization Grant

For input-constrained devices (smart TVs, CLIs, IoT). RFC 8628. The device shows
a code and URL; the user completes authorization on a second device.

### Step 1: Device authorization request

```http
POST /device_authorization HTTP/1.1
Host: authorization-server.example.com
Content-Type: application/x-www-form-urlencoded

client_id=1406020730&scope=profile
```

```json
{
  "device_code": "GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS",
  "user_code": "WDJB-MJHT",
  "verification_uri": "https://example.com/device",
  "verification_uri_complete": "https://example.com/device?user_code=WDJB-MJHT",
  "expires_in": 1800,
  "interval": 5
}
```

### Step 2: Device polls the token endpoint

```http
POST /token HTTP/1.1
Host: authorization-server.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:device_code
&device_code=GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS
&client_id=1406020730
```

While the user has not finished, the server returns an error. Poll no faster
than `interval` seconds:

| Error | Meaning / action |
|-------|------------------|
| `authorization_pending` | Keep polling at the current interval |
| `slow_down` | Increase the interval by 5 seconds |
| `access_denied` | User declined; stop |
| `expired_token` | `device_code` expired; restart the flow |

On success the token endpoint returns the standard token response.

## Refresh Token

Used to obtain a new access token when the current one expires, without
involving the user again. RFC 6749 §6.

```http
POST /token HTTP/1.1
Host: authorization-server.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
&client_id=s6BhdRkqt3
```

OAuth 2.1 guidance: refresh tokens for public clients must be either
sender-constrained (e.g. DPoP) or **rotated** — each use returns a new refresh
token and invalidates the previous one, so a stolen token is detectable.

## Deprecated Grants

### Implicit (`response_type=token`) — removed in OAuth 2.1

Returned an access token directly in the redirect URI fragment, with no code
exchange. It exposes tokens in the browser history and URL and cannot use PKCE.
Use Authorization Code + PKCE instead — modern browsers support CORS, so SPAs
can call the token endpoint directly.

### Resource Owner Password Credentials (ROPC) — removed in OAuth 2.1

The client collected the user's username and password and sent them to the token
endpoint. This defeats the core purpose of OAuth (never sharing credentials with
the client), blocks MFA and federated login, and trains users to enter
credentials into third-party apps. Migrate to Authorization Code + PKCE.

## OAuth 2.1 Summary

- PKCE required for all authorization code clients (public and confidential).
- Implicit and ROPC grants are removed.
- Redirect URIs must be compared by **exact string match**.
- Refresh tokens for public clients must be sender-constrained or rotated.
- Access tokens must not be sent in the query string.
