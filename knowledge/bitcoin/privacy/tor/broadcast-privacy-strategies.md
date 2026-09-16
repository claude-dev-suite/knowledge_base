# Tor Broadcast Privacy Strategies - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/tor`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/doc/p2p-bad-ports.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/tor/SKILL.md

## Concept

The first peer to forward a transaction is often a strong heuristic for the
**originating IP**. Wallets and nodes therefore care a great deal about
**broadcast privacy**: how a freshly signed transaction first enters the
peer-to-peer mesh. Tor and Dandelion (BIP156 stem-and-fluff) are the
two pillars; their combinations differ in trust and bandwidth. Since
Bitcoin Core 31.0 (April 2026) there is a third: `-privatebroadcast`,
the only one of the three that Core implements as a dedicated
transaction-broadcast feature.

## Walkthrough / mechanics

### Strategy 1: own node, Tor-only outbound

Set `proxy=127.0.0.1:9050`, `onlynet=onion`, `listenonion=1`. Wallet talks
to local node via RPC; node broadcasts via Tor to onion peers. The wallet
never reveals its IP. The first onion peer to see the tx still learns
**something**: that some Tor exit (or onion peer) sent it. With
`onlynet=onion` there is no "exit"; the path is end-to-end onion.

Anonymity set: all Tor users running bitcoind onion-only. Onion is now
roughly half the reachable network. The Bitnodes crawler's snapshot of
16 September 2026 lists 27,260 reachable node addresses: 12,787 `.onion`
(46.9 %), 8,377 IPv4 (30.7 %), 4,804 I2P (17.6 %), 1,286 IPv6 (4.7 %)
and 6 CJDNS. A 3 August 2026 snapshot put onion at 51.0 % of 27,569.
Read that as an upper bound on the onion-**only** set: the crawl counts
*listening addresses*, so a node reachable on both clearnet and onion
contributes a row to both columns, and non-listening nodes are invisible
to it.

### Strategy 2: third-party broadcaster over Tor

Wallet builds the tx locally, then HTTP-POSTs it through Tor to an
explorer's `/api/tx` endpoint. Explorer broadcasts to its own peers.

Pro: simple, works from a mobile wallet without running a node.
Con: trust the explorer not to log + correlate; if it links inbound IP
(here: a Tor exit) with the tx, the privacy gain is the Tor user set.

### Strategy 3: Dandelion++ (BIP156, status Closed)

Tx travels through K random "stem" hops as a 1-to-1 forward (looks like
private relay), then "fluffs" to standard inv/tx broadcast. Source IP is
hidden behind the stem path length.

Historical rather than deployable on Bitcoin: BIP156 ("Dandelion -
Privacy Enhancing Routing", Denby/Miller/Fanti/Bakshi/Venkatakrishnan/
Viswanath) was never merged into Bitcoin Core and is listed with status
**Closed** in the BIPs repo (as of September 2026). It was never active
on Bitcoin mainnet; variants ship in some other chains (Grin, Monero).
Core's shipped answer to the same problem is strategy 6 below.

### Strategy 4: P2P-only via several Tor circuits

Run own node, force broadcast via multiple distinct Tor circuits using
`-noconnect` + `-addnode=<peer>:<port>` on rotating SOCKS5 ports
(`SocksPort 9051`, `9052`, ...) so each peer connection isolates a stream.
This minimises a single peer learning the source.

### Strategy 5: blind submission to a public mempool service

Services like `mempool.space` and `blockstream.info` accept POSTs over Tor.
Combine with a `.onion` endpoint to avoid a clearnet exit hop.

### Strategy 6: `-privatebroadcast` on your own node (Core 31.0+)

Bitcoin Core 31.0 (April 2026) added the boolean `-privatebroadcast`.
Transactions submitted via `sendrawtransaction` are broadcast over
short-lived connections through the Tor or I2P networks, without being
put in the local mempool first. A separate connection is used per
transaction, so two otherwise unrelated txs from the same node are not
linkable by shared connection — though a separate connection is not by
itself a separate circuit: bitcoind warns at startup when
`-proxyrandomize` is disabled that private-broadcast circuits may be
correlated with other connections over Tor, so keep `-proxyrandomize=1`
(the default). `getprivatebroadcastinfo` introspects the queue and
`abortprivatebroadcast` drains it. Wallet-submitted txs are not
affected by the option, and the option is incompatible with `-connect`.

Requires 31.1 (8 July 2026) or later. A disclosure of 6 June 2026
(credited to Eugene Siegel) describes an IP leak in 31.0: when private
broadcast picks an IPv4/IPv6 peer advertising BIP324 v2 transport and
the v2 handshake fails, the v1 retry bypasses the proxy and connects
directly, revealing the originator's IP to the recipient. Onion and I2P
peers and the wallet broadcast path are out of scope, as is any node
that cannot make direct clearnet outbound connections at all. Upstream
workarounds on 31.0 are `-privatebroadcast=0`, `-v2transport=0`, or
`-proxy=127.0.0.1:9050`.

## Worked example

Mobile wallet on cellular network:

| Strategy | First-hop visibility | Trust required | Bandwidth |
|----------|----------------------|----------------|-----------|
| Direct broadcast to Electrum server | Server sees IP and tx | server fully | tiny |
| Same, over Tor SOCKS | Server sees Tor exit + tx | server | tiny |
| Explorer onion HTTP | Explorer sees Tor exit + tx | explorer | tiny |
| Wallet -> own onion-only node via VPN | none beyond your own VPN exit | self | full IBD ~600 GB |
| Same over Tor + onlynet=onion | none | self | full IBD over Tor (slow) |
| Own node 31.1+, `-privatebroadcast=1` | none; separate connection per tx | self | full IBD ~600 GB |

Best privacy-per-cost for mobile: strategy 2 (explorer onion) with random
delay and one-tx-per-circuit. If you already run a node, strategy 6
dominates every row above it.

## Common pitfalls

- **Inv flooding**: a tx is `inv`-relayed to all peers ~immediately.
  Increasing the privacy delay (Bitcoin Core's `txconfirmtarget` does NOT
  affect this) means manually rebroadcasting; wallets that bind to a single
  peer can reduce side-channel leaks.
- **Mempool submission via clearnet HTTPS** still leaks to the explorer.
  Always use the `.onion` form when available.
- **Address-format inference**: P2TR-only outputs reduce sender anonymity
  set inside the network even if Tor hides IP. Pick output types matching
  the network mode of the day.
- **First-spend correlation**: if you broadcast tx A from Tor and tx B
  spending A's change from clearnet, the timing + UTXO graph still link
  you. Apply the same broadcast hygiene to follow-up txs.

## References

- bitcoin/doc/tor.md.
- Bitnodes crawler snapshot API — network-type split, snapshots of
  16 September 2026 and 3 August 2026 (bitnodes.io now redirects to
  btcnodes.io).
- "Dandelion: Redesigning the Bitcoin Network for Anonymity" — Venkatakrishnan et al., 2017.
- BIP155 — addrv2.
- BIP156 — Dandelion; status Closed in the BIPs repo (as of September 2026).
- Bitcoin Core PR 21515 — i2p support (parallels Tor model).
- Bitcoin Core PR 29415 — `-privatebroadcast`, shipped in 31.0.
- bitcoincore.org disclosure, 6 June 2026 — "Private Broadcast May Reveal
  Sender IP Address in Bitcoin Core 31.0"; fixed in 31.1 (PRs 35032, 35410).
