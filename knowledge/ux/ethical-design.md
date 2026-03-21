# Ethical Design

Ethical design respects user autonomy, transparency, and trust. Dark patterns — interface tricks that manipulate users against their own interests — have become a significant legal and reputational risk in 2024–2025.

---

## The Scale of the Problem

**2024 FTC / ICPEN / GPEN joint sweep** (27 authorities, 26 countries):
- **76% of reviewed apps and websites** employed at least one dark pattern
- **67% used multiple dark patterns**
- Subscription services were most affected
- Regulators are actively pursuing enforcement

---

## Legal Framework (2024–2025)

| Regulation | Scope | Effect on dark patterns |
|-----------|-------|------------------------|
| **FTC Act Section 5** (USA) | Unfair or deceptive practices | Prohibits patterns that mislead consumers |
| **EU EAA** (European Accessibility Act) | EU digital services | Enforcement began **June 2025**; includes anti-dark-pattern provisions |
| **CCPA** (California) | CA residents' data | Expressly prohibits dark patterns for consent |
| **CDPA** (Virginia) | VA residents' data | Prohibits dark patterns for consent |
| **TDPSA** (Texas) | TX residents' data | Prohibits dark patterns for consent |

**Amazon case (2024)**: FTC case cleared to proceed, alleging Amazon knowingly enrolled users in Prime subscriptions using dark patterns and made cancellation deliberately difficult.

**Liability**: Not just companies — **individual executives** can be held personally liable in some jurisdictions.

---

## Dark Pattern Catalog

### 1. Roach Motel
**Description**: Easy to get in, very difficult to get out. Subscribe in 2 clicks, cancel through 7 screens of "are you sure?" with retention offers.

**Ethical alternative**: Cancel in the same number of steps as subscribe. If users subscribe online, they must be able to cancel online — no phone call required.

```
✗ Dark:   Subscribe → 1 button
          Cancel   → Account > Settings > Subscription > Manage > Cancel > Confirm > Final warning > Done

✓ Ethical: Subscribe → 1 button
           Cancel   → Account > Subscription > Cancel (1 confirmation)
```

### 2. Hidden Costs (Drip Pricing)
**Description**: Show an attractive low price early, reveal fees, taxes, and charges only at checkout.

**Ethical alternative**: Show the total price from the first interaction. Legal requirement in many jurisdictions.

```
✗ Dark:   "From $9/month" → At checkout: $9 + $2 processing + $1.50 tax = $12.50

✓ Ethical: "$12.50/month (includes all fees and taxes)"
```

### 3. Confirm-Shaming
**Description**: Opt-out copy written to shame users into opting in.

```
✗ Dark:   "Yes, keep me informed!"
          "No, I hate saving money."

✓ Ethical: "Yes, sign me up for the newsletter"
           "No thanks" / "Remind me later" / "Skip"
```

### 4. Misdirection (Deceptive Visual Hierarchy)
**Description**: Styling a dangerous action as the primary/prominent button, and the safe action as secondary or hidden.

```
✗ Dark:   [Delete my account]  (filled primary button)
          Cancel               (small link, bottom right)

✓ Ethical: Are you sure you want to delete your account?
           [Cancel]                    (filled primary, left)
           [Delete account]            (outlined/ghost, destructive color, right)
```

**Rule**: Destructive actions = outlined or ghost button, `--color-destructive`. Constructive actions = filled primary button.

### 5. Forced Continuity (Surprise Auto-Renewal)
**Description**: Free trial converts to paid subscription without clear notice, or annual subscription auto-renews without reminder.

**Ethical alternative**: Email 7+ days before renewal with the exact amount, renewal date, and a prominent cancel link. Make it easy, not buried.

```
✗ Dark:   Silent auto-renewal with no reminder

✓ Ethical: Email 7 days before:
           "Your subscription renews on March 28 for $99.
            To cancel before then, click here: [Cancel subscription]"
```

### 6. Privacy Maze
**Description**: Opting out of data collection requires navigating 4+ menu levels: Settings → Account → Data & Privacy → Personalization → Ad Settings → Opt out.

**Ethical alternative**: The opt-out should be on the same page/screen where consent was given, or reachable in 1–2 steps from account settings.

### 7. Trick Questions
**Description**: Confusingly worded checkboxes that mean the opposite of what users expect.

```
✗ Dark:   □ Uncheck to not opt out of not receiving marketing emails
           (triple negative — users don't know what to check)

✓ Ethical: □ Send me marketing emails (opt-in, unchecked by default)
```

### 8. Bait and Switch
**Description**: Advertise one thing, deliver another. Common in ads ("Free!") that lead to paid products, or software that changes behavior after installation.

---

## Ethical Design Checklist

Before shipping any consent, subscription, or cancellation flow:

### Consent & Permissions
- [ ] Default state is "not opted in" for all optional data collection
- [ ] Consent is granular (separate checkboxes for different purposes)
- [ ] Withdrawing consent is as easy as giving it
- [ ] No pre-ticked checkboxes for non-essential processing

### Pricing & Subscriptions
- [ ] Total price (including taxes/fees) shown from first interaction
- [ ] Free trial → paid conversion communicated clearly at signup
- [ ] Cancellation available in same channel as signup (no phone-only cancellation)
- [ ] Auto-renewal reminder sent ≥7 days before charge

### Confirmation UX
- [ ] Destructive/irreversible actions use ghost/outlined buttons with destructive color
- [ ] Primary action (safe/constructive) is visually dominant
- [ ] Confirmation copy is neutral, not shame-based

### Opt-outs
- [ ] Opt-out language is neutral ("No thanks" not "No, I hate deals")
- [ ] Opt-out requires no more steps than opt-in
- [ ] Privacy settings reachable in ≤2 steps from account settings

---

## Building Trust Through Design

Users judge credibility within **milliseconds** based on design quality — **94% of first impressions are based on design alone**.

### Design signals that build trust

| Signal | Why it builds trust |
|--------|-------------------|
| **Visual consistency** | Signals organizational stability and intentionality |
| **Generous whitespace** | Signals confidence and premium quality |
| **Authentic photography** | Real people over stock photos signal credibility |
| **Clear error messages** | Admitting mistakes feels more human and trustworthy |
| **Loading / progress indicators** | Communicating what's happening builds confidence |
| **Readable typography** | Legible content signals respect for the user's time |
| **SSL / security indicators** | Explicit trust signals at sensitive moments |

### Trust is designed, not declared

"Trust is not declared; it is designed. In the era of algorithmic feeds and AI-generated content, trust is the single most valuable currency. It is conveyed almost instantly through design, not credentials alone."

---

## References

- FTC — [FTC, ICPEN, GPEN Announce Results of Review — Dark Patterns (2024)](https://www.ftc.gov/news-events/news/press-releases/2024/07/ftc-icpen-gpen-announce-results-review-use-dark-patterns-affecting-subscription-services-privacy)
- Reed Smith — [Dark Patterns Lead to Enforcement Spotlight: Key Compliance Steps for Businesses](https://www.reedsmith.com/articles/dark-patterns-lead-to-enforcement-spotlight-key-compliance-steps-for-businesses/)
- Deceptive Design — [Types of Dark Patterns](https://www.deceptive.design/types)
- NNGroup — [Trustworthy Design: Building Credibility on the Web](https://www.nngroup.com/articles/trustworthy-design/)
- Princeton Web Transparency Project — [Dark Patterns at Scale (2019)](https://webtransparency.cs.princeton.edu/dark-patterns/)
