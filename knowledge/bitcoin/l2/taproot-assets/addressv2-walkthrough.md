# Taproot Assets AddressV2 Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/taproot-assets`.
> Canonical source: tapd v0.7 release notes (Dec 2025) and BIP-PROTO-006
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/taproot-assets/SKILL.md

## Concept

AddressV2 is the static, reusable Taproot Assets address format introduced in tapd v0.7
(December 2025). The original AddressV1 was a one-shot per-payment address bound to a
specific asset_id and amount; AddressV2 is reusable, supports **grouped assets**, and
allows **zero-amount-friendly** receive flows (the receiver picks the amount per
incoming payment, or accepts any amount). This solves the practical UX problem of
needing a fresh QR per stablecoin invoice.

## Walkthrough / mechanics

### AddressV1 (legacy)

```
AddressV1 = bech32m(
    asset_id || group_key? || script_key || internal_key || amount || asset_version
)
```

Bound to a single payment of a single amount. Same address re-used = wallet can detect
duplicates, but reusing leaks privacy and complicates accounting. Receiver also must
generate a fresh address per asset/amount combo.

### AddressV2 structure

```
AddressV2 = bech32m(
    version_marker = v2,
    universe_pubkey,                 // for proof distribution
    address_id,                      // 32-byte handle stored at universe
    optional asset_specifier {       // can be:
       single_asset_id          OR
       group_key                OR
       universal (any asset)
    }
)
```

Key changes:
- The address itself is a *handle* into the issuer/receiver's universe, not the full
  state.
- Receiver decides, per-incoming-payment, which amount and which output script to use.
- Group-key support lets a single address receive any asset under a group (e.g., a
  set of related issuance batches).
- "Universal" addresses accept *any* TAP asset.

### Send-side flow

```
Sender wants to pay 50 USDT to addr_v2:
  1. Sender wallet decodes AddressV2.
  2. Wallet contacts the embedded universe_pubkey's universe service.
  3. Wallet POSTs an "intent": {address_id, asset=USDT, amount=50, sender_proof_chain}.
  4. Universe forwards intent to the receiver's tapd over its long-poll connection.
  5. Receiver's tapd evaluates: "do I want to accept this?".
       - May reject (e.g., wrong asset for a single-asset address).
       - May accept and respond with a fresh script_key for the output.
  6. Sender wallet finalises Bitcoin tx with the receiver's just-issued script_key.
  7. Tx broadcast.
  8. Receiver receives proof via the universe.
```

The handshake means the address is reusable (no script_key on the chain is reused) and
the receiver can apply policy at receive time (rate limiting, KYC, accept-list).

### Zero-amount-friendly flow

An AddressV2 marked "amount=0/any" lets sender propose any amount. Receiver's tapd can
auto-accept up to a per-day cap, useful for:
- Tipping addresses (receiver doesn't know the amount).
- Refund returns (sender uses receiver's static "refund address").

## Worked example

Bob runs an online shop. He publishes one AddressV2 in his footer:

```
addr_v2 = "tapaddrv2_xxx..." (group_key = USDC-and-USDT, amount = 0)
```

Customer Alice wants to buy a 10 USDT item:

```
T+0     Alice's wallet decodes addr_v2 -> universe = bob.universe.example.
T+50ms  Wallet POST intent: {addr_id, asset=USDT, amount=10}.
T+100ms Bob's tapd auto-accepts (matches group, under per-customer cap).
        tapd responds: script_key = sk_xxx, internal_key = ik_xxx.
T+150ms Wallet builds Bitcoin tx with anchor output committing
        [USDT/10 -> sk_xxx] under MS-SMT.
T+200ms Tx broadcast.
T+10min Tx confirmed.  Bob's tapd fetches proof from universe, validates,
        credits Bob's account with 10 USDT.
```

Customer Carlos sends an unsolicited tip in USDC instead:

```
T+0     Carlos posts intent: {addr_id, asset=USDC, amount=5}.
T+100ms Bob's tapd accepts (group includes USDC, amount > 0 allowed).
        Returns fresh script_key for USDC.
T+200ms Tx built and sent.
```

Bob never had to publish two addresses or pre-coordinate the asset.

## Trade-offs and security

- **Universe reliance**: AddressV2 effectively requires an online universe service to
  mediate. If the universe is down at send time, the sender can fall back to a fresh
  AddressV1 (if the receiver supports it) or wait. This is a UX regression for fully-
  offline scenarios.
- **Receiver policy enforcement**: receiver can refuse a payment via the universe
  before broadcast, but cannot refuse after broadcast. Adversarial sender can ignore
  the handshake and broadcast anyway with a known script_key (still requires getting
  the key somehow); spec says receiver SHOULD use ephemeral keys per intent.
- **Privacy**: AddressV2 reduces address-reuse on chain (each tx uses a fresh
  script_key) but the universe sees all payment intents in plaintext. A privacy-
  conscious receiver runs their own universe.
- **Multi-asset receive**: simplifies UX but complicates accounting; downstream systems
  must handle "one address, many assets".
- **No backwards compatibility**: tapd < 0.7 cannot decode AddressV2.
  Issuers should publish both formats during transition.

## References

- tapd v0.7 release notes - https://github.com/lightninglabs/taproot-assets/releases/tag/v0.7.0
- BIP-PROTO-006 (AddressV2 spec) - https://github.com/lightninglabs/taproot-assets/blob/main/docs/bip-proto-006.md
- Lightning Labs blog, "Address V2 unlocks set-and-forget" (Dec 2025)
