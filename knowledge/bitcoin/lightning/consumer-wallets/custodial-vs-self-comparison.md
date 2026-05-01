# Lightning Consumer Wallets: Custodial vs Self - Comparison Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/consumer-wallets`.
> Canonical source: https://github.com/satoshilabs/wallet-comparison
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/consumer-wallets/SKILL.md

## Concept

Lightning consumer wallets fall on a spectrum from fully **custodial**
(provider holds funds) to fully **self-custodial** (user holds funds in
on-device channels). Each model trades off security, privacy, complexity,
and reliability differently. This article compares the major points
of difference.

## Walkthrough / mechanics

### Custody spectrum

| Tier | Model | Examples | Custody | Key holder |
|------|-------|----------|---------|------------|
| 1 | Fully custodial | Wallet of Satoshi, Strike, Cash App | Provider | Provider |
| 2 | Federated custodial | Fedimint (Mutiny, etc.) | Federation | Federation guardians |
| 3 | LSP-assisted self-custodial | Phoenix, Breez | User (with provider help) | User |
| 4 | Fully self-custodial | Zeus + own LND, Mutiny+ | User | User |

### Tier 1: Fully custodial

- User has username/email login.
- Provider holds keys, channels, funds.
- User signs payments via app authentication.

Pros:
- Smooth UX: instant onboarding, no channel opens.
- No on-chain fees for user.
- Easy multi-device.

Cons:
- Provider can confiscate funds (#NotYourKeysNotYourCoins).
- Subject to KYC, AML.
- Single point of failure.

### Tier 2: Federated custodial

- Federation of guardians holds keys (Fedimint).
- User has Chaumian e-cash notes; mints redeemable for sats.
- Lightning gateway swaps e-cash for routes.

Pros:
- Privacy: federation cannot link payments.
- Reduced trust vs single custodian (need quorum to steal).

Cons:
- Federation collusion still possible.
- Smaller network effects than custodial Tier 1.
- Federation downtime = no payments.

### Tier 3: LSP-assisted self-custodial

- User runs Lightning node on phone (Phoenix, Breez SDK).
- LSP (Lightning Service Provider) opens channels to user on demand.
- LSP doesn't hold user's funds; user has key.

Pros:
- Self-custodial: user can force-close to recover.
- Smooth UX via LSP-managed channel opens.
- Reasonable privacy.

Cons:
- LSP can refuse to forward (DoS).
- On-chain fees for opens (~5,000-50,000 sat per channel).
- LSP knows payment metadata (counterparty, amounts).

### Tier 4: Fully self-custodial

- User runs LND/CLN/LDK on own hardware (or VPS).
- Manages channels, liquidity, watchtowers, backup.

Pros:
- Maximum sovereignty.
- Maximum privacy (no LSP middleman).
- Can route + earn fees.

Cons:
- Operational complexity (channel management, watchtowers).
- 24/7 uptime required for inbound payments.
- Recovery is fragile (Static Channel Backups, on-chain ops).

### Privacy comparison

| Property | Tier 1 | Tier 2 | Tier 3 | Tier 4 |
|----------|--------|--------|--------|--------|
| Provider sees: amounts | Yes | Partial (mint) | Yes (LSP) | No |
| Provider sees: counterparty | Yes | Partial | Partial | No |
| Provider sees: identity | Yes (KYC) | Pseudonymous | Pseudonymous | No |
| KYC required | Often | Sometimes | Rarely | No |

### Reliability comparison

| Property | Tier 1 | Tier 2 | Tier 3 | Tier 4 |
|----------|--------|--------|--------|--------|
| Uptime dependence | Provider | Federation quorum | LSP | Self |
| Recovery on phone loss | Account login | Seed (federation handles) | Seed + SCB | Seed + on-chain ops |
| Receive while offline | Yes (provider holds) | Yes | Yes (LSP holds) | No (must be online) |
| Force-close attack vector | None | None (custodial) | LSP misbehaves | Self-monitor |

## Worked example

Three users with different needs:

**Alice (general consumer)**: wants to send/receive Lightning payments
quickly without thinking about technicalities. Best fit: **Tier 1 or 3**.
Tier 1 (Strike) optimal for simplicity; Tier 3 (Phoenix) gives
self-custody at cost of channel-open fees.

**Bob (privacy-focused)**: wants no provider tracking, accepts complexity.
Best fit: **Tier 4** with Tor + own node. Or **Tier 2** Fedimint
federation he trusts.

**Carol (small merchant)**: needs always-on receive capability for
incoming payments. Best fit: **Tier 4** with watchtower setup, or
**Tier 3** with reliable LSP. Tier 1 sacrificed for direct control of
funds.

## Common pitfalls

- **Tier 1 silent risk**: provider can be hacked, regulated, or
  rugged. Strike's $50k+ outage events have happened.
- **Tier 2 federation chosen poorly**: small federations may collude.
- **Tier 3 LSP fees**: small payments can be eaten by channel-open
  costs; users misunderstand the underlying mechanics.
- **Tier 4 backup loss**: SCB without timely recovery can mean lost
  channels; on-chain wallet recovery is well understood, Lightning
  is not.
- **Mobile UX vs sovereignty**: phones are inherently part-time
  online; Tier 4 requires careful channel management.

## References

- Bitcoin Magazine wallet comparisons.
- "Lightning wallet self-custody guide" — bitcoinerjobs.org.
- Phoenix wallet architecture (ACINQ docs).
- Fedimint wallet integrations.
