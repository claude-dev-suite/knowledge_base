# Graph RAG -- Community Summarization: Global, Local, and DRIFT Search

## TL;DR

Community summarization is the core innovation of Microsoft GraphRAG. After building a knowledge graph from documents, the Leiden algorithm partitions entities into hierarchical communities. Each community receives an LLM-generated summary that captures its key themes, entities, and relationships. At query time, three search strategies leverage these structures differently: **local search** starts from specific entities and traverses their graph neighborhood, **global search** maps queries across all community summaries then reduces partial answers into a final response, and **DRIFT search** iteratively expands from seed entities using community context to guide traversal. This article provides complete implementations of all three strategies with production optimizations.

---

## Community Detection and Hierarchical Summarization

### The Leiden Algorithm

GraphRAG uses the Leiden algorithm (an improvement over Louvain) to detect communities in the knowledge graph. Communities are groups of densely connected entities that likely represent coherent topics or themes.

```python
import networkx as nx
from dataclasses import dataclass, field


@dataclass
class CommunityNode:
    id: int
    level: int
    entity_names: list[str]
    relationship_descriptions: list[str]
    summary: str = ""
    title: str = ""
    rank: float = 0.0
    parent_id: int | None = None
    children_ids: list[int] = field(default_factory=list)


def build_community_hierarchy(
    entities: list[dict],
    relationships: list[dict],
    max_levels: int = 3,
    resolution: float = 1.0,
) -> dict[int, CommunityNode]:
    """Build hierarchical communities using the Leiden algorithm.

    The resolution parameter controls community granularity:
    - Higher resolution -> more, smaller communities
    - Lower resolution -> fewer, larger communities

    Multiple resolution levels create the hierarchy:
    Level 0: finest granularity (many small communities)
    Level 1: medium granularity
    Level 2: coarsest granularity (few large communities)
    """
    # Build NetworkX graph
    G = nx.Graph()
    for entity in entities:
        G.add_node(entity["name"], **entity)
    for rel in relationships:
        G.add_edge(
            rel["source"],
            rel["target"],
            description=rel["description"],
            weight=rel.get("weight", 1.0),
        )

    communities: dict[int, CommunityNode] = {}
    community_id = 0
    prev_level_mapping: dict[str, int] = {}  # entity -> community_id

    for level in range(max_levels):
        # Adjust resolution for each level
        level_resolution = resolution * (0.3 ** level)

        # Run Leiden (using graspologic or leidenalg)
        # Here we use networkx.community as a stand-in
        from networkx.algorithms.community import louvain_communities

        level_communities = louvain_communities(
            G, resolution=level_resolution, seed=42
        )

        for member_set in level_communities:
            member_names = list(member_set)

            # Collect relationship descriptions within this community
            rel_descs = []
            for rel in relationships:
                if rel["source"] in member_set and rel["target"] in member_set:
                    rel_descs.append(rel["description"])

            node = CommunityNode(
                id=community_id,
                level=level,
                entity_names=member_names,
                relationship_descriptions=rel_descs,
            )

            # Link to parent communities from previous level
            if level > 0:
                parent_ids = set()
                for name in member_names:
                    if name in prev_level_mapping:
                        parent_ids.add(prev_level_mapping[name])
                for pid in parent_ids:
                    if pid in communities:
                        communities[pid].children_ids.append(community_id)
                        node.parent_id = pid

            communities[community_id] = node
            for name in member_names:
                prev_level_mapping[name] = community_id
            community_id += 1

    return communities
```

### Generating Community Summaries

Each community summary captures the key themes, entities, relationships, and insights within that community. These summaries are the primary context for global search.

```python
from typing import Protocol


class LLMInterface(Protocol):
    def invoke(self, prompt: str) -> object: ...


COMMUNITY_SUMMARY_PROMPT = """You are an expert analyst. Summarize the following community
of entities and their relationships into a comprehensive report.

## Community Entities
{entities_section}

## Community Relationships
{relationships_section}

## Instructions
Write a structured summary that includes:
1. A descriptive title for this community (one line)
2. A summary paragraph explaining what this community represents
3. Key findings: the most important facts and relationships
4. Potential questions this community can help answer

Keep the summary under {max_tokens} tokens. Focus on information density.
Do not include generic statements -- every sentence should convey specific information."""


def generate_community_summaries(
    communities: dict[int, CommunityNode],
    entities: list[dict],
    llm: LLMInterface,
    max_tokens: int = 1500,
) -> dict[int, CommunityNode]:
    """Generate LLM summaries for each community.

    This is the most expensive part of the indexing pipeline.
    For a corpus with 500 communities, expect ~500 LLM calls.
    """
    entity_lookup = {e["name"]: e for e in entities}

    for community_id, community in communities.items():
        # Build entity descriptions
        entities_section = ""
        for name in community.entity_names[:30]:  # cap to avoid token overflow
            entity = entity_lookup.get(name, {})
            desc = entity.get("description", "No description")
            etype = entity.get("type", "Unknown")
            entities_section += f"- **{name}** ({etype}): {desc}\n"

        # Build relationship descriptions
        relationships_section = ""
        for desc in community.relationship_descriptions[:30]:
            relationships_section += f"- {desc}\n"

        prompt = COMMUNITY_SUMMARY_PROMPT.format(
            entities_section=entities_section,
            relationships_section=relationships_section,
            max_tokens=max_tokens,
        )

        response = llm.invoke(prompt)
        text = response.content

        # Extract title (first line) and summary (rest)
        lines = text.strip().split("\n")
        community.title = lines[0].strip("# ").strip()
        community.summary = text
        community.rank = len(community.entity_names) * len(
            community.relationship_descriptions
        )

    return communities
```

---

## Local Search

Local search is optimized for questions about specific entities and their direct relationships. It starts by identifying entities mentioned in the query, retrieves their graph neighborhood, and generates an answer from that local context.

```python
from dataclasses import dataclass


@dataclass
class LocalSearchResult:
    answer: str
    matched_entities: list[str]
    context_entities: list[dict]
    context_relationships: list[dict]
    community_context: list[str]
    token_budget_used: int


class LocalSearchEngine:
    """GraphRAG local search implementation.

    Local search follows this flow:
    1. Map query to entities (vector similarity on entity embeddings)
    2. Retrieve entity neighborhood (1-2 hops)
    3. Retrieve community reports for matched entities
    4. Combine into context window
    5. Generate answer

    Token budget allocation (default 8000 tokens):
    - Entity descriptions: 30%
    - Relationship descriptions: 25%
    - Community summaries: 25%
    - Source text chunks: 20%
    """

    def __init__(
        self,
        graph_store,
        embedding_model,
        llm: LLMInterface,
        token_budget: int = 8000,
    ):
        self.store = graph_store
        self.embedding_model = embedding_model
        self.llm = llm
        self.token_budget = token_budget

    def search(self, query: str, top_entities: int = 5) -> LocalSearchResult:
        """Execute local search for a query."""
        # Step 1: Find seed entities via vector similarity
        query_embedding = self.embedding_model.embed(query)
        seed_entities = self.store.vector_search_entities(
            query_embedding, top_k=top_entities
        )

        if not seed_entities:
            return LocalSearchResult(
                answer="No relevant entities found in the knowledge graph.",
                matched_entities=[],
                context_entities=[],
                context_relationships=[],
                community_context=[],
                token_budget_used=0,
            )

        # Step 2: Expand to neighborhood
        all_entities = {}
        all_relationships = []
        for entity in seed_entities:
            neighbors = self.store.get_entity_neighborhood(
                entity["name"], hops=2, limit=20
            )
            for n in neighbors:
                all_entities[n["name"]] = n

            rels = self.store.get_entity_relationships(entity["name"])
            all_relationships.extend(rels)

        # Step 3: Get community context for seed entities
        community_summaries = []
        for entity in seed_entities:
            community = self.store.get_entity_community(entity["name"])
            if community and community.get("summary"):
                community_summaries.append(community["summary"])

        # Step 4: Build context within token budget
        context = self._build_context(
            seed_entities=seed_entities,
            neighbor_entities=list(all_entities.values()),
            relationships=all_relationships,
            community_summaries=community_summaries,
        )

        # Step 5: Generate answer
        answer = self.llm.invoke(
            f"Answer the question based only on the provided knowledge graph context.\n"
            f"If the context does not contain enough information, say so.\n\n"
            f"## Knowledge Graph Context\n{context}\n\n"
            f"## Question\n{query}"
        )

        return LocalSearchResult(
            answer=answer.content,
            matched_entities=[e["name"] for e in seed_entities],
            context_entities=list(all_entities.values()),
            context_relationships=all_relationships,
            community_context=community_summaries,
            token_budget_used=self._estimate_tokens(context),
        )

    def _build_context(
        self,
        seed_entities: list[dict],
        neighbor_entities: list[dict],
        relationships: list[dict],
        community_summaries: list[str],
    ) -> str:
        """Build context string within token budget.

        Allocates budget proportionally:
        - 30% for entity descriptions (seed entities first)
        - 25% for relationship descriptions
        - 25% for community summaries
        - 20% for source text chunks
        """
        entity_budget = int(self.token_budget * 0.30)
        rel_budget = int(self.token_budget * 0.25)
        community_budget = int(self.token_budget * 0.25)

        # Entity section
        entity_lines = []
        tokens_used = 0
        for entity in seed_entities + neighbor_entities:
            line = f"- {entity['name']} ({entity.get('type', 'entity')}): {entity.get('description', 'N/A')}"
            line_tokens = self._estimate_tokens(line)
            if tokens_used + line_tokens > entity_budget:
                break
            entity_lines.append(line)
            tokens_used += line_tokens

        # Relationship section
        rel_lines = []
        tokens_used = 0
        seen = set()
        for rel in relationships:
            key = (rel.get("source"), rel.get("target"), rel.get("type"))
            if key in seen:
                continue
            seen.add(key)
            line = f"- {rel.get('source')} --[{rel.get('type')}]--> {rel.get('target')}: {rel.get('description', '')}"
            line_tokens = self._estimate_tokens(line)
            if tokens_used + line_tokens > rel_budget:
                break
            rel_lines.append(line)
            tokens_used += line_tokens

        # Community section
        community_lines = []
        tokens_used = 0
        for summary in community_summaries:
            summary_tokens = self._estimate_tokens(summary)
            if tokens_used + summary_tokens > community_budget:
                # Truncate summary
                available = community_budget - tokens_used
                if available > 100:
                    truncated = summary[: available * 4]  # rough token-to-char
                    community_lines.append(truncated + "...")
                break
            community_lines.append(summary)
            tokens_used += summary_tokens

        # Assemble
        parts = []
        if entity_lines:
            parts.append("### Entities\n" + "\n".join(entity_lines))
        if rel_lines:
            parts.append("### Relationships\n" + "\n".join(rel_lines))
        if community_lines:
            parts.append(
                "### Community Context\n" + "\n---\n".join(community_lines)
            )

        return "\n\n".join(parts)

    @staticmethod
    def _estimate_tokens(text: str) -> int:
        """Rough token estimate: ~4 chars per token for English."""
        return len(text) // 4
```

---

## Global Search

Global search answers questions that require understanding the entire corpus, such as "What are the main themes?" or "Summarize all technical decisions." It uses a map-reduce pattern over community summaries.

```python
import asyncio
from dataclasses import dataclass


@dataclass
class GlobalSearchResult:
    answer: str
    communities_total: int
    communities_relevant: int
    map_results: list[dict]
    reduce_context_tokens: int


class GlobalSearchEngine:
    """GraphRAG global search using map-reduce over community summaries.

    Global search flow:
    1. Load all community summaries (pre-computed during indexing)
    2. MAP: For each community, ask the LLM to generate a partial
       answer relevant to the query (or mark as not relevant)
    3. REDUCE: Combine all partial answers into a final comprehensive answer

    Cost model:
    - MAP phase: N_communities * cost_per_llm_call
    - REDUCE phase: 1 LLM call (with all partial answers as context)
    - For 200 communities: ~201 LLM calls per query
    """

    MAP_PROMPT = """You are an analyst examining a community summary from a knowledge graph.

## Community Summary
{community_summary}

## Question
{query}

Based ONLY on the community summary above, provide any information relevant to the question.
If the community summary contains no relevant information, respond with exactly: "NO_RELEVANT_INFO"

Focus on specific facts, relationships, and findings. Do not speculate beyond what the summary states."""

    REDUCE_PROMPT = """You are an expert synthesizer combining insights from multiple knowledge
graph communities to answer a question comprehensively.

## Partial Answers from Communities
{partial_answers}

## Question
{query}

Synthesize the partial answers above into a single, comprehensive response.
Rules:
- Include all relevant information from the partial answers
- Resolve any contradictions by noting them explicitly
- Organize the response thematically
- Cite which community insights support each point when possible
- If no partial answers contain relevant information, say so clearly"""

    def __init__(
        self,
        community_store,
        llm: LLMInterface,
        max_community_level: int = 2,
        map_concurrency: int = 10,
    ):
        self.community_store = community_store
        self.llm = llm
        self.max_community_level = max_community_level
        self.map_concurrency = map_concurrency

    def search(self, query: str) -> GlobalSearchResult:
        """Execute global search using map-reduce over communities."""
        # Step 1: Load community summaries
        communities = self.community_store.get_communities_at_level(
            self.max_community_level
        )

        # Step 2: MAP phase -- generate partial answers
        map_results = self._map_phase(query, communities)

        # Step 3: Filter irrelevant results
        relevant_results = [
            r for r in map_results if r["is_relevant"]
        ]

        if not relevant_results:
            return GlobalSearchResult(
                answer="No relevant information found across knowledge graph communities.",
                communities_total=len(communities),
                communities_relevant=0,
                map_results=[],
                reduce_context_tokens=0,
            )

        # Step 4: REDUCE phase -- synthesize final answer
        answer, context_tokens = self._reduce_phase(query, relevant_results)

        return GlobalSearchResult(
            answer=answer,
            communities_total=len(communities),
            communities_relevant=len(relevant_results),
            map_results=relevant_results,
            reduce_context_tokens=context_tokens,
        )

    def _map_phase(
        self, query: str, communities: list[dict]
    ) -> list[dict]:
        """Generate partial answers from each community."""
        results = []

        for community in communities:
            prompt = self.MAP_PROMPT.format(
                community_summary=community["summary"],
                query=query,
            )
            response = self.llm.invoke(prompt)
            content = response.content.strip()

            is_relevant = "NO_RELEVANT_INFO" not in content
            results.append({
                "community_id": community["id"],
                "community_title": community.get("title", f"Community {community['id']}"),
                "partial_answer": content if is_relevant else "",
                "is_relevant": is_relevant,
            })

        return results

    def _reduce_phase(
        self, query: str, relevant_results: list[dict]
    ) -> tuple[str, int]:
        """Synthesize partial answers into a final response."""
        # Sort by community rank (larger communities first)
        sorted_results = sorted(
            relevant_results,
            key=lambda x: x.get("rank", 0),
            reverse=True,
        )

        # Build partial answers section
        partial_answers = ""
        for i, result in enumerate(sorted_results, 1):
            partial_answers += (
                f"\n### Community: {result['community_title']}\n"
                f"{result['partial_answer']}\n"
            )

        prompt = self.REDUCE_PROMPT.format(
            partial_answers=partial_answers,
            query=query,
        )

        context_tokens = len(prompt) // 4
        response = self.llm.invoke(prompt)

        return response.content, context_tokens

    async def search_async(self, query: str) -> GlobalSearchResult:
        """Async version with concurrent MAP phase for better latency.

        With map_concurrency=10 and 200 communities, the MAP phase
        takes ~20 sequential batches instead of 200 sequential calls.
        """
        communities = self.community_store.get_communities_at_level(
            self.max_community_level
        )

        # Concurrent MAP
        semaphore = asyncio.Semaphore(self.map_concurrency)

        async def map_one(community: dict) -> dict:
            async with semaphore:
                prompt = self.MAP_PROMPT.format(
                    community_summary=community["summary"],
                    query=query,
                )
                response = await self.llm.ainvoke(prompt)
                content = response.content.strip()
                is_relevant = "NO_RELEVANT_INFO" not in content
                return {
                    "community_id": community["id"],
                    "community_title": community.get("title", ""),
                    "partial_answer": content if is_relevant else "",
                    "is_relevant": is_relevant,
                }

        map_results = await asyncio.gather(
            *(map_one(c) for c in communities)
        )

        relevant = [r for r in map_results if r["is_relevant"]]
        if not relevant:
            return GlobalSearchResult(
                answer="No relevant information found.",
                communities_total=len(communities),
                communities_relevant=0,
                map_results=[],
                reduce_context_tokens=0,
            )

        answer, tokens = self._reduce_phase(query, relevant)
        return GlobalSearchResult(
            answer=answer,
            communities_total=len(communities),
            communities_relevant=len(relevant),
            map_results=relevant,
            reduce_context_tokens=tokens,
        )
```

---

## DRIFT Search

DRIFT (Dynamic Reasoning and Inference with Flexible Traversal) combines the precision of local search with the coverage of global search. It starts from seed entities, uses community context to guide exploration, and iteratively expands until the question is answered.

```python
from dataclasses import dataclass, field


@dataclass
class DRIFTSearchResult:
    answer: str
    iterations: int
    entities_visited: list[str]
    communities_consulted: list[int]
    intermediate_answers: list[str]
    converged: bool


class DRIFTSearchEngine:
    """GraphRAG DRIFT search: iterative graph traversal with community guidance.

    DRIFT search flow:
    1. Identify seed entities from query (like local search)
    2. Retrieve community context for seed entities
    3. Generate an intermediate answer + follow-up questions
    4. Use follow-up questions to identify next entities to explore
    5. Repeat until the answer is sufficient or max iterations reached

    DRIFT is best for questions that span multiple communities or
    require following chains of relationships.
    """

    DRIFT_PROMPT = """You are exploring a knowledge graph to answer a question.

## Current Graph Context
{current_context}

## Community Background
{community_context}

## Previous Findings
{previous_findings}

## Original Question
{query}

Based on the context above:
1. Provide your best current answer to the question
2. Rate your confidence (1-10) that the answer is complete
3. If confidence < 8, list 1-3 specific follow-up questions that would help
   complete the answer. Frame these as entity-specific queries.

Format your response as:
ANSWER: <your current answer>
CONFIDENCE: <1-10>
FOLLOW_UP: <question 1>
FOLLOW_UP: <question 2>
FOLLOW_UP: <question 3>"""

    def __init__(
        self,
        graph_store,
        community_store,
        embedding_model,
        llm: LLMInterface,
        max_iterations: int = 4,
        confidence_threshold: int = 8,
    ):
        self.graph_store = graph_store
        self.community_store = community_store
        self.embedding_model = embedding_model
        self.llm = llm
        self.max_iterations = max_iterations
        self.confidence_threshold = confidence_threshold

    def search(self, query: str) -> DRIFTSearchResult:
        """Execute DRIFT search with iterative expansion."""
        # Initialize: find seed entities
        query_embedding = self.embedding_model.embed(query)
        seed_entities = self.graph_store.vector_search_entities(
            query_embedding, top_k=3
        )

        visited_entities: set[str] = set()
        consulted_communities: set[int] = set()
        intermediate_answers: list[str] = []
        previous_findings = "None yet."

        current_entities = seed_entities
        converged = False

        for iteration in range(self.max_iterations):
            # Get context for current entities
            current_context = self._get_entity_context(
                current_entities, visited_entities
            )

            # Get community context
            community_context = self._get_community_context(
                current_entities, consulted_communities
            )

            # Mark as visited
            for entity in current_entities:
                visited_entities.add(entity["name"])

            # Generate intermediate answer
            prompt = self.DRIFT_PROMPT.format(
                current_context=current_context,
                community_context=community_context,
                previous_findings=previous_findings,
                query=query,
            )
            response = self.llm.invoke(prompt)
            parsed = self._parse_drift_response(response.content)

            intermediate_answers.append(parsed["answer"])

            # Check convergence
            if parsed["confidence"] >= self.confidence_threshold:
                converged = True
                break

            # No follow-up questions means we cannot expand further
            if not parsed["follow_ups"]:
                break

            # Find next entities to explore based on follow-up questions
            previous_findings = "\n".join(
                f"- Iteration {i + 1}: {ans}"
                for i, ans in enumerate(intermediate_answers)
            )

            current_entities = self._find_follow_up_entities(
                parsed["follow_ups"], visited_entities
            )

            if not current_entities:
                break

        # Final answer is the last intermediate answer
        final_answer = intermediate_answers[-1] if intermediate_answers else (
            "Unable to find relevant information in the knowledge graph."
        )

        return DRIFTSearchResult(
            answer=final_answer,
            iterations=iteration + 1 if intermediate_answers else 0,
            entities_visited=list(visited_entities),
            communities_consulted=list(consulted_communities),
            intermediate_answers=intermediate_answers,
            converged=converged,
        )

    def _get_entity_context(
        self, entities: list[dict], visited: set[str]
    ) -> str:
        """Build context from entity neighborhoods, excluding already-visited nodes."""
        parts = []
        for entity in entities:
            neighbors = self.graph_store.get_entity_neighborhood(
                entity["name"], hops=1, limit=15
            )
            unvisited_neighbors = [
                n for n in neighbors if n["name"] not in visited
            ]

            part = f"**{entity['name']}** ({entity.get('type', 'entity')})\n"
            part += f"Description: {entity.get('description', 'N/A')}\n"
            part += "Connections:\n"
            for n in unvisited_neighbors[:10]:
                part += f"  -> {n['name']}: {n.get('description', 'N/A')}\n"
            parts.append(part)

        return "\n\n".join(parts) if parts else "No entity context available."

    def _get_community_context(
        self, entities: list[dict], consulted: set[int]
    ) -> str:
        """Get community summaries for the entities being explored."""
        summaries = []
        for entity in entities:
            community = self.community_store.get_entity_community(entity["name"])
            if community and community["id"] not in consulted:
                consulted.add(community["id"])
                summaries.append(
                    f"**{community.get('title', 'Community')}**: "
                    f"{community['summary'][:500]}"
                )
        return "\n\n".join(summaries) if summaries else "No additional community context."

    def _find_follow_up_entities(
        self, follow_up_questions: list[str], visited: set[str]
    ) -> list[dict]:
        """Map follow-up questions to unvisited entities in the graph."""
        candidates = []
        for question in follow_up_questions:
            q_embedding = self.embedding_model.embed(question)
            matches = self.graph_store.vector_search_entities(q_embedding, top_k=3)
            for match in matches:
                if match["name"] not in visited:
                    candidates.append(match)

        # Deduplicate and take top candidates
        seen = set()
        unique = []
        for c in candidates:
            if c["name"] not in seen:
                seen.add(c["name"])
                unique.append(c)

        return unique[:5]

    @staticmethod
    def _parse_drift_response(text: str) -> dict:
        """Parse the structured DRIFT response."""
        result = {"answer": "", "confidence": 0, "follow_ups": []}

        lines = text.strip().split("\n")
        answer_lines = []
        capture_answer = False

        for line in lines:
            stripped = line.strip()
            if stripped.startswith("ANSWER:"):
                capture_answer = True
                answer_lines.append(stripped[7:].strip())
            elif stripped.startswith("CONFIDENCE:"):
                capture_answer = False
                try:
                    result["confidence"] = int(stripped.split(":")[1].strip())
                except (ValueError, IndexError):
                    result["confidence"] = 5
            elif stripped.startswith("FOLLOW_UP:"):
                capture_answer = False
                follow_up = stripped[10:].strip()
                if follow_up:
                    result["follow_ups"].append(follow_up)
            elif capture_answer:
                answer_lines.append(stripped)

        result["answer"] = " ".join(answer_lines).strip()
        return result
```

---

## Optimizing Community Summaries

### Caching Strategy

Community summaries are expensive to generate but stable once the graph is built. Implement a content-addressed cache to avoid regeneration.

```python
import hashlib
import json
from pathlib import Path


class CommunitySummaryCache:
    """Content-addressed cache for community summaries.

    Cache key = hash of (entity_names + relationship_descriptions)
    so summaries are only regenerated when community membership changes.
    """

    def __init__(self, cache_dir: str = ".graphrag_cache/summaries"):
        self.cache_dir = Path(cache_dir)
        self.cache_dir.mkdir(parents=True, exist_ok=True)

    def _compute_key(self, community: CommunityNode) -> str:
        """Hash community contents for cache key."""
        content = json.dumps({
            "entities": sorted(community.entity_names),
            "relationships": sorted(community.relationship_descriptions),
        }, sort_keys=True)
        return hashlib.sha256(content.encode()).hexdigest()[:16]

    def get(self, community: CommunityNode) -> str | None:
        """Return cached summary or None."""
        key = self._compute_key(community)
        cache_path = self.cache_dir / f"{key}.json"
        if cache_path.exists():
            data = json.loads(cache_path.read_text())
            return data.get("summary")
        return None

    def set(self, community: CommunityNode, summary: str) -> None:
        """Cache a community summary."""
        key = self._compute_key(community)
        cache_path = self.cache_dir / f"{key}.json"
        cache_path.write_text(json.dumps({
            "community_id": community.id,
            "level": community.level,
            "entity_count": len(community.entity_names),
            "summary": summary,
        }))

    def generate_with_cache(
        self,
        community: CommunityNode,
        llm: LLMInterface,
        entity_lookup: dict[str, dict],
    ) -> str:
        """Generate summary with caching."""
        cached = self.get(community)
        if cached:
            return cached

        # Build prompt and generate (same as generate_community_summaries)
        entities_section = ""
        for name in community.entity_names[:30]:
            entity = entity_lookup.get(name, {})
            desc = entity.get("description", "No description")
            entities_section += f"- {name}: {desc}\n"

        relationships_section = "\n".join(
            f"- {d}" for d in community.relationship_descriptions[:30]
        )

        prompt = COMMUNITY_SUMMARY_PROMPT.format(
            entities_section=entities_section,
            relationships_section=relationships_section,
            max_tokens=1500,
        )

        response = llm.invoke(prompt)
        summary = response.content

        self.set(community, summary)
        return summary
```

### Cost Optimization for Global Search

```python
def optimize_global_search_cost(
    communities: list[dict],
    query: str,
    embedding_model,
    relevance_threshold: float = 0.3,
) -> list[dict]:
    """Pre-filter communities using embedding similarity before LLM MAP calls.

    Instead of sending every community to the LLM (expensive), first
    compute embedding similarity between the query and each community
    summary. Only send communities above the threshold to the MAP phase.

    This typically reduces MAP calls by 60-80% with minimal quality loss.
    """
    query_embedding = embedding_model.embed(query)

    scored = []
    for community in communities:
        summary_embedding = embedding_model.embed(community["summary"][:500])
        similarity = cosine_similarity(query_embedding, summary_embedding)
        scored.append({**community, "pre_filter_score": similarity})

    # Filter by threshold
    filtered = [
        c for c in scored if c["pre_filter_score"] >= relevance_threshold
    ]

    # Sort by relevance
    filtered.sort(key=lambda x: x["pre_filter_score"], reverse=True)

    return filtered


def cosine_similarity(a, b) -> float:
    """Compute cosine similarity between two vectors."""
    import numpy as np
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
```

---

## Choosing Between Search Modes

| Signal | Recommended Mode |
|--------|-----------------|
| Query contains entity names | Local Search |
| Query asks "what are all" or "summarize" | Global Search |
| Query asks about chains/influence/effects | DRIFT Search |
| Query mixes specific + broad elements | DRIFT Search |
| Latency budget < 3 seconds | Local Search |
| Latency budget < 10 seconds | Local or DRIFT |
| Latency budget > 10 seconds acceptable | Any mode |

### Automatic Query Routing

```python
def route_query(query: str, llm: LLMInterface) -> str:
    """Route a query to the best search mode.

    In production, replace the LLM classifier with a fine-tuned
    lightweight model (e.g., distilbert) for lower latency and cost.
    """
    response = llm.invoke(
        "Classify this question for knowledge graph search.\n\n"
        f"Question: {query}\n\n"
        "Respond with exactly one word:\n"
        "- LOCAL: question about specific entities or their direct relationships\n"
        "- GLOBAL: question about corpus-wide themes, trends, or summaries\n"
        "- DRIFT: question about chains of influence, indirect effects, or "
        "connections spanning multiple topics"
    )
    mode = response.content.strip().upper()
    if mode not in ("LOCAL", "GLOBAL", "DRIFT"):
        return "LOCAL"  # safe default
    return mode
```

---

## Common Pitfalls

1. **Running global search with too many communities.** If you have 1000+ communities, global search makes 1000+ LLM calls per query. Use the embedding pre-filter above, or restrict to a specific community level.

2. **Not tuning Leiden resolution.** Default resolution may produce communities that are too large (everything in one cluster) or too small (every entity is its own community). Experiment with resolution values between 0.5 and 2.0.

3. **DRIFT search infinite expansion.** Without a strict max_iterations cap and confidence threshold, DRIFT can keep exploring forever. Set max_iterations=4 and confidence_threshold=8 as defaults.

4. **Stale community summaries.** When the underlying graph changes (new entities/relationships), community summaries become stale. Use the content-addressed cache above to detect when regeneration is needed.

5. **Ignoring community hierarchy.** Global search at level 0 (finest) has the most communities and highest cost. Start at the coarsest level (level 2) for broad questions. Only drill into finer levels if the coarse summaries are insufficient.

---

## References

- Edge et al. "From Local to Global: A Graph RAG Approach to Query-Focused Summarization" (2024)
- DRIFT: "Dynamic Reasoning and Inference with Flexible Traversal" (Microsoft Research, 2024)
- Traag et al. "From Louvain to Leiden: guaranteeing well-connected communities" (Scientific Reports, 2019)
- Microsoft GraphRAG documentation: https://microsoft.github.io/graphrag/
