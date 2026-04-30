# Lightning Address Server Implementation - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/lightning-address`.
> Canonical source: LUD-16 + LUD-06 + LUD-09 + LUD-12
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/lightning-address/SKILL.md

## Concept

A Lightning Address server is essentially a thin HTTPS proxy that turns
`/.well-known/lnurlp/{user}` requests into invoices on a backend node.
The hard parts are not in the LNURL plumbing but in the operational
choices: custody model, multi-tenancy, rate limiting, payment-routing
back to the user account, and abuse defenses.

## Walkthrough / mechanics

Minimal architecture:

```
Internet -> HTTPS LB (TLS terminator) -> Web app (Express/Flask/Go)
                                               |
                                               v
                                       Lightning node RPC
                                       (LND/CLN/phoenixd/LDK)
```

Endpoints:

```
GET  /.well-known/lnurlp/{user}              -> descriptor
GET  /.well-known/lnurlp/{user}/cb?amount=N  -> invoice (callback)
GET  /.well-known/lnurlp/{user}/verify/{id}  -> LUD-21 status
```

Per-request flow on callback:

1. Look up user account by local-part (DB query).
2. Validate amount is within `[minSendable, maxSendable]`.
3. Build metadata string deterministically (must match descriptor):

```
metadata = json_array([
  ["text/identifier", f"{user}@{domain}"],
  ["text/plain", f"Sats for {user}"]
])
```

4. Compute `description_hash = sha256(metadata + payerdata_str)`.
5. Call backend `addinvoice` with `description_hash`, amount, expiry.
6. Persist `(payment_hash -> user_id)` mapping for routing settled funds.
7. Return JSON with `pr`, `successAction`, `verify` URL.

For custodial setups, on `invoice_paid` event from the backend node
(LND `SubscribeInvoices`, CLN `waitanyinvoice`), credit the user balance.

For non-custodial setups, the server typically operates an LSP that
opens / splices a JIT channel to the user's mobile wallet on incoming
payments (BLIP-52 flow).

## Worked example

Express.js skeleton (illustrative):

```
app.get('/.well-known/lnurlp/:user', async (req, res) => {
  const user = req.params.user.toLowerCase();
  if (!/^[a-z0-9._+-]+$/.test(user)) return res.status(400).end();
  const acct = await db.users.findByLocal(user);
  if (!acct) return res.status(404).end();
  res.json({
    tag: 'payRequest',
    callback: `https://${DOMAIN}/.well-known/lnurlp/${user}/cb`,
    minSendable: acct.min_msat ?? 1000,
    maxSendable: acct.max_msat ?? 10_000_000_000,
    metadata: buildMetadata(user, acct),
    commentAllowed: 200,
  });
});

app.get('/.well-known/lnurlp/:user/cb', async (req, res) => {
  const user = req.params.user.toLowerCase();
  const amountMsat = Number(req.query.amount);
  const acct = await db.users.findByLocal(user);
  if (!acct) return res.status(404).end();
  if (amountMsat < (acct.min_msat ?? 1000) ||
      amountMsat > (acct.max_msat ?? 10e9))
    return res.json({ status:'ERROR', reason:'amount out of bounds' });
  const meta = buildMetadata(user, acct);
  const descHash = sha256(meta + (req.query.payerdata ?? ''));
  const inv = await lnd.addInvoice({
    value_msat: amountMsat,
    description_hash: descHash,
    expiry: 600,
  });
  await db.invoices.insert({ payment_hash: inv.r_hash, user_id: acct.id });
  res.json({
    pr: inv.payment_request,
    verify: `https://${DOMAIN}/.well-known/lnurlp/${user}/verify/${inv.r_hash}`,
    successAction: { tag:'message', message:`Sats for ${user}!` },
    routes: []
  });
});
```

## Common bugs / pitfalls

- `description_hash` mismatch: server returns descriptor with metadata X but
  signs invoice with metadata Y after re-serialization. Always cache the
  exact metadata STRING used in the descriptor and reuse it byte-for-byte.
- Rate limiting absent: an attacker can request thousands of invoices a
  second to fill the backend node's HTLC-slot budget. Enforce
  per-IP + per-user rate limits.
- Per-request invoice expiry too short. < 60 s makes mobile + LSP flows
  unreliable. Recommended: 600-1800 s.
- Custodial accounting drift: failing to log preimage release in a
  durable store. On crash, server may double-credit or miss credits.
- TLS cert renewal failure (Let's Encrypt) silently breaks discovery.
  Monitor TLS expiry; many wallets refuse self-signed certs.
- IPv6 vs IPv4 misconfig. Some wallets resolve only AAAA; serve both.
- Unicode in metadata: emoji-rich descriptions break SHA-256 if the server
  serializes JSON differently than it returns it. Pin canonical encoding.

## References

- LUD-16: https://github.com/lnurl/luds/blob/luds/16.md
- LNbits Lightning Address extension source
- BTCPay Server LN Address plugin source
- ACINQ phoenixd hosting guide
