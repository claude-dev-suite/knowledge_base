# Compact Block Filters (BIP157/158) - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/p2p`.
> Canonical source: BIP157, BIP158
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/p2p/SKILL.md

## Concept

Compact block filters let light clients privately scan the chain for
transactions affecting their addresses. The server pre-computes one
filter per block (BIP158: a Golomb-Rice-coded set over scriptPubKeys
and outpoints); the client downloads the chain of filter HEADERS plus
filters of interest and queries each filter locally. Server learns
nothing about which addresses the client cares about - a major upgrade
over BIP37 bloom filters.

The skill covers the two BIPs at a high level; this article walks the
GCS encoding, the false-positive rate analysis, and the
filter-header-chain authentication scheme.

## Walkthrough / mechanics

**BIP158: filter format.**

For each block, the filter contains:

```
For every output spent or created in the block:
  add scriptPubKey to the set (excluding OP_RETURN outputs and unspendable)

(Optionally: also previous-output-script for inputs)
```

**Encoding pipeline:**

```
1. Hash each item: f_i = SipHash-2-4(K, item) mod (N * M)
   K  = first 16 bytes of block hash
   N  = number of items
   M  = 2^P, where P = false-positive parameter (BIP158 specifies P = 19)
2. Sort and deduplicate: H = sorted({f_0, f_1, ..., f_{N-1}})
3. Encode differences (delta encoding): d_i = H_i - H_{i-1} - 1
4. Golomb-Rice encode each delta: quotient (unary) + remainder (P bits)
```

False-positive rate: `1/M = 2^-P`. With P=19, ~1 false positive per
524k items. Per block (avg 4k items), expected ~0.008 hits per query
on a non-matching client.

Average filter size: ~600 bytes per block (vs ~1.5 MB block).

**BIP157: P2P messages.**

| Message | Purpose |
|---------|---------|
| `getcfheaders` | Request filter headers in range |
| `cfheaders` | Reply with header chain |
| `getcfcheckpt` | Request checkpoint hashes (every 1000 blocks) |
| `cfcheckpt` | Reply |
| `getcfilters` | Request filters in range |
| `cfilter` | Reply with filter for one block |

A filter HEADER chains filter hashes:

```
filter_header_n = SHA256d(filter_hash_n || filter_header_{n-1})
filter_hash_n   = SHA256d(filter_n)
```

Clients verify continuity: each header is a hash of the prior header
+ current filter hash. To detect lying servers, query multiple peers
and require headers match.

## Worked example

**Client wallet sync flow:**

```python
def sync(client, target_height):
    # 1. Get filter checkpoints (every 1000 blocks) from N peers.
    checkpoints = {}
    for peer in client.peers:
        cps = peer.getcfcheckpt(0, target_height)
        for h, cp in cps.items():
            checkpoints.setdefault(h, []).append(cp)

    # 2. Verify majority agreement; flag disagreements.
    verified = {h: most_common(cps) for h, cps in checkpoints.items()}

    # 3. Download filter headers, in chunks of 2000.
    headers = []
    for start in range(0, target_height, 2000):
        peer = client.choose_peer()
        h = peer.getcfheaders(start, min(2000, target_height - start))
        # check h ends at the expected checkpoint
        headers.extend(h)

    # 4. Lazy: download filters only when needed.
    for height in range(0, target_height):
        if not need_to_check(height):
            continue
        f = peer.getcfilter(height)
        # verify f hashes to expected
        items_to_check = [hash160(addr) for addr in client.watch_addresses]
        if any_match(f, items_to_check):
            block = peer.getblock(height)
            client.scan(block)
```

**False-positive analysis with P=19:**

```
M = 2^19 = 524_288

For a block with 4_000 items, query for 1 address:
  P(hit | not in block) = 4000 / 524_288 ~= 0.0076

If client watches 100 addresses on a 700_000-block chain:
  expected_hits = 700_000 * 100 * 0.0076 = 5_320_000
                  = 5.3M unnecessary block downloads
                  
Wait -- that's per-block-per-address. Need to dedupe. With 100 addrs
checked against 1 filter:
  P(any hit) ~= 100 * 4000 / 524_288 = 0.76 per block
```

This sounds bad but in practice clients batch multiple-address queries
by combining them into the same filter probe; only ONE filter
fetch+match is done per block, and only one block download triggered
per (real or false) match.

**Filter encoding example (Python sketch):**

```python
import hashlib
def siphash(key, data):
    # placeholder: use a real SipHash-2-4 lib
    ...

def make_filter(items, block_hash, P=19):
    M = 1 << P
    K = block_hash[:16]
    N = len(items)
    F = N * M
    hashed = sorted(set(siphash(K, x) % F for x in items))
    deltas = [hashed[0]] + [hashed[i] - hashed[i-1] - 1 for i in range(1, len(hashed))]

    # Golomb-Rice encode each delta:
    out_bits = []
    for d in deltas:
        q, r = divmod(d, M)
        out_bits.append('1' * q + '0')              # quotient unary
        out_bits.append(format(r, f'0{P}b'))        # remainder P bits
    bitstring = ''.join(out_bits)
    # pad to byte and emit
    ...
```

## Alternative designs and measurements (2026)

BIP157/158 is the deployed answer, but it is not the settled one. Four
lines of work as of September 2026, none of them shipped:

| Proposal | Changes | Where discussed |
|----------|---------|-----------------|
| Binary fuse (Fuse16) filters | Replaces the GCS encoding above | Delving Bitcoin, Optech #403 (2026-05-01) |
| Block-range filters | Adds a hierarchical filter layer above per-block filters | Delving Bitcoin RFC, Optech #420 (2026-08-28) |
| BlindBit Oracle v2 | Drops filters entirely for silent payments | Bitcoin-Dev, Optech #422 (2026-09-11) |
| UTXO set over P2P | Unrelated to filters; serves assumeUTXO snapshots | Bitcoin-Dev draft BIP, Optech #405 (2026-05-15) |

**Binary fuse filters.** Csaba Purszki's research replaces the
Golomb-Rice coded set with a Fuse16 binary fuse filter: a probabilistic
set-membership structure with O(1) query time (a GCS is O(N) - it must
be decoded sequentially), zero false negatives and a false-positive
rate of 1/2^k for k bits. Measured over 10 wallet use cases (24 up to
480 scripts) on 50,000 mainnet blocks and two CPUs, it gave a 9x-80x
query speedup on desktop x86_64 and 6x-45x on ARM, at a bandwidth cost
of 0%-3%.

**Block-range filters.** Optout's RFC keeps BIP157 intact and adds a
filter per *range* of blocks. A client downloads only range filters;
when a script matches a range it then downloads that range's individual
block filters and proceeds exactly as described above. Both range and
block filters are fetched for matching ranges, so the saving comes from
never fetching per-block filters for non-matching ranges. Simulations
on ~30k blocks with two script sets (one at 4-6 transactions, one at
20-30) found the total range-filter size falls as the range grows but
that most of the saving is cancelled if the range grows too far; the
best trade-off measured was a 256-block range, reducing total download
size by roughly 70-80% for the tested sets.

**Filters vs. per-output streaming (silent payments).** Rob Segers
benchmarked BlindBit Oracle v2 - a BIP352 indexing server that drops
filters completely and instead streams per-output data (txid, tweak,
and an 8-byte output prefix) - against BIP158 filters and taproot-only
filters. On unsampled block data from taproot activation through block
965,089 (255,434 blocks), the streaming approach downloads about 2.1x
as many bytes as a taproot-only filter plus the raw tweak data a filter
client still needs, and that comparison excludes the full block a
filter client must fetch on every match. What the extra bytes buy is no
false positives and no per-match block fetches. Segers also noted an
unsolved integrity gap: a client cannot tell that a server *omitted* a
tweak for a block, which silently loses the receiver money. His server
publishes per-block commitments over the sorted tweak set and
checkpoints them to nostr every six hours, which makes omissions
attributable after the fact but does not prevent them - clients should
still fetch the full block on a match. For the per-transaction scanning
cost these bandwidth numbers trade against, see
`../../privacy/silent-payments/scan-cost-analysis.md`.

**UTXO set over P2P.** Not a filter scheme, but the other 2026 proposal
for getting bulk chain data from peers: Fabian Jahr's draft BIP defines
a new service bit, four new P2P messages and a UTXO-set merkle root
known to the requester, so a new node can obtain an assumeUTXO snapshot
from peers rather than an external download. See the p2p SKILL.md.

## Common bugs / pitfalls

1. **Not authenticating filter headers.** A single peer can lie about
   the filter content (omit or fabricate items). Always cross-check
   with multiple peers; mismatch = ban.
2. **Querying with raw addresses instead of scriptPubKeys.** The
   filter contains scriptPubKey bytes, not address strings. Hash
   `OP_0 <hash160>` for P2WPKH, `OP_1 <Q_x>` for P2TR.
3. **Forgetting OP_RETURN exclusion.** BIP158 explicitly excludes
   OP_RETURN from filters (would flood with metadata-only spam).
4. **Filter version mismatch.** BIP158 has a "filter type" byte; type
   0x00 is "Basic" (output and prev-output scripts). Other types may
   exist (BIP158 was open-ended). Specify type 0 explicitly.
5. **Block height vs hash.** `getcfilter` takes a block hash, not
   height. Convert via header chain.
6. **Memory blowup on initial sync.** Storing all filter headers in
   memory for 800k blocks = ~25 MB. Storing all filters = ~500 MB.
   Use disk storage.
7. **Confusing "compact filters" with "compact blocks".** BIP158
   filters are for SPV clients. BIP152 compact blocks are for
   full-node block propagation. Different problems entirely.

## References

- BIP157: https://github.com/bitcoin/bips/blob/master/bip-0157.mediawiki
- BIP158: https://github.com/bitcoin/bips/blob/master/bip-0158.mediawiki
- LDK Neutrino: https://github.com/lightninglabs/neutrino
- btcsuite/btcwallet filter scan: https://github.com/btcsuite/btcwallet
- BDK chain sync: https://docs.rs/bdk_chain/latest/
- Binary fuse filters vs GCS (Optech #403, 2026-05-01):
  https://bitcoinops.org/en/newsletters/2026/05/01/
- Block-range filters RFC (Optech #420, 2026-08-28):
  https://bitcoinops.org/en/newsletters/2026/08/28/
- Silent-payments light-client benchmarks (Optech #422, 2026-09-11):
  https://bitcoinops.org/en/newsletters/2026/09/11/
- UTXO set sharing over P2P draft BIP (Optech #405, 2026-05-15):
  https://bitcoinops.org/en/newsletters/2026/05/15/
