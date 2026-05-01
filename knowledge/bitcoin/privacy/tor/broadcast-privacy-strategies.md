# Tor Broadcast Privacy Strategies - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/privacy/tor`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/doc/p2p-bad-ports.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/privacy/tor/SKILL.md

## Concept

The first peer to forward a transaction is often a strong heuristic for the
**originating IP**. Wallets and nodes therefore care a great deal about
**broadcast privacy**: how a freshly signed transaction first enters the
peer-to-peer mesh. Tor and Dandelion (BIP152-style stem-and-fluff) are
the two pillars; their combinations differ in trust and bandwidth.

## Walkthrough / mechanics

### Strategy 1: own node, Tor-only outbound

Set `proxy=127.0.0.1:9050`, `onlynet=onion`, `listenonion=1`. Wallet talks
to local node via RPC; node broadcasts via Tor to onion peers. The wallet
never reveals its IP. The first onion peer to see the tx still learns
**something**: that some Tor exit (or onion peer) sent it. With
`onlynet=onion` there is no "exit"; the path is end-to-end onion.

Anonymity set: all Tor users running bitcoind onion-only. \~20–40 % of the
public network in 2025 (per Tor Project consensus measurements + Bitcoin
Core node-id surveys).

### Strategy 2: third-party broadcaster over Tor

Wallet builds the tx locally, then HTTP-POSTs it through Tor to an
explorer's `/api/tx` endpoint. Explorer broadcasts to its own peers.

Pro: simple, works from a mobile wallet without running a node.
Con: trust the explorer not to log + correlate; if it links inbound IP
(here: a Tor exit) with the tx, the privacy gain is the Tor user set.

### Strategy 3: Dandelion++ (BIP not assigned; bitcoinj has experimental)

Tx travels through K random "stem" hops as a 1-to-1 forward (looks like
private relay), then "fluffs" to standard inv/tx broadcast. Source IP is
hidden behind the stem path length. Not currently active in Bitcoin Core
mainnet but used in some forks (Grin, Monero).

### Strategy 4: P2P-only via several Tor circuits

Run own node, force broadcast via multiple distinct Tor circuits using
`-noconnect` + `-addnode=<peer>:<port>` on rotating SOCKS5 ports
(`SocksPort 9051`, `9052`, ...) so each peer connection isolates a stream.
This minimises a single peer learning the source.

### Strategy 5: blind submission to a public mempool service

Services like `mempool.space` and `blockstream.info` accept POSTs over Tor.
Combine with a `.onion` endpoint to avoid a clearnet exit hop.

## Worked example

Mobile wallet on cellular network:

| Strategy | First-hop visibility | Trust required | Bandwidth |
|----------|----------------------|----------------|-----------|
| Direct broadcast to Electrum server | Server sees IP and tx | server fully | tiny |
| Same, over Tor SOCKS | Server sees Tor exit + tx | server | tiny |
| Explorer onion HTTP | Explorer sees Tor exit + tx | explorer | tiny |
| Wallet -> own onion-only node via VPN | none beyond your own VPN exit | self | full IBD ~600 GB |
| Same over Tor + onlynet=onion | none | self | full IBD over Tor (slow) |

Best privacy-per-cost for mobile: strategy 3 (explorer onion) with random
delay and one-tx-per-circuit.

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
- "Dandelion: Redesigning the Bitcoin Network for Anonymity" — Venkatakrishnan et al., 2017.
- BIP155 — addrv2.
- Bitcoin Core PR 21515 — i2p support (parallels Tor model).
