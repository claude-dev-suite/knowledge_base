# Spark Lightning Bridge Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/spark`.
> Canonical source: https://docs.spark.money/lightning
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/spark/SKILL.md

## Concept

Spark integrates with Lightning so that Spark balance can be used to pay an LN invoice
and LN payment can be received as Spark balance. Bridging is provided by the SO
collective, leveraging Lightspark's existing LN routing infrastructure. Unlike a generic
custodial bridge, Spark's design preserves the user's exit guarantee: if the LN payment
fails, the leaf is restored; if it succeeds, the leaf is debited with FROST-signed
authority.

## Walkthrough / mechanics

```
+----------+         +-------------+        +------------+
|  Spark   |  HTLC   |     SO      | LN HTLC|  Lightning |
|  Wallet  +<------->+  Collective +<------>+   Network  |
+----------+         +-------------+        +------------+
       Spark side                Lightning side
       (FROST leaves)            (BOLT11/BOLT12)
```

### Pay LN from Spark balance

1. Alice's wallet calls `payLightningInvoice(invoice)`.
2. Coordinator (Lightspark API) parses BOLT11, extracts payment_hash H and amount A.
3. Coordinator asks SOs to **conditionally debit** Alice's leaf by A:
   - SOs FROST-pre-sign a new tree state where Alice's leaf is reduced by A, but the
     change is conditional on the LN HTLC settling.
   - The pre-signed state is held in escrow.
4. Lightspark's LN node sends an HTLC for amount A with payment_hash H over the LN
   graph.
5. On settlement (preimage R revealed):
   - LN node receives the funds.
   - Coordinator publishes preimage R to SOs.
   - SOs commit the leaf debit (FROST sign final tree state).
6. On HTLC timeout:
   - LN HTLC reverts.
   - SOs invalidate the conditional state; Alice's leaf is unchanged.

### Receive LN to Spark balance

1. Bob's wallet asks coordinator to issue an LN invoice for B sats payable to Bob's
   Spark leaf.
2. Coordinator generates a payment_hash H (preimage held by Lightspark), publishes
   BOLT11 invoice for B sats.
3. Payer pays the invoice via LN. Lightspark's LN node receives the HTLC.
4. Coordinator asks SOs to FROST-sign a leaf-creation transaction for Bob worth B.
5. SOs sign; Bob's wallet receives the new leaf.
6. Coordinator releases preimage R to settle the LN HTLC, getting reimbursed for the B
   sats it forwarded.

### Atomicity guarantees

The two halves of the bridge are tied via the LN preimage:

- Pay-out: SOs commit leaf debit *only* after preimage is published (LN settled).
- Receive: SOs create new leaf *before* preimage is released (LN settles only after
  Spark side is ready).

If anything fails mid-way the user is back to the pre-flow state. There is a brief
window during pay-out where the LN payment has settled but the leaf debit has not yet
been committed; SOs cover this with their own working capital.

## Worked example

Alice pays a 50,000 sat coffee invoice:

```
T+0     Alice scans QR -> wallet has invoice.  Calls Spark API.
T+10ms  Coordinator hashlock H = ..., amount 50000.
T+20ms  SOs pre-sign conditional leaf state: alice_leaf 100000 -> 50000.
T+30ms  Lightspark LN node sends HTLC out to merchant via 2 hops.
T+450ms HTLC settled, merchant returns preimage R.
T+460ms Coordinator publishes R to SOs.
T+475ms SOs FROST-sign final leaf state.  alice_leaf is now 50000 sats.
T+480ms Alice's wallet shows updated balance.

Total user-perceived latency: ~500ms (LN-bound).
```

If LN routing fails (e.g., insufficient liquidity at hop 2):

```
T+0   ... T+450  same as above, but HTLC fails at hop 2.
T+460  Coordinator notifies SOs: invalidate conditional state.
T+470  SOs delete the pre-signed conditional update.
T+475  Alice's leaf still 100000 sats.  Wallet shows "payment failed".
```

## Trade-offs and security

- **Trust on Lightspark LN node**: pay-out requires trusting that Lightspark will not
  pocket the HTLC after preimage release without committing the leaf debit. The window
  is small (single-digit ms) but does exist. Mitigated by Lightspark's SLA + fact that
  user can prove fraud via the SOs' signed conditional state.
- **Preimage equivocation**: a malicious LN node could publish preimage R to settle the
  HTLC but withhold from SOs to delay leaf debit. Coordinator monitors LN graph and
  forces commit on detection. SOs are also required to commit within a fixed deadline.
- **Asset support on LN**: only sats today. Asset-on-Spark transactions (post-roadmap)
  will require asset-on-LN protocols (Taproot Assets, RGB) for through-routing.
- **Privacy**: payer/payee unlinked on the LN side via onion routing, but Spark
  coordinator sees the full Spark-leg metadata (sender leaf, amount, invoice).
- **Compared to LN custodial wallets**: similar UX, but Spark's k-of-n SO security
  model is stronger than 1-of-1 custodial. User can always exit on-chain.

## References

- Spark Lightning integration docs - https://docs.spark.money/lightning
- Lightspark blog, "Spark + Lightning" launch (2025)
- BOLT11 / BOLT12 - https://github.com/lightning/bolts
- Lightspark UMA - https://www.uma.me/
