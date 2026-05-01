# Loop Submarine-Swap Mechanics - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/loop-pool-lit`.
> Canonical source: https://docs.lightning.engineering/the-lightning-network/loop
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/loop-pool-lit/SKILL.md

## Concept

Lightning Labs **Loop** is a service that swaps Lightning balance for
on-chain BTC and vice versa. It uses **submarine swaps** (HTLC swaps
across layers): the user holds an HTLC on Lightning, the Loop service
holds the matching HTLC on Bitcoin. The preimage reveal on one side
forces the matching reveal on the other, atomically swapping value.

- **Loop Out**: user pays Lightning, receives BTC on chain.
  Used to drain inbound liquidity / deposit cold storage.
- **Loop In**: user pays BTC on chain, receives Lightning balance.
  Used to refill outbound liquidity.

## Walkthrough / mechanics

### Loop Out flow

1. User opens Loop Out request specifying:
   - Amount X sat.
   - On-chain destination address.
2. Loop server returns:
   - **Lightning invoice** for X + fee.
   - Loop server's **HTLC public key** for the on-chain script.
   - Refund pubkey for the user.
   - Timeout block heights.
3. User constructs the on-chain HTLC script:
   ```
   IF
     <SHA256 H> EQUALVERIFY <Loop pubkey> CHECKSIG
   ELSE
     <timeout> CHECKLOCKTIMEVERIFY DROP <user_refund_pubkey> CHECKSIG
   ENDIF
   ```
   Server prepares this and inscribes a P2WSH (or P2TR) output to it,
   funded by the server with X sat.
4. Server publishes the on-chain HTLC.
5. User pays Lightning invoice. Loop server, on receiving the LN
   payment, knows the preimage `R` (because they generated it).
6. **Race**: server reveals preimage to spend on-chain HTLC to user's
   destination address (server cannot wait too long because user might
   reveal preimage on chain after the LN settles).
7. User waits for the on-chain HTLC to confirm; verifies preimage
   matches the LN invoice's payment_hash.

If server fails to act, user gets refund via Lightning's failure
mechanism (LN HTLC times out, refunding user). User does NOT need to
act on chain.

### Loop In flow

1. User requests Loop In for X sat.
2. Server returns Lightning invoice (will be settled by user funding
   on-chain HTLC) + on-chain HTLC script tagged to user.
3. User funds the on-chain HTLC output with X + fee.
4. Server, on detecting on-chain HTLC, pays user's invoice over
   Lightning. User reveals preimage by settling.
5. Server uses preimage to claim the on-chain HTLC.

### Confirmation requirements

- **Loop Out**: server typically uses 1-conf for the on-chain HTLC
  (zero-conf opt-in available). User pays LN invoice, server reveals
  preimage on chain.
- **Loop In**: server requires 1-3 confs of user's on-chain HTLC
  before paying LN invoice; otherwise reorg risk to server.

### Fees

Loop charges:
- Service fee (e.g. 0.1-0.5 % of swap amount).
- On-chain mining fee.
- Lightning routing fee (server's outbound to user).

User specifies max fee budget; Loop fails if quote exceeds.

### Nested swap-bumping

If on-chain HTLC's fee is too low, Loop supports CPFP via the user's
refund anchor (with anchor-channels enabled). Replacement-cycling-style
attacks are mitigated by short timeouts + replacement-by-fee on user
side.

## Worked example

Bob has too much inbound liquidity (channels are mostly outbound).
He uses Loop Out to convert 0.05 LN-BTC to on-chain.

```
1. POST /loop-out {amount: 5000000, dest_addr: bc1q...}
   -> {invoice: lnbc..., quote_fee: 5000 sat, htlc_pubkey: P_loop, ...}
2. Server publishes on-chain HTLC tx with output:
     P2WSH(IF SHA256 H EQ P_loop CHECKSIG ELSE CSV refund ENDIF) 5,000,000 sat
3. After 1 BTC conf, Bob pays LN invoice for 5005000 sat (or fewer
   if successful routing).
4. Loop server, receiving LN payment, derives preimage R.
5. Server broadcasts BTC tx spending HTLC to Bob's bc1q...
6. Bob waits 1 conf -> 5000000 sat on chain.
```

## Common pitfalls

- **On-chain fee timing**: at high BTC fee rate, on-chain HTLC's
  refund tx may be too cheap; user needs RBF/CPFP capable wallet to
  bump.
- **Server downtime mid-swap**: Loop In requires server to pay LN
  invoice; if server crashes, user must wait for on-chain HTLC timeout
  to refund (user's funds locked for several blocks).
- **Preimage exposure**: a third party watching chain can see the
  preimage as soon as Loop reveals it. They cannot do anything with it
  (Lightning HTLC is already settled), but it's a small information
  leak.
- **Fee market spike**: between quote and execution, fees may spike;
  Loop locks fee at quote time but server may not honor at full rate
  if too far above market.
- **Replacement cycling attacks** (general HTLC topic): Loop's HTLCs
  with short timeouts are theoretically vulnerable. v2 anchor-channel
  swaps mitigate.

## References

- Lightning Labs Loop documentation.
- "Submarine Swaps" — Alex Bosworth 2018 explanation.
- BOLT-03 HTLC structure.
