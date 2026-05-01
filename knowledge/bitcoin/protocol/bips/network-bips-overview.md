# Network BIPs Overview - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/bips`.
> Canonical source: BIPs 31, 35, 37, 130, 133, 144, 152, 155, 156, 157, 158, 324, 330, 331, 339
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/bips/SKILL.md

## Concept

The P2P / network BIP family covers everything the wire protocol does
beyond raw block/tx serialization: handshake, peer discovery, gossip
optimization, encrypted transport, privacy-enhancing relay, and
client-side filter formats. The skill mentions individual BIPs in the
P2P category; this article groups them by what they solve (latency,
bandwidth, privacy, encryption) and explains adoption status.

## Walkthrough / mechanics

**Handshake & versioning:**

| BIP | Purpose | Status |
|-----|---------|--------|
| 31  | `pong` reply to `ping` | Active since 2011 |
| 130 | `sendheaders` for header-first sync | Active since Core 0.12 |
| 133 | `feefilter` mempool fee floor advertisement | Active since Core 0.13 |
| 339 | `wtxid` relay (`sendwtxidrelay`) | Active since Core 22.0 |

`sendheaders` reduces initial block download latency: peers send
`headers` proactively instead of waiting for `inv`/`getheaders`.
`wtxidrelay` was a foundational fix that lets nodes track txns by
witness-included id, preventing certain malleability-related relay
bugs.

**Compact transmission:**

| BIP | Purpose |
|-----|---------|
| 152 | Compact blocks - send 6-byte short txids instead of full txs |
| 330 | Erlay - reconciliation-based tx announcement |
| 331 | Package relay - submit related txs atomically |

BIP152 saved ~200ms per block hop in 2016, critical for mining latency.
BIP330 is proposed; reduces O(n*peers) tx-announcement bandwidth to
O(n*sqrt(peers)) via set reconciliation (sketches). BIP331 lets
mempool see parent+child as a unit (covered in package-relay/ articles).

**Filtering and SPV:**

| BIP | Purpose |
|-----|---------|
| 37  | Bloom filter SPV (deprecated - privacy issues, fingerprinting) |
| 157 | Compact block filter (CBF) protocol messages |
| 158 | Golomb-coded set filter format (BIP-158 GCS) |

CBF lets light clients download a per-block filter (~0.4% of block
size), check locally if any matching outputs exist, and then download
ONLY blocks they need. Better privacy than BIP37 because the server
cannot learn which addresses the client cares about.

**Address gossip:**

| BIP | Purpose |
|-----|---------|
| 155 | `addrv2` - new address message supporting Tor v3, I2P, CJDNS |
| 156 | Dandelion++ relay (proposed) |

`addrv2` extends the legacy `addr` to encode network type as a
discriminated union; legacy `addr` could only carry IPv4/IPv6.

**Encryption:**

| BIP | Purpose |
|-----|---------|
| 324 | v2 P2P transport - encrypted, authenticated, length-hidden |

BIP324 protects against passive eavesdropping (ISP-level surveillance
of which txs you're announcing) and resists fingerprinting because
all bytes look random. Doesn't authenticate peer identity; that's
deliberate (no key infrastructure needed).

## Worked example

**Modern node connection lifecycle:**

```
1. Client opens TCP to peer:8333.

2. (BIP324) Client sends 64-byte ephemeral key + ChaCha20Poly1305-derived
   garbage. Old peer sees first 4 bytes = magic mismatch, falls back
   to v1; new peer derives session key.

3. Both sides exchange encrypted VERSION:
     services    = NODE_NETWORK | NODE_WITNESS | NODE_NETWORK_LIMITED |
                   NODE_COMPACT_FILTERS | NODE_P2P_V2
     user_agent  = "/Satoshi:27.0/"

4. (BIP339) Either side sends `sendwtxidrelay` -> all subsequent
   tx announcements use wtxid.

5. (BIP130) Either side sends `sendheaders` -> peer sends `headers`
   pushes for new blocks instead of `inv`.

6. (BIP152) `sendcmpct` with version 2 (witness-aware).

7. (BIP155) `getaddr` -> peer responds with `addrv2` (mixed IP/Tor/I2P).

8. (BIP331) When submitting a low-fee parent + high-fee child:
     submitpackage [parent, child]
     mempool evaluates as a set.
```

**Bandwidth profile of a single block hop with all optimizations:**

| Stage | Bytes |
|-------|-------|
| `cmpctblock` (BIP152): 80-byte header + 6-byte short txids per tx | ~6 * num_txs |
| Missing-tx `getblocktxn` round-trip | ~0-1KB typically |
| `blocktxn` reply with full witness data of missing | varies |

Compare to pre-BIP152: full block (~1-2 MB) per hop.

## Common bugs / pitfalls

1. **Assuming BIP37 still works.** BIP37 is disabled by default in
   Bitcoin Core since 0.19. Light clients must use BIP157/158 or
   ElectrumX-style proxy.
2. **`addr` vs `addrv2` confusion.** A peer that hasn't sent
   `sendaddrv2` MUST receive only legacy `addr`. Sending `addrv2` to
   such a peer triggers disconnect.
3. **`feefilter` not respected by old peers.** `feefilter` is a hint;
   old peers ignore. Don't rely on it for spam reduction.
4. **BIP324 fallback storms.** A v2 node attempting v1 fallback after
   handshake failure can pollute peer banscore if too aggressive.
5. **Compact block "high-bandwidth mode" misuse.** Mode 1 = unsolicited
   `cmpctblock` after each block; mode 0 = wait for inv first. Setting
   mode 1 with too many peers floods bandwidth.
6. **Service flags taking effect mid-connection.** Service flags are
   negotiated only at handshake. Adding a service post-VERSION
   requires reconnect.
7. **Filter header desync.** BIP157 clients must verify filter
   headers form a chain (`prev_header` == hash of previous filter).
   Any single bad header from a malicious server stalls scan; clients
   should query multiple peers.

## References

- BIPs repo: https://github.com/bitcoin/bips
- Bitcoin Core net layer: https://github.com/bitcoin/bitcoin/tree/master/src/net.cpp
- BIP324 reference: https://github.com/bitcoin/bips/blob/master/bip-0324.mediawiki
- BIP158 GCS spec: https://github.com/bitcoin/bips/blob/master/bip-0158.mediawiki
- LDK Neutrino client: https://github.com/lightningnetwork/lnd/tree/master/lnrpc
