# Mobile Lightning Wallet Uptime Tradeoffs - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/consumer-wallets`.
> Canonical source: https://github.com/lightningdevkit/rust-lightning (LDK mobile docs)
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/consumer-wallets/SKILL.md

## Concept

Lightning's HTLC-based protocol assumes both peers are online during
payment routing and channel monitoring. Mobile wallets break this
assumption: phones spend most of their time asleep, on flaky cellular,
or offline. Various design tradeoffs are made to deliver Lightning UX
on mobile while still preserving security (force-close monitoring) and
reliability (incoming payment receipt).

## Walkthrough / mechanics

### Core uptime requirements

A self-custodial Lightning node ideally:

1. Maintains TCP connection to channel peers (heartbeat ~60s).
2. Watches Bitcoin chain for force-close attempts.
3. Responds to incoming HTLCs within tens of seconds.
4. Updates revocation states promptly.

A mobile phone violates 1-3 unpredictably (background, sleep, offline).

### Tradeoff dimensions

| Dimension | Always-on Approach | Mobile-friendly Approach |
|-----------|-------------------|--------------------------|
| Receive payments | Direct HTLC accept | Push notification + on-demand wakeup |
| Channel monitoring | Self-watching | Watchtower (third-party) |
| Force-close detection | Real-time | On wallet open |
| Channel opens | Manual | LSP automated |

### Mobile design patterns

#### Pattern A: Push notification (Phoenix, Breez)

- LSP routes inbound payment to user.
- LSP can't fully accept without user wallet awake (LDK signer must
  sign).
- LSP holds HTLC briefly; sends push notification (FCM/APNS) to wallet.
- Wallet wakes, signs commitment update, payment completes.
- Latency: 5-30 seconds depending on push delivery.

Tradeoff: requires Apple/Google push servers; small PII leak. Push
fails -> payment fails.

#### Pattern B: HODL invoice (Strike, Wallet of Satoshi)

- Custodial provider holds HTLC.
- User opens app, provider settles, user sees update.
- Latency: bounded by app open time.

Tradeoff: custodial only. Self-custodial wallets can't use this
naively because they don't sign for the HTLC.

#### Pattern C: Async receive (LDK + LSP coordination)

- LSP holds HTLC up to several minutes (CLTV).
- Sends silent push.
- Wallet wakes in background, completes payment.
- If too slow, LSP fails the HTLC.

Most modern self-custodial mobile wallets use this pattern.

### Channel monitoring on mobile

- **Watchtowers**: third-party services watch the chain on user's
  behalf. User uploads "justice transactions" pre-signed; tower
  broadcasts on revocation event detection.
- **Self-watch on app open**: wallet checks chain when launched.
  Risky if user doesn't open app for >2 weeks (CSV expires).

LDK's `ChainMonitor` + `BackgroundProcessor` patterns let phones do
limited self-watching when foregrounded; watchtower is fallback.

### Background execution limits

- iOS: ~30s background time per app launch; 1 minute background
  refresh per ~hour.
- Android: more flexible but battery-saver kills background tasks.

Real-world: assume <1 minute of background time per hour. Push
notifications wake the app for short bursts.

### Push notification reliability

- FCM/APNS deliver ~95-99% within 30s, but failures and delays exist.
- Critical wakeups for payment completion may miss the CLTV window.
- Latency-sensitive payments (e.g. POS transactions) require user
  to have app open.

### Battery considerations

- Continuous Lightning monitoring drains battery significantly.
- Background TCP keepalive every 60s can consume 5-10% battery/day.
- Mobile wallets reduce keepalive frequency, accept some lag.

## Worked example

Phoenix wallet receiving a payment while phone is in pocket:

```
1. Sender pays Phoenix's invoice (lnbc...).
2. ACINQ LSP receives HTLC at Phoenix-LSP shared channel.
3. LSP sends silent push: "incoming HTLC, sign now".
4. Phone receives push -> Phoenix wakes (~5-10s).
5. Phoenix signs commitment update, completes HTLC -> sender succeeds.
6. Phoenix shows local notification: "+5,000 sat from Bob".
7. After ~30s, Phoenix puts itself back to sleep.
```

If push delivery fails:

```
1. Sender pays invoice.
2. LSP holds HTLC.
3. Push fails (phone offline, push server down).
4. After ~60s, LSP times out HTLC; sender sees payment failure.
5. Phoenix never knew about the payment.
```

## Common pitfalls

- **Force-close while phone is off for >2 weeks**: revocation period
  lapses. Use a watchtower as backup.
- **iOS background limitations**: even silent pushes don't always wake
  the app; iOS may throttle. Test extensively on real devices.
- **Battery drain from frequent wakes**: tune polling carefully;
  too aggressive battery; too lax = missed payments.
- **App store policy**: foreground notifications for crypto are subject
  to app review.
- **Outdoors / cellular**: poor connectivity delays HTLC signing past
  CLTV; pre-set CLTV margin generously.

## References

- LDK documentation on mobile (lightningdevkit.org).
- ACINQ "Phoenix on Mobile" blog series.
- BOLT-04 onion + BOLT-02 timing requirements.
- Apple Push Notification Service APNS docs.
