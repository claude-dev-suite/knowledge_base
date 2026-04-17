# Auto-Merging / Hierarchical Retrieval

## Overview / TL;DR

Auto-merging retrieval is a dynamic hierarchical pattern where leaf chunks are retrieved first, and then if enough leaf chunks from the same parent are present in the results, they are automatically merged into the parent chunk. This addresses the same precision-context trade-off as parent-document retrieval, but with a key difference: the merge is conditional. A parent is only promoted when the evidence (multiple matching leaves) justifies returning the broader context. This prevents the over-inclusion problem where a parent chunk is returned based on a single tangentially relevant child. LlamaIndex provides a built-in `AutoMergingRetriever` implementation. The critical tuning parameter is the merge threshold (what fraction of a parent's leaves must be retrieved to trigger a merge).

---

## How Auto-Merging Differs from Parent-Document Retrieval

| Aspect | Parent-Document | Auto-Merging |
|--------|----------------|-------------|
| Merge trigger | Always (every child maps to parent) | Conditional (threshold of children) |
| Context size | Fixed (always parent size) | Variable (leaf or parent, depending) |
| Precision risk | Parent may include irrelevant content | More precise (merge only when warranted) |
| Token efficiency | Lower (always returns large chunks) | Higher (returns leaves when parent isn't needed) |
| Complexity | Lower | Slightly higher |

**Example**:

A parent chunk covers "Database Connection Pooling" and has 5 leaf chunks:
1. "Connection pool sizing"
2. "Pool timeout configuration"
3. "Health check intervals"
4. "Connection leak detection"
5. "Pool monitoring metrics"

**Query**: "How do I configure connection pool timeouts?"

- **Parent-document**: Returns the full parent (all 5 topics) regardless of which children match.
- **Auto-merging (threshold=0.4)**: If only leaf 2 matches, returns just leaf 2. If leaves 1, 2, and 3 match (3/5 = 60% > 40% threshold), merges into the parent.

---

## The Algorithm

```
1. Retrieve top-K leaf nodes by similarity
2. For each leaf, look up its parent node
3. Count how many of each parent's children are in the result set
4. For each parent where (matched_children / total_children) >= threshold:
     - Remove the individual child nodes from results
     - Add the parent node instead
5. Return the modified result set
```

```python
from dataclasses import dataclass, field
from collections import defaultdict


@dataclass
class HierarchicalNode:
    id: str
    text: str
    embedding: list[float]
    level: int  # 0 = leaf, 1 = parent, 2 = grandparent, etc.
    parent_id: str = None
    children_ids: list[str] = field(default_factory=list)
    metadata: dict = field(default_factory=dict)


class AutoMergingRetriever:
    """Retrieve leaf nodes, auto-merge into parents when threshold is met."""

    def __init__(
        self,
        nodes: dict[str, HierarchicalNode],
        embed_model,
        merge_threshold: float = 0.4,
        top_k_leaves: int = 12,
        final_top_k: int = 5,
    ):
        self.nodes = nodes  # id -> HierarchicalNode
        self.embed_model = embed_model
        self.merge_threshold = merge_threshold
        self.top_k_leaves = top_k_leaves
        self.final_top_k = final_top_k

        # Index only leaf nodes
        self.leaf_nodes = {
            nid: node for nid, node in nodes.items() if node.level == 0
        }

    def retrieve(self, query: str) -> list[dict]:
        """Retrieve with auto-merging."""
        import numpy as np

        # Step 1: Retrieve leaf nodes
        query_emb = self.embed_model.encode(query, normalize_embeddings=True)
        leaf_scores = {}
        for nid, node in self.leaf_nodes.items():
            score = float(np.dot(query_emb, node.embedding))
            leaf_scores[nid] = score

        top_leaf_ids = sorted(
            leaf_scores, key=leaf_scores.get, reverse=True
        )[:self.top_k_leaves]

        # Step 2: Check merge conditions
        retrieved_set = set(top_leaf_ids)
        merged_results = self._try_merge(retrieved_set, leaf_scores)

        # Step 3: Sort by score and return
        merged_results.sort(key=lambda x: x["score"], reverse=True)
        return merged_results[:self.final_top_k]

    def _try_merge(
        self,
        retrieved_leaf_ids: set[str],
        leaf_scores: dict[str, float],
    ) -> list[dict]:
        """Attempt to merge leaf nodes into parent nodes."""
        # Count matched children per parent
        parent_child_counts = defaultdict(list)
        for leaf_id in retrieved_leaf_ids:
            leaf = self.nodes[leaf_id]
            if leaf.parent_id:
                parent_child_counts[leaf.parent_id].append(leaf_id)

        merged_results = []
        consumed_leaves = set()  # Leaves that were merged into a parent

        for parent_id, matched_children in parent_child_counts.items():
            parent = self.nodes[parent_id]
            total_children = len(parent.children_ids)
            match_ratio = len(matched_children) / total_children

            if match_ratio >= self.merge_threshold:
                # Merge: use parent instead of individual children
                best_child_score = max(
                    leaf_scores[cid] for cid in matched_children
                )
                merged_results.append({
                    "id": parent_id,
                    "text": parent.text,
                    "score": best_child_score,
                    "level": parent.level,
                    "merge_ratio": match_ratio,
                    "merged_children": len(matched_children),
                    "total_children": total_children,
                    "metadata": parent.metadata,
                })
                consumed_leaves.update(matched_children)

                # Recursively try merging parents into grandparents
                if parent.parent_id:
                    grandparent = self.nodes.get(parent.parent_id)
                    if grandparent:
                        # Check if enough of grandparent's children are merged
                        sibling_parents_merged = sum(
                            1 for sibling_id in grandparent.children_ids
                            if sibling_id in [r["id"] for r in merged_results]
                        )
                        gp_ratio = sibling_parents_merged / len(grandparent.children_ids)
                        if gp_ratio >= self.merge_threshold:
                            # Promote to grandparent level
                            merged_results = [
                                r for r in merged_results
                                if r["id"] not in grandparent.children_ids
                            ]
                            merged_results.append({
                                "id": grandparent.id,
                                "text": grandparent.text,
                                "score": best_child_score,
                                "level": grandparent.level,
                                "merge_ratio": gp_ratio,
                                "metadata": grandparent.metadata,
                            })

        # Add un-merged leaves
        for leaf_id in retrieved_leaf_ids:
            if leaf_id not in consumed_leaves:
                leaf = self.nodes[leaf_id]
                merged_results.append({
                    "id": leaf_id,
                    "text": leaf.text,
                    "score": leaf_scores[leaf_id],
                    "level": 0,
                    "merge_ratio": 0,
                    "metadata": leaf.metadata,
                })

        return merged_results
```

---

## LlamaIndex AutoMergingRetriever

LlamaIndex provides a built-in implementation that handles hierarchical node creation and auto-merging.

### Building the Hierarchical Index

```python
from llama_index.core import VectorStoreIndex, StorageContext, Document
from llama_index.core.node_parser import HierarchicalNodeParser, get_leaf_nodes
from llama_index.core.storage.docstore import SimpleDocumentStore
from llama_index.core.retrievers import AutoMergingRetriever

# Load documents
documents = [Document(text=text, metadata={"source": "guide.md"})]

# Create hierarchical nodes (3 levels)
node_parser = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[2048, 512, 128],  # Level 2 (grandparent), Level 1 (parent), Level 0 (leaf)
)
nodes = node_parser.get_nodes_from_documents(documents)

# Store ALL nodes in the docstore (needed for parent lookup)
docstore = SimpleDocumentStore()
docstore.add_documents(nodes)

# Index ONLY leaf nodes in the vector store
leaf_nodes = get_leaf_nodes(nodes)
storage_context = StorageContext.from_defaults(docstore=docstore)
index = VectorStoreIndex(
    leaf_nodes,
    storage_context=storage_context,
)

# Create the auto-merging retriever
retriever = AutoMergingRetriever(
    vector_retriever=index.as_retriever(similarity_top_k=12),
    storage_context=storage_context,
    simple_ratio_thresh=0.4,  # Merge when 40%+ of children are retrieved
)
```

### Querying

```python
# The retriever automatically handles merging
nodes = retriever.retrieve("How does the authentication system work?")

for node in nodes:
    print(f"Level: {node.node.metadata.get('level', 'leaf')}")
    print(f"Score: {node.score:.3f}")
    print(f"Text length: {len(node.node.text)} chars")
    print(f"Text preview: {node.node.text[:150]}...")
    print("---")
```

### With Query Engine

```python
from llama_index.core.query_engine import RetrieverQueryEngine
from llama_index.core.response_synthesizers import get_response_synthesizer

# Combine with a response synthesizer
query_engine = RetrieverQueryEngine.from_args(
    retriever=retriever,
    response_synthesizer=get_response_synthesizer(
        response_mode="compact",
    ),
)

response = query_engine.query("Explain the deployment architecture")
print(response)
```

---

## Threshold Tuning

The `simple_ratio_thresh` parameter controls when merging occurs:

### Impact of Threshold Values

| Threshold | Behavior | Context Size | Precision | Recall |
|-----------|----------|-------------|-----------|--------|
| 0.2 | Aggressive merging (1 out of 5 children triggers) | Large (parent-heavy) | Lower | Higher |
| 0.3 | Moderate-aggressive | Medium-large | Medium | Medium-high |
| **0.4** | **Balanced (recommended default)** | **Medium** | **Medium-high** | **Medium-high** |
| 0.5 | Moderate-conservative | Medium-small | Higher | Medium |
| 0.6 | Conservative (majority of children must match) | Small (leaf-heavy) | Higher | Lower |
| 0.8 | Very conservative (almost all children must match) | Small | Highest | Lowest |
| 1.0 | Never merges (effectively parent-document without parents) | Smallest | Highest | Lowest |

### How to Choose

```python
def evaluate_threshold(
    retriever_factory,
    eval_set: list[dict],
    thresholds: list[float] = [0.2, 0.3, 0.4, 0.5, 0.6, 0.8],
) -> list[dict]:
    """Evaluate different merge thresholds on your eval set."""
    results = []
    for thresh in thresholds:
        retriever = retriever_factory(merge_threshold=thresh)

        # Run eval
        metrics = evaluate_retrieval(retriever, eval_set)
        avg_context_tokens = measure_avg_context_size(retriever, eval_set)
        merge_rate = measure_merge_rate(retriever, eval_set)

        results.append({
            "threshold": thresh,
            "context_precision": metrics["context_precision"],
            "context_recall": metrics["context_recall"],
            "faithfulness": metrics["faithfulness"],
            "avg_context_tokens": avg_context_tokens,
            "merge_rate": merge_rate,  # % of queries that triggered a merge
        })

    return results
```

**Typical results** (technical documentation):

| Threshold | CP | CR | Faithfulness | Avg Context Tokens | Merge Rate |
|-----------|------|------|-------------|-------------------|-----------|
| 0.2 | 0.71 | 0.78 | 0.87 | 4,200 | 85% |
| 0.3 | 0.74 | 0.76 | 0.88 | 3,500 | 70% |
| 0.4 | 0.77 | 0.74 | 0.89 | 2,800 | 55% |
| 0.5 | 0.79 | 0.72 | 0.87 | 2,200 | 40% |
| 0.6 | 0.80 | 0.69 | 0.85 | 1,800 | 25% |
| 0.8 | 0.81 | 0.66 | 0.83 | 1,400 | 10% |

The sweet spot depends on whether you prioritize precision (higher threshold) or recall/faithfulness (lower threshold). For most applications, 0.3-0.5 is optimal.

---

## Tree Depth Considerations

Auto-merging works with any number of levels, but the practical sweet spot is 2-3 levels.

### Two Levels (Parent + Leaf)

```python
node_parser = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[1024, 256],  # Parent and leaf
)
```

**Best for**: Most use cases. Simple to reason about. Merging decision is straightforward: "did enough small chunks match?"

### Three Levels (Grandparent + Parent + Leaf)

```python
node_parser = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[2048, 512, 128],  # Grandparent, parent, leaf
)
```

**Best for**: Long documents with clear hierarchical structure (book chapters > sections > paragraphs). Allows cascading merges: leaves -> parent -> grandparent.

**Cascading merge behavior**:
1. Retrieve leaf chunks (128 tokens).
2. If 40%+ of a parent's leaves are retrieved, merge into the parent (512 tokens).
3. If 40%+ of a grandparent's children (parents) are merged, merge into the grandparent (2048 tokens).

### Four+ Levels

Rarely useful. Each additional level adds complexity to the merge logic and storage. The additional granularity levels provide diminishing returns for most document types.

---

## Comparison with Alternatives

### Auto-Merging vs Parent-Document

```
Query: "How do I configure pool timeouts?"

Parent-Document:
  - Always returns the full "Connection Pooling" section (2000 tokens)
  - Includes timeout config, sizing, health checks, leak detection
  - Works, but wastes tokens on irrelevant subsections

Auto-Merging (threshold=0.4):
  - Matches only "Pool timeout configuration" leaf (128 tokens)
  - Only 1 out of 5 children matched -> no merge
  - Returns just the timeout leaf -> precise and token-efficient

Query: "Explain how connection pooling works end to end"

Auto-Merging (threshold=0.4):
  - Matches leaves for sizing, timeouts, health checks (3/5 = 60%)
  - Triggers merge -> returns full parent section
  - Same result as parent-document, but only when warranted
```

### Auto-Merging vs RAPTOR

| Feature | Auto-Merging | RAPTOR |
|---------|-------------|--------|
| Tree construction | Chunk size hierarchy (mechanical) | Cluster + summarize (semantic) |
| Internal nodes | Original text (parent chunk) | Summaries (new text) |
| Query matching | Only at leaf level initially | At any level directly |
| Update cost | Re-chunk and re-embed | Re-cluster and re-summarize |
| Best for | Structured documents with clear hierarchy | Long documents needing multi-level abstraction |

---

## Production Considerations

### Storage Efficiency

Auto-merging requires storing all levels of the hierarchy:

```
Documents: 10,000 pages (~5M tokens)

Two levels (parent 1024 + leaf 256):
  Leaf nodes: ~20,000 (indexed in vector store)
  Parent nodes: ~5,000 (in docstore)
  Vector storage: 20,000 * 768 dims * 4 bytes = ~60 MB
  Docstore: ~5,000 * 1024 tokens * 4 chars = ~20 MB
  Total: ~80 MB

Three levels (2048 + 512 + 128):
  Leaf nodes: ~40,000
  Parent nodes: ~10,000
  Grandparent nodes: ~2,500
  Vector storage: 40,000 * 768 * 4 = ~120 MB
  Docstore: ~12,500 nodes * avg 800 tokens * 4 = ~40 MB
  Total: ~160 MB
```

Storage is rarely a bottleneck. The vector store is the main consumer, and adding more nodes at the leaf level is the primary cost (all levels are in the docstore, but only leaves are in the vector store).

### Incremental Updates

When a document changes, you need to:

1. Remove all nodes (leaf + parent + grandparent) from the old version.
2. Re-chunk and re-create the hierarchy for the new version.
3. Re-embed leaf nodes and add to the vector store.
4. Update the docstore with new parent/grandparent nodes.

```python
def update_document(doc_id: str, new_text: str):
    """Update a document's hierarchical nodes."""
    # Remove old nodes
    old_leaf_ids = get_leaf_ids_for_document(doc_id)
    old_parent_ids = get_parent_ids_for_document(doc_id)
    vector_store.delete(old_leaf_ids)
    for pid in old_parent_ids:
        docstore.delete(pid)

    # Create new hierarchy
    new_document = Document(text=new_text, metadata={"doc_id": doc_id})
    new_nodes = node_parser.get_nodes_from_documents([new_document])

    # Index new nodes
    docstore.add_documents(new_nodes)
    new_leaf_nodes = get_leaf_nodes(new_nodes)
    vector_store.add(new_leaf_nodes)
```

---

## Common Pitfalls

1. **Setting the threshold too low (0.1-0.2).** Nearly every query triggers a merge, making auto-merging equivalent to parent-document retrieval. The conditional nature is lost.
2. **Setting the threshold too high (0.8-1.0).** Merging almost never occurs, making auto-merging equivalent to flat retrieval. The context benefit is lost.
3. **Not retrieving enough leaf candidates.** If `top_k_leaves=5` and a parent has 8 children, at most 5/8 children can match (62.5%). This limits the maximum possible merge ratio. Set `top_k_leaves` to at least 2x your `final_top_k`.
4. **Ignoring the cascading merge impact.** With 3 levels, a cascading merge (leaf -> parent -> grandparent) can return a very large chunk (2048+ tokens). Ensure your token budget can handle it.
5. **Not storing all hierarchy levels in the docstore.** Auto-merging needs to look up parent nodes. If only leaves are stored, the merge cannot find the parent text.
6. **Using uneven chunk sizes.** If parent and child sizes are too close (e.g., 512 and 400), the merge provides minimal additional context. Maintain at least a 3:1 ratio between levels.

---

## References

- LlamaIndex AutoMergingRetriever -- https://docs.llamaindex.ai/en/stable/examples/retrievers/auto_merging_retriever/
- LlamaIndex HierarchicalNodeParser -- https://docs.llamaindex.ai/en/stable/api_reference/node_parsers/hierarchical/
- LlamaIndex advanced retrieval tutorial -- https://docs.llamaindex.ai/en/stable/optimizing/advanced_retrieval/advanced_retrieval/
