# Personalization RAG -- Memory Systems

## TL;DR

Memory systems are the infrastructure layer that makes RAG personalization work. They store per-user preferences, interaction history, entity knowledge, and behavioral signals that the retrieval and generation layers consume. This guide covers four memory system options in depth: Mem0 (managed memory with automatic extraction), Zep (session-aware long-term memory with fact synthesis), Letta/MemGPT (self-editing memory with autonomous management), and custom vector-based memory (build your own with any vector store). For each system, it provides integration code, memory schema design, read/write patterns, and an honest assessment of tradeoffs. The right choice depends on whether you want managed infrastructure (Mem0/Zep), maximum control (custom), or autonomous memory management (Letta).

---

## Memory Architecture for Personalized RAG

### What to Store per User

| Memory Type | Examples | Update Frequency | Storage Strategy |
|---|---|---|---|
| **Profile data** | Role, team, expertise level | Rarely (user-set) | Key-value store |
| **Preferences** | Preferred languages, frameworks, verbosity | Updated from interactions | Key-value with history |
| **Interaction history** | Past queries, answers, feedback | Every interaction | Time-series / append-only |
| **Entity memory** | "Uses PostgreSQL 15", "Project X uses React" | Extracted from interactions | Knowledge graph / structured |
| **Session context** | Current conversation, working file | Per session | Ephemeral store |
| **Behavioral signals** | Click patterns, dwell time, thumbs up/down | Every interaction | Analytics store |

### Memory Read/Write Flow

```
Write path (after each interaction):
  User query + answer --> Memory Extractor --> Extract entities, preferences, facts
  --> Store in per-user memory with timestamp

Read path (before each query):
  New query --> Retrieve relevant memories for this user
  --> Inject into retrieval filter + generation prompt
  --> Generate personalized answer
```

---

## System 1: Mem0

Mem0 is a managed memory layer designed specifically for AI applications. It automatically extracts and organizes memories from conversations.

### Setup

```bash
pip install mem0ai
```

### Integration with RAG

```python
"""
Mem0 integration for personalized RAG.
"""
import os
from datetime import datetime

from mem0 import Memory


class Mem0PersonalizationLayer:
    """
    Use Mem0 as the memory backend for personalized RAG.

    Mem0 automatically:
    - Extracts facts and preferences from conversations
    - Deduplicates and updates memories
    - Provides relevance-based memory retrieval
    """

    def __init__(
        self,
        api_key: str | None = None,
        config: dict | None = None,
    ):
        if api_key:
            # Managed (cloud) version
            os.environ["MEM0_API_KEY"] = api_key
            self.memory = Memory()
        else:
            # Self-hosted version with custom config
            default_config = {
                "llm": {
                    "provider": "anthropic",
                    "config": {
                        "model": "claude-sonnet-4-20250514",
                        "temperature": 0,
                    },
                },
                "embedder": {
                    "provider": "openai",
                    "config": {
                        "model": "text-embedding-3-small",
                    },
                },
                "vector_store": {
                    "provider": "qdrant",
                    "config": {
                        "collection_name": "user_memories",
                        "host": "localhost",
                        "port": 6333,
                    },
                },
            }
            self.memory = Memory.from_config(config or default_config)

    def add_interaction(
        self,
        user_id: str,
        query: str,
        answer: str,
        metadata: dict | None = None,
    ) -> list[dict]:
        """
        Store an interaction and let Mem0 extract memories.

        Mem0 will automatically:
        - Extract facts ("user prefers Python")
        - Extract preferences ("user wants concise answers")
        - Extract entities ("user works on Project Alpha")
        - Deduplicate against existing memories
        - Update existing memories if new info contradicts old
        """
        messages = [
            {"role": "user", "content": query},
            {"role": "assistant", "content": answer},
        ]

        result = self.memory.add(
            messages,
            user_id=user_id,
            metadata=metadata or {"timestamp": datetime.now().isoformat()},
        )

        return result.get("results", [])

    def get_relevant_memories(
        self,
        user_id: str,
        query: str,
        limit: int = 10,
    ) -> list[dict]:
        """
        Retrieve memories relevant to the current query.

        Mem0 uses semantic search to find memories that are
        relevant to the query context.
        """
        results = self.memory.search(
            query=query,
            user_id=user_id,
            limit=limit,
        )

        return results.get("results", [])

    def get_all_memories(self, user_id: str) -> list[dict]:
        """Get all memories for a user (for profile display)."""
        results = self.memory.get_all(user_id=user_id)
        return results.get("results", [])

    def format_memories_for_prompt(
        self,
        memories: list[dict],
    ) -> str:
        """Format memories as context for the generation prompt."""
        if not memories:
            return ""

        lines = ["Known information about this user:"]
        for mem in memories:
            text = mem.get("memory", mem.get("text", ""))
            if text:
                lines.append(f"- {text}")

        return "\n".join(lines)

    def delete_user_memories(self, user_id: str) -> None:
        """Delete all memories for a user (GDPR compliance)."""
        self.memory.delete_all(user_id=user_id)

    def update_memory(self, memory_id: str, new_data: str) -> None:
        """Update a specific memory entry."""
        self.memory.update(memory_id, data=new_data)


# Integration with RAG pipeline
class Mem0RAGPipeline:
    """RAG pipeline with Mem0 personalization."""

    def __init__(self, retriever, generator, mem0_layer: Mem0PersonalizationLayer):
        self.retriever = retriever
        self.generator = generator
        self.mem0 = mem0_layer

    def invoke(self, query: str, user_id: str) -> dict:
        """Process a query with personalized memory."""
        # 1. Retrieve relevant memories
        memories = self.mem0.get_relevant_memories(user_id, query, limit=5)
        memory_context = self.mem0.format_memories_for_prompt(memories)

        # 2. Retrieve documents (optionally using memory for filtering)
        contexts = self.retriever.search(query, k=5)

        # 3. Generate with memory context
        full_context = f"{memory_context}\n\n---\n\n" + "\n\n".join(
            doc["text"] for doc in contexts
        )
        answer = self.generator.generate(query, full_context)

        # 4. Store this interaction for future personalization
        self.mem0.add_interaction(user_id, query, answer)

        return {"answer": answer, "contexts": contexts, "memories_used": len(memories)}
```

### Mem0 Tradeoffs

| Strength | Limitation |
|---|---|
| Automatic memory extraction | Relies on LLM for extraction (cost + latency) |
| Built-in deduplication | Cloud version has vendor lock-in |
| Simple API | Self-hosted requires Qdrant/Chroma setup |
| Handles memory updates/conflicts | Limited control over extraction logic |

---

## System 2: Zep

Zep provides session-aware long-term memory with automatic fact synthesis and temporal awareness.

### Setup

```bash
pip install zep-python
```

### Integration with RAG

```python
"""
Zep integration for personalized RAG.
"""
import asyncio
from datetime import datetime

from zep_python import ZepClient
from zep_python.memory import Memory, Message, Session
from zep_python.user import CreateUserRequest


class ZepPersonalizationLayer:
    """
    Use Zep as the memory backend for personalized RAG.

    Zep provides:
    - Session-based memory (conversations grouped by session)
    - Automatic fact extraction and synthesis
    - Temporal awareness (knows when things happened)
    - Entity relationship tracking
    - Built-in search across all user sessions
    """

    def __init__(self, api_url: str, api_key: str | None = None):
        self.client = ZepClient(base_url=api_url, api_key=api_key)

    async def ensure_user(
        self,
        user_id: str,
        metadata: dict | None = None,
    ) -> None:
        """Create or update user in Zep."""
        try:
            await self.client.user.get(user_id)
        except Exception:
            request = CreateUserRequest(
                user_id=user_id,
                metadata=metadata or {},
            )
            await self.client.user.add(request)

    async def create_session(
        self,
        session_id: str,
        user_id: str,
        metadata: dict | None = None,
    ) -> None:
        """Create a new conversation session."""
        session = Session(
            session_id=session_id,
            user_id=user_id,
            metadata=metadata or {"created_at": datetime.now().isoformat()},
        )
        await self.client.memory.add_session(session)

    async def add_interaction(
        self,
        session_id: str,
        query: str,
        answer: str,
    ) -> None:
        """
        Add a conversation exchange to Zep.

        Zep will automatically:
        - Extract facts from the conversation
        - Build entity relationships
        - Synthesize summaries of long conversations
        - Track temporal context
        """
        messages = [
            Message(
                role="user",
                content=query,
                role_type="user",
            ),
            Message(
                role="assistant",
                content=answer,
                role_type="assistant",
            ),
        ]

        memory = Memory(messages=messages)
        await self.client.memory.add_memory(session_id, memory)

    async def search_memories(
        self,
        user_id: str,
        query: str,
        limit: int = 5,
        search_type: str = "similarity",
    ) -> list[dict]:
        """
        Search user memories across all sessions.

        search_type options:
        - "similarity": semantic search across memories
        - "mmr": maximal marginal relevance (diversity-focused)
        """
        from zep_python.memory import MemorySearchPayload

        payload = MemorySearchPayload(
            text=query,
            search_type=search_type,
        )

        results = await self.client.memory.search_memory(
            session_id=None,  # Search across all sessions
            search_payload=payload,
            limit=limit,
        )

        return [
            {
                "text": r.message.get("content", ""),
                "role": r.message.get("role", ""),
                "score": r.score,
                "session_id": r.session_id,
                "created_at": r.message.get("created_at", ""),
            }
            for r in results
        ]

    async def get_user_facts(self, user_id: str) -> list[str]:
        """
        Get synthesized facts about a user.

        Zep automatically extracts and synthesizes facts from
        all conversations, e.g.:
        - "User prefers Python for backend development"
        - "User's project uses PostgreSQL and Redis"
        - "User has been working on authentication features"
        """
        try:
            user = await self.client.user.get(user_id)
            return user.facts or []
        except Exception:
            return []

    async def format_for_prompt(
        self,
        user_id: str,
        query: str,
    ) -> str:
        """Build personalization context for the prompt."""
        # Get synthesized facts
        facts = await self.get_user_facts(user_id)

        # Get relevant memories
        memories = await self.search_memories(user_id, query, limit=3)

        sections = []

        if facts:
            fact_lines = "\n".join(f"- {f}" for f in facts[:10])
            sections.append(f"User facts:\n{fact_lines}")

        if memories:
            mem_lines = []
            for m in memories:
                if m["role"] == "user":
                    mem_lines.append(f"- Previously asked: {m['text'][:100]}")
                else:
                    mem_lines.append(f"  Answer: {m['text'][:100]}")
            sections.append(f"Relevant history:\n" + "\n".join(mem_lines))

        return "\n\n".join(sections) if sections else ""

    async def delete_user(self, user_id: str) -> None:
        """Delete user and all their data (GDPR compliance)."""
        await self.client.user.delete(user_id)


# Synchronous wrapper for non-async codebases
class ZepSync:
    """Synchronous wrapper for Zep operations."""

    def __init__(self, zep_layer: ZepPersonalizationLayer):
        self.zep = zep_layer

    def add_interaction(self, session_id: str, query: str, answer: str):
        return asyncio.run(self.zep.add_interaction(session_id, query, answer))

    def get_context(self, user_id: str, query: str) -> str:
        return asyncio.run(self.zep.format_for_prompt(user_id, query))
```

### Zep Tradeoffs

| Strength | Limitation |
|---|---|
| Session-aware memory management | Requires running Zep server (Docker) |
| Automatic fact synthesis | Cloud version needed for full features |
| Temporal awareness | More complex setup than Mem0 |
| Entity relationship tracking | Async-first API (needs wrappers for sync) |

---

## System 3: Letta (MemGPT)

Letta implements self-editing memory: the AI agent autonomously decides what to remember, forget, and update.

### Setup

```bash
pip install letta
```

### Integration with RAG

```python
"""
Letta (MemGPT) integration for personalized RAG.
"""
from letta import create_client


class LettaPersonalizationLayer:
    """
    Use Letta for autonomous memory management.

    Letta (formerly MemGPT) provides:
    - Self-editing memory: the agent decides what to store
    - Tiered memory: core memory (always present) + archival (searchable)
    - Autonomous memory management (the agent edits its own memory)
    - Persistent state across conversations
    """

    def __init__(self):
        self.client = create_client()
        self._agents = {}  # Cache per-user agents

    def get_or_create_agent(
        self,
        user_id: str,
        system_prompt: str | None = None,
    ) -> str:
        """
        Get or create a Letta agent for this user.

        Each user gets their own agent with its own memory state.
        """
        if user_id in self._agents:
            return self._agents[user_id]

        default_prompt = (
            "You are a personalized technical assistant. "
            "Use your memory to track the user's preferences, "
            "tech stack, expertise level, and project context. "
            "When answering questions, adapt your response to "
            "what you know about this user."
        )

        agent_state = self.client.create_agent(
            name=f"rag-personal-{user_id}",
            system=system_prompt or default_prompt,
            memory_human=f"User ID: {user_id}",
            memory_persona=(
                "I am a personalized RAG assistant. I remember "
                "user preferences and past interactions to provide "
                "tailored answers."
            ),
        )

        self._agents[user_id] = agent_state.id
        return agent_state.id

    def query_with_memory(
        self,
        user_id: str,
        query: str,
        rag_context: str = "",
    ) -> dict:
        """
        Process a query through the Letta agent.

        The agent will:
        1. Check its memory for relevant user context
        2. Use the RAG context to formulate an answer
        3. Automatically update its memory with new information
        """
        agent_id = self.get_or_create_agent(user_id)

        # Provide RAG context as part of the message
        message = query
        if rag_context:
            message = (
                f"[Retrieved documentation]\n{rag_context}\n\n"
                f"[User question]\n{query}\n\n"
                "Answer the question using the documentation above. "
                "Remember any new preferences or context from this interaction."
            )

        response = self.client.send_message(
            agent_id=agent_id,
            message=message,
            role="user",
        )

        # Extract the assistant's response
        answer = ""
        memory_updates = []
        for msg in response.messages:
            if hasattr(msg, "assistant_message") and msg.assistant_message:
                answer = msg.assistant_message
            if hasattr(msg, "function_call") and msg.function_call:
                # Letta agents call functions to edit memory
                if "memory" in str(msg.function_call).lower():
                    memory_updates.append(str(msg.function_call))

        return {
            "answer": answer,
            "memory_updates": memory_updates,
            "agent_id": agent_id,
        }

    def get_user_memory(self, user_id: str) -> dict:
        """
        Get the current memory state for a user's agent.

        Returns both core memory (always-present context) and
        a summary of archival memory (searchable store).
        """
        agent_id = self.get_or_create_agent(user_id)
        memory = self.client.get_agent_memory(agent_id)

        return {
            "core_memory": {
                "human": memory.human if hasattr(memory, "human") else "",
                "persona": memory.persona if hasattr(memory, "persona") else "",
            },
            "archival_count": self.client.get_archival_memory_summary(agent_id),
        }

    def search_archival_memory(
        self,
        user_id: str,
        query: str,
        limit: int = 5,
    ) -> list[dict]:
        """Search the user's archival memory."""
        agent_id = self.get_or_create_agent(user_id)

        results = self.client.get_archival_memory(
            agent_id=agent_id,
            search=query,
            limit=limit,
        )

        return [
            {"text": r.text, "created_at": r.created_at}
            for r in results
        ]

    def delete_user(self, user_id: str) -> None:
        """Delete user agent and all memory (GDPR compliance)."""
        if user_id in self._agents:
            agent_id = self._agents.pop(user_id)
            self.client.delete_agent(agent_id)
```

### Letta Tradeoffs

| Strength | Limitation |
|---|---|
| Autonomous memory management | Most complex setup of the four options |
| Agent decides what to remember/forget | Higher LLM cost (agent uses tokens for memory ops) |
| Tiered memory (core + archival) | Memory edits can be unpredictable |
| Persistent conversational state | Harder to debug memory issues |

---

## System 4: Custom Vector Memory

Build your own memory system using any vector store. Maximum control, minimum vendor dependency.

```python
"""
Custom vector-based memory system for personalized RAG.
"""
import hashlib
import json
from dataclasses import asdict, dataclass, field
from datetime import datetime
from typing import Any

import numpy as np


@dataclass
class MemoryEntry:
    """A single memory entry for a user."""
    memory_id: str
    user_id: str
    memory_type: str  # "preference", "fact", "interaction", "entity"
    content: str
    metadata: dict = field(default_factory=dict)
    created_at: str = ""
    updated_at: str = ""
    access_count: int = 0
    importance: float = 0.5  # 0-1, used for eviction

    def __post_init__(self):
        if not self.memory_id:
            self.memory_id = hashlib.md5(
                f"{self.user_id}:{self.content}".encode()
            ).hexdigest()[:12]
        if not self.created_at:
            self.created_at = datetime.now().isoformat()
        self.updated_at = datetime.now().isoformat()


class CustomVectorMemory:
    """
    Custom memory system using a vector store.

    Features:
    - Per-user namespaced memory
    - Typed memory entries (preferences, facts, entities)
    - Importance-based eviction
    - Temporal decay
    - Full control over extraction and storage
    """

    def __init__(
        self,
        embedding_fn,
        max_memories_per_user: int = 200,
        decay_half_life_days: int = 30,
    ):
        """
        Args:
            embedding_fn: function that takes str -> np.ndarray
            max_memories_per_user: evict oldest/least important when exceeded
            decay_half_life_days: memories lose half their importance weight
                                  over this many days
        """
        self.embed = embedding_fn
        self.max_memories = max_memories_per_user
        self.decay_half_life = decay_half_life_days

        # In-memory store (replace with persistent vector store in production)
        self._memories: dict[str, list[MemoryEntry]] = {}
        self._embeddings: dict[str, list[np.ndarray]] = {}

    def add_memory(
        self,
        user_id: str,
        content: str,
        memory_type: str = "fact",
        importance: float = 0.5,
        metadata: dict | None = None,
    ) -> MemoryEntry:
        """Add a memory entry for a user."""
        # Check for duplicates
        existing = self._find_duplicate(user_id, content)
        if existing:
            # Update existing memory instead of adding duplicate
            existing.updated_at = datetime.now().isoformat()
            existing.access_count += 1
            existing.importance = max(existing.importance, importance)
            return existing

        entry = MemoryEntry(
            memory_id="",
            user_id=user_id,
            memory_type=memory_type,
            content=content,
            importance=importance,
            metadata=metadata or {},
        )

        if user_id not in self._memories:
            self._memories[user_id] = []
            self._embeddings[user_id] = []

        # Evict if at capacity
        if len(self._memories[user_id]) >= self.max_memories:
            self._evict_least_important(user_id)

        embedding = self.embed(content)
        self._memories[user_id].append(entry)
        self._embeddings[user_id].append(embedding)

        return entry

    def search(
        self,
        user_id: str,
        query: str,
        limit: int = 5,
        memory_type: str | None = None,
    ) -> list[tuple[MemoryEntry, float]]:
        """
        Search memories by semantic similarity.

        Returns list of (memory, score) tuples.
        """
        if user_id not in self._memories:
            return []

        query_embedding = self.embed(query)
        memories = self._memories[user_id]
        embeddings = self._embeddings[user_id]

        # Calculate similarity scores with temporal decay
        scored = []
        for mem, emb in zip(memories, embeddings):
            # Apply type filter
            if memory_type and mem.memory_type != memory_type:
                continue

            # Cosine similarity
            sim = float(np.dot(query_embedding, emb) / (
                np.linalg.norm(query_embedding) * np.linalg.norm(emb) + 1e-8
            ))

            # Temporal decay
            age_days = (
                datetime.now() - datetime.fromisoformat(mem.updated_at)
            ).days
            decay = 0.5 ** (age_days / self.decay_half_life)

            # Combined score: similarity * importance * decay
            score = sim * mem.importance * decay

            scored.append((mem, score))

        scored.sort(key=lambda x: x[1], reverse=True)

        # Update access counts for returned memories
        for mem, _ in scored[:limit]:
            mem.access_count += 1

        return scored[:limit]

    def extract_and_store(
        self,
        user_id: str,
        query: str,
        answer: str,
        extractor_fn=None,
    ) -> list[MemoryEntry]:
        """
        Extract memories from an interaction and store them.

        Uses an extractor function (typically LLM-based) to
        identify facts, preferences, and entities.
        """
        if extractor_fn is None:
            extractor_fn = self._default_extractor

        extracted = extractor_fn(query, answer)
        entries = []

        for item in extracted:
            entry = self.add_memory(
                user_id=user_id,
                content=item["content"],
                memory_type=item.get("type", "fact"),
                importance=item.get("importance", 0.5),
                metadata={"source_query": query[:100]},
            )
            entries.append(entry)

        return entries

    def _default_extractor(
        self,
        query: str,
        answer: str,
    ) -> list[dict]:
        """
        Simple rule-based memory extraction.

        For production, replace with LLM-based extraction.
        """
        extracted = []

        # Store the interaction itself
        extracted.append({
            "content": f"Asked: {query[:200]}",
            "type": "interaction",
            "importance": 0.3,
        })

        # Extract technology mentions
        tech_keywords = [
            "python", "javascript", "react", "postgresql", "redis",
            "docker", "kubernetes", "terraform", "aws", "gcp",
        ]
        mentioned = [
            t for t in tech_keywords
            if t in query.lower() or t in answer.lower()
        ]
        for tech in mentioned:
            extracted.append({
                "content": f"User works with {tech}",
                "type": "entity",
                "importance": 0.6,
            })

        # Detect preference signals
        preference_signals = {
            "i prefer": 0.8,
            "i like": 0.7,
            "i usually": 0.6,
            "i always": 0.7,
            "my team uses": 0.7,
        }
        for signal, importance in preference_signals.items():
            if signal in query.lower():
                idx = query.lower().index(signal)
                # Extract the rest of the sentence
                rest = query[idx:].split(".")[0].split("?")[0]
                extracted.append({
                    "content": rest.strip(),
                    "type": "preference",
                    "importance": importance,
                })

        return extracted

    def _find_duplicate(
        self,
        user_id: str,
        content: str,
    ) -> MemoryEntry | None:
        """Find a semantically similar existing memory."""
        if user_id not in self._memories:
            return None

        content_embedding = self.embed(content)

        for mem, emb in zip(self._memories[user_id], self._embeddings[user_id]):
            sim = float(np.dot(content_embedding, emb) / (
                np.linalg.norm(content_embedding) * np.linalg.norm(emb) + 1e-8
            ))
            if sim > 0.90:  # High similarity threshold
                return mem

        return None

    def _evict_least_important(self, user_id: str) -> None:
        """Remove the least important memory to make room."""
        if not self._memories[user_id]:
            return

        # Find the memory with lowest combined importance and recency
        min_score = float("inf")
        min_idx = 0

        for i, mem in enumerate(self._memories[user_id]):
            age_days = (
                datetime.now() - datetime.fromisoformat(mem.updated_at)
            ).days
            decay = 0.5 ** (age_days / self.decay_half_life)
            score = mem.importance * decay * (1 + 0.1 * mem.access_count)

            if score < min_score:
                min_score = score
                min_idx = i

        self._memories[user_id].pop(min_idx)
        self._embeddings[user_id].pop(min_idx)

    def format_for_prompt(
        self,
        user_id: str,
        query: str,
        limit: int = 5,
    ) -> str:
        """Format retrieved memories for injection into the prompt."""
        results = self.search(user_id, query, limit=limit)
        if not results:
            return ""

        lines = ["What we know about this user:"]
        by_type = {}
        for mem, score in results:
            if mem.memory_type not in by_type:
                by_type[mem.memory_type] = []
            by_type[mem.memory_type].append(mem.content)

        for mem_type, contents in by_type.items():
            lines.append(f"\n{mem_type.title()}:")
            for content in contents:
                lines.append(f"  - {content}")

        return "\n".join(lines)

    def get_user_summary(self, user_id: str) -> dict:
        """Get a summary of stored user memories."""
        if user_id not in self._memories:
            return {"total": 0}

        memories = self._memories[user_id]
        by_type = {}
        for mem in memories:
            by_type[mem.memory_type] = by_type.get(mem.memory_type, 0) + 1

        return {
            "total": len(memories),
            "by_type": by_type,
            "oldest": min(m.created_at for m in memories) if memories else None,
            "newest": max(m.updated_at for m in memories) if memories else None,
        }

    def delete_user(self, user_id: str) -> int:
        """Delete all memories for a user. Returns count deleted."""
        count = len(self._memories.get(user_id, []))
        self._memories.pop(user_id, None)
        self._embeddings.pop(user_id, None)
        return count
```

---

## Comparison Matrix

| Feature | Mem0 | Zep | Letta | Custom Vector |
|---|---|---|---|---|
| **Setup complexity** | Low | Medium | High | Medium |
| **Memory extraction** | Automatic (LLM) | Automatic (built-in) | Autonomous (agent) | Manual / custom |
| **Deduplication** | Built-in | Built-in | Agent-managed | Must implement |
| **Session awareness** | Basic | Strong | Strong | Must implement |
| **Temporal decay** | No | Yes | No (agent decides) | Must implement |
| **Cost per interaction** | LLM call for extraction | Included in server | LLM call (agent) | Only embedding |
| **Vendor lock-in** | Medium (cloud) / Low (OSS) | Medium | Low | None |
| **GDPR deletion** | API call | API call | Delete agent | Direct control |
| **Debugging** | Dashboard | Dashboard | Agent logs | Full control |
| **Recommended for** | Quick start, managed | Session-heavy apps | Autonomous agents | Maximum control |

### Decision Guide

- **Choose Mem0** when you want the fastest path to working personalization with minimal infrastructure
- **Choose Zep** when your application has clear session boundaries and you need temporal awareness
- **Choose Letta** when you want the agent to autonomously manage its own memory (advanced use case)
- **Choose Custom** when you need full control over memory logic, have privacy constraints that prevent using third-party services, or want to minimize per-interaction costs

---

## Common Pitfalls

1. **Storing too much**: not every interaction needs to be remembered. Storing everything creates noise that drowns out important signals. Focus on preferences, facts, and entities -- not every query.

2. **No memory eviction**: without eviction, memory grows unbounded. Old, irrelevant memories dilute retrieval quality. Implement importance-based eviction and temporal decay.

3. **LLM extraction errors**: Mem0 and Letta use LLMs to extract memories, which can produce incorrect extractions ("User prefers Java" when the user mentioned Java only in passing). Validate critical extractions.

4. **Missing deduplication**: without deduplication, the same fact gets stored multiple times ("User uses Python" x 50), wasting memory capacity and biasing retrieval toward frequently mentioned topics.

5. **Not testing memory quality**: treat memory retrieval like any other retrieval -- measure precision and recall. Bad memories in the prompt are worse than no memories.

6. **Ignoring cold start**: new users have no memories. Design a fallback path that provides reasonable answers without personalization, then gradually improve as memories accumulate.

---

## References

- Mem0: https://docs.mem0.ai/
- Zep: https://docs.getzep.com/
- Letta (MemGPT): https://docs.letta.com/
- Packer, C. et al. "MemGPT: Towards LLMs as Operating Systems." arXiv 2023.
- Park, J.S. et al. "Generative Agents: Interactive Simulacra of Human Behavior." UIST 2023.
