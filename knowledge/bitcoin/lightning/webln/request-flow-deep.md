# WebLN Request Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/webln`.
> Canonical source: WebLN Guide (https://www.webln.guide/)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/webln/SKILL.md

## Concept

WebLN is a `window.webln` JS object injected by a wallet extension or
in-page provider, exposing a small set of asynchronous methods. The flow
is permission-gated: the page calls `enable()` to request access, then
calls a method, the wallet UI prompts the user, and the result resolves
or rejects. Unlike LNURL/NWC there is no transport layer to design - the
browser is the bus, and the wallet is the in-process or content-script
endpoint.

## Walkthrough / mechanics

Lifecycle of a WebLN call:

```
[Page]                  [WebLN provider]                [Wallet]
   |                          |                            |
   |-- detect window.webln -->|                            |
   |-- enable() ------------->| origin shown to user -->   |
   |                          |<-- approve / deny ---------|
   |<-- resolve(undefined) ---|                            |
   |                                                       |
   |-- sendPayment(invoice)-->| invoice shown, fee shown ->|
   |                          |<-- user confirms or cancels|
   |                          | wallet pays invoice        |
   |                          |<-- preimage on settle -----|
   |<-- { preimage } ---------|                            |
```

Permissions are scoped per (origin, method). Most providers cache an
"enabled" state per origin for the session; some (Alby) expose
auto-approve rules per method + per amount cap.

Method semantics:

- `enable()` -> returns once the user grants. MUST be called from a user
  gesture (click handler) or browsers may block.
- `getInfo()` -> returns wallet identity. Sensitive: leaks node alias /
  pubkey to the page; some providers return only a stub.
- `sendPayment(bolt11)` -> resolves with `{ preimage }` on settle, rejects
  with `code` and `message` on user denial, fee budget overflow, or route
  failure.
- `makeInvoice({ amount, defaultMemo })` -> returns BOLT11.
- `signMessage(msg)` -> returns LN-node signed message (lnd-style zbase32).
- `keysend({ destination, amount, customRecords })` -> spontaneous payment.

## Worked example

```js
async function tipAuthor(invoice) {
  if (typeof window.webln === 'undefined') {
    return alert('No WebLN provider. Install Alby or use Bitcoin Connect.');
  }
  try {
    await window.webln.enable();      // user gesture required
  } catch (e) {
    return alert('Wallet access denied');
  }
  let info;
  try {
    info = await window.webln.getInfo();
    console.log('Connected to', info.node?.alias);
  } catch { /* some providers stub this */ }

  try {
    const r = await window.webln.sendPayment(invoice);
    // r = { preimage: 'a1b2...' }
    if (sha256(hexDecode(r.preimage)) !== bolt11.parse(invoice).payment_hash) {
      throw new Error('preimage does not match');
    }
    console.log('paid, preimage =', r.preimage);
  } catch (e) {
    // common codes: USER_REJECTED, INSUFFICIENT_BALANCE, ROUTING_ERROR
    console.error('payment failed', e);
  }
}

document.querySelector('#tip').addEventListener('click', () =>
  tipAuthor('lnbc500u1p3...'));
```

Server-side, the merchant verifies the payment by polling its own LN
node for invoice settled status (or LUD-21 if the invoice came via
LNURL). The page-supplied preimage is for UX feedback only; never trust
the page's claim that payment occurred.

## Common bugs / pitfalls

- `enable()` outside user gesture: Chrome / Firefox extension content
  scripts can block injection if not initiated from `click` / `keydown`.
- Hard-coded msat vs sat: `makeInvoice.amount` is **sat**, but BOLT11
  encodes msat. Some providers misinterpret floats.
- Auto-pay loops: page calls `sendPayment` in `setInterval`. Wallet
  rate-limits; user gets prompt fatigue. Always require explicit user
  click for each payment.
- Memory leak via listeners: pages that subscribe to provider events
  (`webln.on('paid')`) without cleanup leak across SPA route changes.
- Provider feature drift: `keysend` is optional; `signMessage` returns
  different formats across Alby (zbase32), Joule (DER hex), Mutiny
  (compact). Check `info.methods` first.
- Phishing: a malicious page uses `window.webln.signMessage` to gather
  identity proofs without clear UX. Wallets must surface the message
  body, not just origin.

## References

- WebLN Guide: https://www.webln.guide/
- Alby Browser Extension developer docs
- Bitcoin Connect library source: https://github.com/getAlby/bitcoin-connect
- WebLN type definitions (`@webbtc/webln-types`)
