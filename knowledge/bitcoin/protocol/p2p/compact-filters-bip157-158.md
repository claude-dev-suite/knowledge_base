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
