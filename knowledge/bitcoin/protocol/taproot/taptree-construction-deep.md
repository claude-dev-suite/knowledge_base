# Taptree Construction - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/protocol/taproot`.
> Canonical source: BIP341 (sections "Tagged Hashes" and "Constructing a Taproot output")
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/protocol/taproot/SKILL.md

## Concept

The taptree is a Merkle tree of leaves where each leaf is a TapLeafHash
of (`leaf_version`, `script`). Internal nodes use TapBranchHash with
lexicographic sorting of children. The taptree quick-ref covers basic
shape selection; this article goes deeper on weighted (Huffman) tree
construction, depth-cost trade-offs, and edge cases like single-leaf
or duplicate leaves.

## Walkthrough / mechanics

**Hash primitives:**

```
TapLeafHash(v, s)   = TaggedHash("TapLeaf",  v || compact_size(len(s)) || s)
TapBranchHash(a, b) = TaggedHash("TapBranch", min(a,b) || max(a,b))
```

The `min/max` ordering means a leaf's merkle path is independent of
left/right placement at construction; only the multiset of sibling
hashes matters. This simplifies path computation and makes equivalent
trees produce identical roots.

**Cost per leaf at depth d:**

```
control_block_size = 33 + 32 * d         # bytes
witness_weight     = 1 * (control_block_size + script_size + stack_inputs)
witness_vbytes     = witness_weight / 4
```

Each level of depth adds 32 bytes (8 vB) to the spend cost. So a
2^k-leaf balanced tree costs 8k vB extra per spend vs the cheapest
single-leaf case.

**Huffman construction:** given leaves `L_i` with probabilities `p_i`,
the cheapest expected witness uses a Huffman tree:

1. Heap of `(p_i, leaf_hash_i)`.
2. Pop two smallest, merge into branch, push back with summed
   probability.
3. Repeat until one node left.

The min/max ordering happens at hash time, not at heap time - heap is
keyed by probability.

## Worked example

Three leaves with probabilities `(0.7, 0.2, 0.1)`:

```python
from hashlib import sha256
import heapq

def tagged_hash(tag, msg):
    h = sha256(tag.encode()).digest()
    return sha256(h + h + msg).digest()

def leaf_hash(script, ver=0xc0):
    return tagged_hash("TapLeaf", bytes([ver]) + len(script).to_bytes(1, 'little') + script)

def branch_hash(a, b):
    lo, hi = sorted([a, b])
    return tagged_hash("TapBranch", lo + hi)

leaves = [
    (0.7, leaf_hash(b"\x21" + b"A"*32 + b"\xac")),   # likely path
    (0.2, leaf_hash(b"\x21" + b"B"*32 + b"\xac")),
    (0.1, leaf_hash(b"\x21" + b"C"*32 + b"\xac")),
]

# Huffman: merge two smallest first.
heap = [(p, i, h) for i, (p, h) in enumerate(leaves)]
heapq.heapify(heap)
nxt_id = len(leaves)
depth = {h: 0 for _, _, h in heap}
while len(heap) > 1:
    p1, _, h1 = heapq.heappop(heap)
    p2, _, h2 = heapq.heappop(heap)
    for h in (h1, h2):
        # propagate depth increment to all leaves under h
        depth[h] = depth.get(h, 0) + 1
    merged = branch_hash(h1, h2)
    heapq.heappush(heap, (p1 + p2, nxt_id, merged))
    nxt_id += 1

root = heap[0][2]
# leaves[0] -> depth 1 (33 + 32 = 65-byte control block)
# leaves[1] -> depth 2 (33 + 64 = 97-byte control block)
# leaves[2] -> depth 2
```

Expected witness overhead = `0.7*1 + 0.2*2 + 0.1*2 = 1.3` levels =
`~1.3*8 = 10.4` vB amortized vs the cheapest possible.

## Common bugs / pitfalls

1. **Sorting at heap level instead of hash level.** Huffman uses
   probabilities; TapBranchHash uses lexicographic sort of HASHES.
   Mixing these ruins both the Merkle commitment and the cost optimization.
2. **Single-leaf trees:** `merkle_root = leaf_hash` directly; no
   TapBranchHash needed. Control block is 33 bytes (no path).
3. **Depth overflow:** consensus permits up to 128 levels. Practical
   wallets cap at ~6-8 for sane fees. Going deeper signals a script
   compiler bug.
4. **Duplicate leaves:** allowed by BIP341 because of the sorted
   TapBranchHash. Two identical leaves at the same depth produce
   `branch_hash(h, h)` which is a valid intermediate.
5. **Forgetting compact_size length prefix in TapLeafHash.** The
   serialized script length is encoded as a CompactSize varint, not
   a fixed 1-byte length. For scripts >=253 bytes this matters.
6. **Tree shape choices that defeat privacy.** A heavily skewed tree
   with a 1-leaf branch reveals the depth at spend time, hinting at
   tree shape. Use balanced trees if path-frequency analysis is a
   concern; pure cost optimization may hurt fingerprinting resistance.

## References

- BIP341: https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
- BIP342: https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki
- rust-miniscript taptree builder: https://docs.rs/miniscript/latest/miniscript/descriptor/struct.TapTree.html
- rust-bitcoin `taproot::TaprootBuilder`: https://docs.rs/bitcoin/latest/bitcoin/taproot/struct.TaprootBuilder.html
