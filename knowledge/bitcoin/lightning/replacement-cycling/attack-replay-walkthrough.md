# Replacement-Cycling Attack Replay - Walkthrough Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/lightning/replacement-cycling`.
> Canonical source: https://lists.linuxfoundation.org/pipermail/lightning-dev/2023-October/004176.html
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/lightning/replacement-cycling/SKILL.md

## Concept

The **replacement-cycling attack** (Antoine Riard, October 2023)
exploits a subtle interaction between Bitcoin mempool replacement
policy and Lightning HTLC enforcement. By repeatedly replacing
opposing transactions in the mempool, an attacker can prevent a
victim's HTLC from confirming, eventually causing the victim to lose
funds (typically the HTLC value, ~hundreds of sat to thousands per
attack instance).

This article walks through a concrete replay of the attack scenario.

## Walkthrough / mechanics

### Setup

- Channel: Alice — Bob — Carol (3-node chain).
- Alice routes a 100,000 sat HTLC to Carol via Bob.
- Bob is the victim. Alice and Carol are colluding attackers.

### State machine recap

When the Bob<->Carol channel goes on chain (force close), Bob has a
**HTLC-success transaction** he can broadcast using the preimage to
claim the HTLC value. Carol can claim via **HTLC-timeout** if the
preimage is not revealed before the CLTV timeout.

### Attack sequence

1. Force-close: Carol broadcasts the commitment tx for the Bob-Carol
   channel.
2. Bob enters `LOCAL_FORCE_CLOSE` state. He extracts the HTLC outputs
   and prepares his HTLC-success tx using the preimage Alice gave him
   when forwarding.
3. Bob broadcasts HTLC-success.
4. Carol broadcasts a **conflicting** HTLC-timeout tx with higher fee.
   This is allowed by Bitcoin Core mempool replacement rules (Rule 3:
   replacement must pay higher fee).
5. Bob's HTLC-success is **evicted** from mempool.
6. Bob re-broadcasts HTLC-success with even higher fee.
7. Carol replaces with another HTLC-timeout-conflicting tx.
8. Steps 5-7 repeat. Each iteration costs Carol some fee, but each
   replacement evicts Bob's package.
9. Carol's strategy: between cycles, Carol drops her own tx so Bob's
   tx gets back into mempool. But Carol monitors mempool; just before
   block confirmation, she does another replace, re-evicting Bob's
   tx.
10. Eventually CLTV timeout passes; Bob can no longer claim the HTLC.
    Carol's HTLC-timeout confirms; Carol gets back the HTLC value.

### Key mechanism

The **replacement cycling** is a sequence of mempool-state tweaks that
forces Bob's HTLC-success to either:

- Be evicted whenever a block is about to confirm.
- Or be confirmed while Carol pays a small replacement fee penalty.

The attacker only needs Bob's HTLC-success to NOT confirm before
CLTV timeout. They don't need to confirm anything themselves.

### Why this works

Bitcoin Core mempool RBF policy (BIP125 rules):
- Replacement must pay higher fee than original.
- Doesn't require the replacement to actually confirm.

So Carol can keep "replacing" with progressively higher fees, and at
peak congestion drop her replacement, rebroadcasting only when Bob
re-tries. The pattern is asymmetric: Carol can keep playing this
shell game cheaply.

### Fix attempts

- **Anchor outputs with CPFP** help but can be similarly evicted by
  cycling the parent commitment.
- **TRUC (BIP-431) / v3 tx**: limits replacement to specific
  topologies, prevents the cycling pattern.
- **Watchtowers**: aggressively rebroadcast on Bob's behalf, but if
  Bob's node misses one window, attack still works.

## Worked example

Concrete trace for HTLC of 100,000 sat:

```
Block height H:
  Carol force-closes Bob-Carol channel.
  Mempool: [commitment_tx]

H+1:
  Bob's commitment confirmed. Bob broadcasts HTLC-success @ 5 sat/vB.
  Mempool: [HTLC-success(Bob)]

H+1, 5 min later:
  Carol broadcasts HTLC-timeout @ 6 sat/vB conflicting with the same HTLC output.
  Mempool: [HTLC-timeout(Carol)] (Bob's evicted)

H+1, 10 min later:
  Bob notices, broadcasts HTLC-success @ 8 sat/vB.
  Mempool: [HTLC-success(Bob)]

H+1, 15 min later:
  Carol replaces with HTLC-timeout @ 10 sat/vB.
  Mempool: [HTLC-timeout(Carol)]

H+2:
  Block contains neither (Carol times the next replace just before the
  block).

H+3, just before block:
  Carol drops her tx (mempool empty).
  Bob's HTLC-success not in mempool either (he saw Carol's higher fee
  and gave up).

H+3:
  Block mined; nothing relevant included.

... repeats ...

H+CLTV (timeout):
  Carol broadcasts HTLC-timeout, no one can replace.
  Block confirms HTLC-timeout. Carol claims 100,000 sat.

Bob lost 100,000 sat (he had advanced this to Alice as forwarded payment).
```

## Common pitfalls

- **Watchtower latency**: even a 1-block missed window suffices for the
  attack to land if CLTV is very close.
- **Underestimating attacker fee budget**: each cycle costs the
  attacker only the *difference* between their tx fee and the previous.
  Cycling 100 times might cost ~10 % of HTLC value.
- **Anchor channels alone insufficient**: anchors help in normal
  scenarios but the cycling attack circumvents them.
- **Watching only the channel commitment**: must monitor HTLC-success
  and HTLC-timeout transactions separately.
- **Believing eviction means failure**: a Bitcoin node operator might
  see "Bob's tx removed from mempool" and fail to rebroadcast,
  permanently losing the funds.

## References

- Antoine Riard. "Replacement Cycling Attack" lightning-dev mailing list, October 2023.
- BIP-431 (Topologically Restricted Until Confirmation).
- BIP-125 (RBF policy).
