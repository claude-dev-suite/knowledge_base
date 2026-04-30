# Bitcoin Connect - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/webln`.
> Canonical source: Bitcoin Connect (https://github.com/getAlby/bitcoin-connect)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/webln/SKILL.md

## Concept

Bitcoin Connect is a connector library that abstracts over the heterogeneous
wallet-connection landscape: WebLN extensions, Nostr Wallet Connect (NWC),
LNC (Lightning Node Connect for LND), and Alby OAuth. It exposes a single
`window.webln` shim regardless of the underlying transport, and ships
drop-in web components that render a wallet picker. From the page's
perspective, every transport looks like WebLN; from the user's perspective,
they pick the connector that fits their wallet.

## Walkthrough / mechanics

Bitcoin Connect's core abstraction:

```
                       Page calls window.webln.sendPayment(...)
                                       |
                                       v
                        Bitcoin Connect dispatcher
              /            |                   |              \
   webln-ext / NWC / LNC (LND) / Alby OAuth / NIP-55 mobile signer
```

When the page imports the library and renders `<bc-button>`, the user
clicks and is presented with a modal of available connectors:

1. "Use my browser extension" - calls the extension's existing
   `window.webln`.
2. "Connect wallet via Nostr (NWC)" - paste a `nostr+walletconnect://` URI.
   BC stores the URI, opens a relay subscription, exposes a NWC-backed
   `window.webln` shim.
3. "Connect via Alby" - OAuth flow against `getalby.com`, returns access
   token, BC uses Alby's REST endpoints.
4. "LNC (LND)" - paste a pairing phrase, BC opens a Lightning-Node-Connect
   noise tunnel.

Once a connector is chosen, BC sets `window.webln` to the shim. Pages
that already use raw WebLN need zero changes.

State and persistence:

- Selected connector + credentials encrypted with a password (or stored
  in browser localStorage as plain JSON if no password).
- Auto-reconnect on next page load.
- `bitcoinConnect.disconnect()` clears state.

## Worked example

HTML+JS integration:

```html
<script type="module"
  src="https://esm.sh/@getalby/bitcoin-connect@3"></script>

<bc-button onConnected="onConnected"></bc-button>

<button id="pay" disabled>Pay 100 sats</button>

<script>
  import { onConnected, requestProvider } from '@getalby/bitcoin-connect';
  onConnected(async () => {
    document.getElementById('pay').disabled = false;
  });
  document.getElementById('pay').addEventListener('click', async () => {
    const provider = await requestProvider();
    const r = await provider.sendPayment('lnbc1u1pr...');
    console.log('preimage', r.preimage);
  });
</script>
```

NWC connector flow under the hood:

```
User pastes nostr+walletconnect://<wallet_pubkey>?relay=wss://relay.io&secret=<hex>
BC stores secret, computes app_pubkey = secret * G.
BC opens WS connection to wss://relay.io.
On sendPayment(invoice):
  payload = JSON.stringify({ method: 'pay_invoice',
                              params: { invoice } })
  cipher = NIP-04(secret, wallet_pubkey).encrypt(payload)
  event = { kind: 23194, pubkey: app_pubkey,
            tags: [['p', wallet_pubkey]],
            content: cipher,
            created_at: now() }
  signed = sign(secret, event)
  publish(signed) to wss://relay.io
  await response kind=23195 with p-tag matching event id
  decrypt -> { result: { preimage: '...' } }
  return { preimage }
```

The page only sees `provider.sendPayment(...)` returning a preimage; the
NWC encryption + relay round-trip is hidden.

## Common bugs / pitfalls

- Page assumes `window.webln` exists synchronously on load. BC sets it
  asynchronously after user picks. Use `onConnected` or `await
  requestProvider()`.
- Multiple BC instances on one page (e.g. via two iframes) race for the
  `window.webln` slot. Ensure single import.
- LocalStorage password forgotten -> credentials unrecoverable. BC does
  not have a recovery flow.
- Latency: NWC connector adds 1-3 s round trip; users may double-click.
  Disable button during in-flight calls.
- Mobile Safari blocks third-party storage in iframes; BC fails silently.
  Use first-party page origin for embedding.
- Some pages pass NWC URIs through URL params - never put a `secret=`
  in browser history.

## References

- Bitcoin Connect repo: https://github.com/getAlby/bitcoin-connect
- NWC NIP-47: https://github.com/nostr-protocol/nips/blob/master/47.md
- Lightning Node Connect spec: https://github.com/lightninglabs/lightning-node-connect
- Alby OAuth API docs
