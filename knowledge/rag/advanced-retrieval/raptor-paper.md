# RAPTOR: Recursive Abstractive Processing for Tree-Organized Retrieval

## Overview / TL;DR

RAPTOR (Sarthi et al., 2024) constructs a hierarchical tree from documents by recursively clustering chunks, summarizing each cluster, and embedding the summaries. The result is a multi-level index where leaf nodes are original text chunks and internal nodes are progressively more abstract summaries. At query time, retrieval can match at any level of the tree -- detailed queries match leaf nodes while high-level queries match summary nodes. RAPTOR improves performance on multi-level abstraction tasks by 15-25% over flat retrieval and is particularly effective for long documents requiring both detail-level and theme-level question answering.

---

## The Problem RAPTOR Solves

Standard flat vector indexes embed every chunk at the same level of granularity. This creates two failure modes:

**Failure Mode 1: Summarization queries on a flat index**
- Query: "What is the main finding of this research paper?"
- The answer is spread across the abstract, results, and conclusion sections.
- No single chunk contains the complete answer.
- Retrieval returns fragmented pieces that the LLM struggles to synthesize.

**Failure Mode 2: Detail queries on a summary index**
- Query: "What was the exact p-value for the treatment group?"
- A summary-level chunk says "results were statistically significant" but omits the specific number.
- The detail is only in the original text.

RAPTOR solves both by maintaining chunks at every level of abstraction in a single index.

---

## Tree Construction Algorithm

### Step 1: Start with Leaf Nodes

Chunk the documents normally (recursive character splitting, 512 tokens, etc.). These chunks become the leaf nodes of the tree.

### Step 2: Cluster Leaf Nodes

Group semantically similar chunks into clusters using their embeddings. RAPTOR uses Gaussian Mixture Models (GMM) for soft clustering (a chunk can belong to multiple clusters), but k-means or hierarchical agglomerative clustering also work.

### Step 3: Summarize Each Cluster

Use an LLM to generate a concise summary of each cluster. The summary captures the shared theme of the cluster at a higher level of abstraction.

### Step 4: Embed the Summaries

Embed the summary texts using the same embedding model as the leaf nodes.

### Step 5: Recurse

Treat the summary nodes as a new set of "leaf" nodes. Cluster them, summarize the clusters, and embed the summaries. Repeat until you reach a single root summary or until the number of nodes at a level falls below a threshold.

### Step 6: Index All Nodes

Add all nodes (leaves + all levels of summaries) to a single vector index. Each node has metadata indicating its level in the tree and its parent/child relationships.

```
Level 0 (leaves):       [C1] [C2] [C3] [C4] [C5] [C6] [C7] [C8]
                           \   /       \ | /       \   |   /
Level 1 (cluster sums):   [S1]        [S2]        [S3]
                              \          |          /
Level 2 (root summary):        [Root Summary]

All 12 nodes are in the vector index.
A detail query matches C1-C8.
A theme query matches S1-S3 or Root.
```

---

## Implementation

### Full RAPTOR Pipeline

```python
import numpy as np
import anthropic
from sentence_transformers import SentenceTransformer
from sklearn.mixture import GaussianMixture
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class RAPTORNode:
    text: str
    embedding: np.ndarray
    level: int
    children: list["RAPTORNode"] = field(default_factory=list)
    parent: Optional["RAPTORNode"] = None
    metadata: dict = field(default_factory=dict)
    node_id: str = ""


class RAPTORTree:
    """Build a RAPTOR tree from documents."""

    def __init__(
        self,
        embed_model_name: str = "BAAI/bge-base-en-v1.5",
        summary_model: str = "claude-haiku-4-20250514",
        max_cluster_size: int = 10,
        min_cluster_size: int = 2,
        max_levels: int = 4,
        reduction_factor: float = 0.5,  # Target: reduce nodes by 50% each level
    ):
        self.embed_model = SentenceTransformer(embed_model_name)
        self.client = anthropic.Anthropic()
        self.summary_model = summary_model
        self.max_cluster_size = max_cluster_size
        self.min_cluster_size = min_cluster_size
        self.max_levels = max_levels
        self.reduction_factor = reduction_factor
        self.all_nodes: list[RAPTORNode] = []

    def build(self, chunks: list[dict]) -> list[RAPTORNode]:
        """Build the full RAPTOR tree."""
        # Step 1: Create leaf nodes
        texts = [c["text"] for c in chunks]
        embeddings = self.embed_model.encode(texts, normalize_embeddings=True)

        leaf_nodes = []
        for i, chunk in enumerate(chunks):
            node = RAPTORNode(
                text=chunk["text"],
                embedding=embeddings[i],
                level=0,
                metadata=chunk.get("metadata", {}),
                node_id=f"L0-{i}",
            )
            leaf_nodes.append(node)

        self.all_nodes.extend(leaf_nodes)
        current_level_nodes = leaf_nodes

        # Steps 2-5: Recursively cluster and summarize
        for level in range(1, self.max_levels + 1):
            if len(current_level_nodes) < self.min_cluster_size:
                break

            # Determine number of clusters
            n_clusters = max(
                self.min_cluster_size,
                int(len(current_level_nodes) * self.reduction_factor),
            )
            n_clusters = min(n_clusters, len(current_level_nodes) // self.min_cluster_size)

            if n_clusters < 2:
                break

            # Cluster nodes
            clusters = self._cluster_nodes(current_level_nodes, n_clusters)

            # Summarize each cluster
            summary_nodes = []
            for cluster_idx, cluster in enumerate(clusters):
                if len(cluster) < self.min_cluster_size:
                    continue

                summary_text = self._summarize_cluster(cluster)
                summary_embedding = self.embed_model.encode(
                    summary_text, normalize_embeddings=True
                )

                node = RAPTORNode(
                    text=summary_text,
                    embedding=summary_embedding,
                    level=level,
                    children=cluster,
                    metadata={"cluster_size": len(cluster)},
                    node_id=f"L{level}-{cluster_idx}",
                )
                for child in cluster:
                    child.parent = node

                summary_nodes.append(node)

            self.all_nodes.extend(summary_nodes)
            current_level_nodes = summary_nodes
            print(f"Level {level}: {len(summary_nodes)} summary nodes from {len(clusters)} clusters")

        print(f"RAPTOR tree complete: {len(self.all_nodes)} total nodes")
        return self.all_nodes

    def _cluster_nodes(
        self,
        nodes: list[RAPTORNode],
        n_clusters: int,
    ) -> list[list[RAPTORNode]]:
        """Cluster nodes using Gaussian Mixture Models."""
        embeddings = np.array([n.embedding for n in nodes])

        # GMM for soft clustering
        gmm = GaussianMixture(
            n_components=n_clusters,
            covariance_type="full",
            random_state=42,
        )
        gmm.fit(embeddings)

        # Assign each node to its most likely cluster
        labels = gmm.predict(embeddings)

        clusters = [[] for _ in range(n_clusters)]
        for i, label in enumerate(labels):
            clusters[label].append(nodes[i])

        # Remove empty clusters
        return [c for c in clusters if len(c) >= self.min_cluster_size]

    def _summarize_cluster(self, cluster: list[RAPTORNode]) -> str:
        """Generate a summary of a cluster of nodes."""
        cluster_text = "\n\n---\n\n".join(node.text for node in cluster)

        # Truncate if very long
        if len(cluster_text) > 10000:
            cluster_text = cluster_text[:10000] + "\n\n[truncated]"

        response = self.client.messages.create(
            model=self.summary_model,
            max_tokens=300,
            temperature=0,
            messages=[{
                "role": "user",
                "content": (
                    "Summarize the following text passages into a single, coherent "
                    "summary that captures the key information and themes. "
                    "The summary should be detailed enough to answer questions "
                    "about the main topics covered.\n\n"
                    f"Passages:\n{cluster_text}"
                ),
            }],
        )
        return response.content[0].text.strip()

    def get_all_embeddings(self) -> tuple[np.ndarray, list[dict]]:
        """Return all node embeddings and metadata for indexing."""
        embeddings = np.array([n.embedding for n in self.all_nodes])
        metadata = [
            {
                "text": n.text,
                "level": n.level,
                "node_id": n.node_id,
                "has_children": len(n.children) > 0,
                **n.metadata,
            }
            for n in self.all_nodes
        ]
        return embeddings, metadata
```

### Retrieval from RAPTOR Tree

Two retrieval strategies:

**Strategy 1: Flat retrieval (simple)** -- search all nodes regardless of level.

```python
def raptor_flat_retrieve(
    query: str,
    tree: RAPTORTree,
    top_k: int = 5,
) -> list[dict]:
    """Search all nodes in the tree (leaves and summaries) equally."""
    query_emb = tree.embed_model.encode(query, normalize_embeddings=True)
    embeddings, metadata = tree.get_all_embeddings()

    similarities = np.dot(embeddings, query_emb)
    top_indices = np.argsort(similarities)[::-1][:top_k]

    results = []
    for idx in top_indices:
        results.append({
            **metadata[idx],
            "score": float(similarities[idx]),
        })
    return results
```

**Strategy 2: Tree traversal** -- start at the top, traverse down to find the best level.

```python
def raptor_tree_traverse(
    query: str,
    tree: RAPTORTree,
    top_k: int = 5,
    start_level: int = None,
) -> list[dict]:
    """Traverse the tree from top to bottom, selecting the most relevant
    branch at each level, then return nodes at the appropriate granularity."""
    query_emb = tree.embed_model.encode(query, normalize_embeddings=True)

    # Start at the highest level
    if start_level is None:
        start_level = max(n.level for n in tree.all_nodes)

    current_nodes = [n for n in tree.all_nodes if n.level == start_level]

    while current_nodes:
        # Score all nodes at this level
        scored = []
        for node in current_nodes:
            sim = float(np.dot(node.embedding, query_emb))
            scored.append((node, sim))

        scored.sort(key=lambda x: x[1], reverse=True)

        # Take the best nodes at this level
        best = scored[:top_k]

        # Check if descending to children would be better
        if best[0][0].children:
            all_children = []
            for node, score in best:
                all_children.extend(node.children)

            child_scores = [
                (child, float(np.dot(child.embedding, query_emb)))
                for child in all_children
            ]
            child_scores.sort(key=lambda x: x[1], reverse=True)

            # If children score better, descend
            if child_scores[0][1] > best[0][1]:
                current_nodes = [c for c, s in child_scores[:top_k * 2]]
                continue

        # Return nodes at this level
        return [
            {"text": node.text, "level": node.level, "score": score, "node_id": node.node_id}
            for node, score in best[:top_k]
        ]

    return []
```

---

## LlamaIndex RAPTOR Implementation

```python
from llama_index.packs.raptor import RaptorRetriever, RaptorPack
from llama_index.core import SimpleDirectoryReader
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding

# Load documents
documents = SimpleDirectoryReader("./data").load_data()

# Create RAPTOR pack (handles tree construction)
raptor_pack = RaptorPack(
    documents=documents,
    embed_model=OpenAIEmbedding(model_name="text-embedding-3-small"),
    llm=OpenAI(model="gpt-4o-mini"),
    tree_depth=3,          # Number of summary levels
    similarity_top_k=5,
    mode="tree_traversal",  # or "collapsed" for flat retrieval
)

# Query
response = raptor_pack.run("What is the main theme of the report?")
print(response)
```

---

## When RAPTOR Wins

RAPTOR is most effective when:

1. **Documents are long** (10+ pages). Short documents do not need multi-level summarization.
2. **Queries span multiple abstraction levels**. A mix of "What is the overall conclusion?" and "What was the exact methodology?" in the same system.
3. **Summarization quality matters**. The summary nodes capture cross-chunk themes that no individual chunk contains.
4. **The corpus is relatively static**. RAPTOR tree construction is expensive; frequent updates require rebuilding.

### Benchmarks (from the RAPTOR paper)

On the QuALITY benchmark (long-document QA):

| Method | Accuracy |
|--------|----------|
| Standard RAG (flat chunks) | 56.6% |
| RAPTOR (collapsed / flat retrieval) | 62.4% |
| RAPTOR (tree traversal) | **63.7%** |

On NarrativeQA (book-length QA):

| Method | ROUGE-L |
|--------|---------|
| Standard RAG | 28.2 |
| RAPTOR | **32.1** |

The improvement is 7-14% depending on the benchmark and document length. The gains are largest on questions that require understanding themes or cross-section information.

---

## Tree Construction Cost

RAPTOR requires LLM calls to summarize each cluster at each level.

| Corpus Size | Leaf Chunks | L1 Summaries | L2 Summaries | Total LLM Calls | Cost (Haiku) |
|------------|-------------|-------------|-------------|-----------------|-------------|
| 10 docs (50 pages) | 200 | 40 | 8 | 48 | ~$0.50 |
| 100 docs (500 pages) | 2,000 | 400 | 80 | 480 | ~$5 |
| 1,000 docs (5K pages) | 20,000 | 4,000 | 800 | 4,800 | ~$50 |
| 10,000 docs (50K pages) | 200,000 | 40,000 | 8,000 | 48,000 | ~$500 |

The cost is a one-time investment (per corpus version). For frequently-updated corpora, consider building RAPTOR trees only for stable document subsets.

---

## Common Pitfalls

1. **Building RAPTOR for small corpora.** If your corpus has fewer than 50 chunks, the tree has only 1-2 levels with minimal benefit. RAPTOR shines on 500+ chunk corpora.
2. **Using too many tree levels.** More than 3-4 levels rarely helps and increases construction cost. The top-level summary becomes too abstract to be useful.
3. **Low-quality summaries.** The tree is only as good as its summaries. Use a model that produces informative, factual summaries (Haiku is usually sufficient, but check quality).
4. **Not indexing all levels.** The power of RAPTOR comes from having all levels in the same index. If you only index leaf nodes, you lose the multi-level benefit.
5. **Ignoring update costs.** Adding a new document requires re-clustering and re-summarizing (at least partially). Plan for incremental updates or periodic full rebuilds.
6. **Using hard clustering.** GMM (soft clustering) allows chunks to contribute to multiple cluster summaries, which produces better summaries for overlapping topics. K-means assigns each chunk to exactly one cluster, potentially splitting related content.

---

## References

- Sarthi et al., "RAPTOR: Recursive Abstractive Processing for Tree-Organized Retrieval" (2024) -- https://arxiv.org/abs/2401.18059
- LlamaIndex RAPTOR Pack -- https://llamahub.ai/l/llama-packs/llama-index-packs-raptor
- GitHub implementation -- https://github.com/parthsarthi03/raptor
