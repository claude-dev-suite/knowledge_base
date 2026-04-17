# Memory Strategies for Conversational RAG

## TL;DR

Multi-turn RAG requires memory -- a mechanism to retain and selectively recall information from prior conversation turns. The challenge is not storage (conversations are small) but selection: which past information is relevant to the current turn? This article covers five memory strategies (sliding window, summary, vector, entity, and hybrid), token budget management, memory store implementations (Redis, Postgres, Zep, mem0), and a comparison of LangChain memory types. The right strategy depends on conversation length, topic diversity, and latency budget.

---

## Memory Strategy Landscape

| Strategy | Mechanism | Best For | Token Cost | Complexity |
|----------|-----------|----------|------------|------------|
| Sliding Window | Keep last N turns verbatim | Short conversations (<10 turns) | Low, fixed | Trivial |
| Summary | Summarize old turns, keep recent verbatim | Long conversations, single topic | Medium | Low |
| Vector Memory | Embed turns, retrieve relevant ones | Long conversations, diverse topics | Low per turn | Medium |
| Entity Memory | Track entities and their attributes | Entity-centric domains (CRM, support) | Medium | Medium |
| Hybrid | Recent window + summary of older + vector for retrieval | Production systems | Configurable | High |

---

## Strategy 1: Sliding Window

Keep the most recent N turns (N human + N assistant messages). Drop everything older.

### Implementation

```python
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage


class SlidingWindowMemory:
    """Keep the last N conversation turns."""

    def __init__(self, max_turns: int = 10):
        self.max_turns = max_turns
        self.messages: list[BaseMessage] = []

    def add_user_message(self, content: str):
        self.messages.append(HumanMessage(content=content))
        self._trim()

    def add_ai_message(self, content: str):
        self.messages.append(AIMessage(content=content))
        self._trim()

    def _trim(self):
        """Keep only the last max_turns * 2 messages (each turn = user + AI)."""
        max_messages = self.max_turns * 2
        if len(self.messages) > max_messages:
            self.messages = self.messages[-max_messages:]

    def get_messages(self) -> list[BaseMessage]:
        return self.messages

    def clear(self):
        self.messages = []
```

### When to Use

- Conversations are short (under 10 turns)
- Topics do not change (continuous discussion about one subject)
- Low latency requirement (no LLM call for memory management)
- Simple implementation is preferred

### Limitations

- Information from turn 1 is permanently lost after N+1 turns
- No way to recall early context even if it becomes relevant again
- Fixed token cost regardless of relevance

---

## Strategy 2: Summary Memory

Summarize older turns into a condensed representation. Keep recent turns verbatim.

### Implementation

```python
import anthropic
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage, SystemMessage


class SummaryMemory:
    """Summarize old turns, keep recent turns verbatim."""

    def __init__(
        self,
        recent_turns: int = 5,
        model: str = "claude-3-5-haiku-20241022",
    ):
        self.recent_turns = recent_turns
        self.messages: list[BaseMessage] = []
        self.summary: str = ""
        self.client = anthropic.Anthropic()
        self.model = model

    def add_user_message(self, content: str):
        self.messages.append(HumanMessage(content=content))
        self._maybe_summarize()

    def add_ai_message(self, content: str):
        self.messages.append(AIMessage(content=content))

    def _maybe_summarize(self):
        """Summarize old turns when history exceeds threshold."""
        max_messages = self.recent_turns * 2
        if len(self.messages) <= max_messages + 2:
            return

        # Messages to summarize (everything except the most recent)
        to_summarize = self.messages[:-(max_messages)]
        self.messages = self.messages[-(max_messages):]

        # Build text to summarize
        text_parts = []
        if self.summary:
            text_parts.append(f"Previous summary: {self.summary}")
        for msg in to_summarize:
            role = "User" if isinstance(msg, HumanMessage) else "Assistant"
            text_parts.append(f"{role}: {msg.content}")

        response = self.client.messages.create(
            model=self.model,
            max_tokens=300,
            messages=[{
                "role": "user",
                "content": (
                    "Summarize this conversation in 2-4 sentences, "
                    "focusing on key topics, decisions, and facts mentioned.\n\n"
                    + "\n".join(text_parts)
                ),
            }],
        )
        self.summary = response.content[0].text.strip()

    def get_messages(self) -> list[BaseMessage]:
        """Return summary + recent messages."""
        result = []
        if self.summary:
            result.append(
                SystemMessage(content=f"Conversation summary: {self.summary}")
            )
        result.extend(self.messages)
        return result

    def clear(self):
        self.messages = []
        self.summary = ""
```

### When to Use

- Conversations are long (10-50+ turns)
- Single topic throughout (summary captures the thread)
- Moderate latency is acceptable (summary LLM call)
- Need to preserve key facts from early turns

### Limitations

- Summary loses detail (specific numbers, exact phrasing)
- Summary quality depends on the summarization model
- Each summarization adds 100-200ms latency
- Topic switches make summaries incoherent

---

## Strategy 3: Vector Memory

Embed each turn and store in a vector database. When processing a new turn, retrieve the most relevant past turns.

### Implementation

```python
from langchain_openai import OpenAIEmbeddings
import numpy as np
from datetime import datetime


class VectorMemory:
    """Embed and retrieve relevant past turns."""

    def __init__(
        self,
        embedding_model: str = "text-embedding-3-small",
        max_retrieved: int = 5,
        relevance_threshold: float = 0.7,
    ):
        self.embeddings = OpenAIEmbeddings(model=embedding_model)
        self.max_retrieved = max_retrieved
        self.relevance_threshold = relevance_threshold
        self.turns: list[dict] = []  # {content, embedding, role, timestamp, turn_idx}

    def add_turn(self, role: str, content: str, turn_idx: int):
        """Add a turn to memory with its embedding."""
        embedding = self.embeddings.embed_query(content)
        self.turns.append({
            "content": content,
            "embedding": embedding,
            "role": role,
            "timestamp": datetime.now().isoformat(),
            "turn_idx": turn_idx,
        })

    def retrieve_relevant(self, query: str) -> list[dict]:
        """Retrieve the most relevant past turns for a query."""
        if not self.turns:
            return []

        query_embedding = self.embeddings.embed_query(query)

        # Compute similarities
        scored = []
        for turn in self.turns:
            similarity = np.dot(query_embedding, turn["embedding"]) / (
                np.linalg.norm(query_embedding) * np.linalg.norm(turn["embedding"])
            )
            if similarity >= self.relevance_threshold:
                scored.append((turn, similarity))

        # Sort by similarity, take top-K
        scored.sort(key=lambda x: x[1], reverse=True)
        return [turn for turn, _ in scored[:self.max_retrieved]]

    def get_context_for_query(self, query: str) -> str:
        """Get formatted context from relevant past turns."""
        relevant = self.retrieve_relevant(query)
        if not relevant:
            return ""

        # Sort by turn index for chronological order
        relevant.sort(key=lambda t: t["turn_idx"])

        lines = []
        for turn in relevant:
            role = "User" if turn["role"] == "human" else "Assistant"
            lines.append(f"{role} (turn {turn['turn_idx']}): {turn['content']}")

        return "Relevant past conversation:\n" + "\n".join(lines)

    def clear(self):
        self.turns = []
```

### When to Use

- Conversations are very long (50+ turns)
- Topics change frequently within a conversation
- User may reference something from many turns ago
- Willing to pay embedding cost per turn

### Limitations

- Embedding cost per turn (small but non-zero)
- Relevance matching may miss important turns if they are not semantically similar to the current query
- Recent context may be missed if not semantically similar (but temporally relevant)
- Cannot capture the "flow" of conversation -- only individual turn relevance

---

## Strategy 4: Entity Memory

Track named entities and their attributes across the conversation.

### Implementation

```python
import anthropic
import json


class EntityMemory:
    """Track entities and their attributes across conversation."""

    def __init__(self, model: str = "claude-3-5-haiku-20241022"):
        self.entities: dict[str, dict] = {}
        self.client = anthropic.Anthropic()
        self.model = model

    def extract_entities(self, text: str) -> dict[str, dict]:
        """Extract entities and their attributes from text."""
        response = self.client.messages.create(
            model=self.model,
            max_tokens=500,
            messages=[{
                "role": "user",
                "content": (
                    "Extract named entities and their attributes from this text.\n"
                    "Return JSON: {\"entity_name\": {\"type\": \"...\", "
                    "\"attributes\": {\"key\": \"value\"}}}\n\n"
                    f"Text: {text}"
                ),
            }],
        )
        try:
            return json.loads(response.content[0].text)
        except json.JSONDecodeError:
            return {}

    def update(self, user_message: str, ai_response: str):
        """Update entity memory from a conversation turn."""
        combined = f"User: {user_message}\nAssistant: {ai_response}"
        new_entities = self.extract_entities(combined)

        for name, info in new_entities.items():
            if name in self.entities:
                # Merge attributes
                self.entities[name]["attributes"].update(
                    info.get("attributes", {})
                )
            else:
                self.entities[name] = info

    def get_context(self) -> str:
        """Get entity memory as formatted context."""
        if not self.entities:
            return ""

        lines = ["Known entities:"]
        for name, info in self.entities.items():
            attrs = ", ".join(
                f"{k}: {v}" for k, v in info.get("attributes", {}).items()
            )
            lines.append(f"- {name} ({info.get('type', 'unknown')}): {attrs}")

        return "\n".join(lines)

    def clear(self):
        self.entities = {}
```

### When to Use

- Domain has clear entities (customers, products, servers, databases)
- Conversation builds up information about specific entities
- User asks follow-up questions about previously mentioned entities
- Examples: customer support, IT troubleshooting, project management

### Limitations

- Entity extraction is imperfect (LLM call per turn)
- Does not capture relationships between entities well
- Does not preserve conversational flow
- High latency per turn for extraction

---

## Strategy 5: Hybrid Memory (Recommended for Production)

Combine multiple strategies to get the best of each:

```python
class HybridMemory:
    """Combine sliding window + summary + vector memory."""

    def __init__(
        self,
        recent_turns: int = 5,
        summary_model: str = "claude-3-5-haiku-20241022",
        embedding_model: str = "text-embedding-3-small",
        max_vector_results: int = 3,
    ):
        self.window = SlidingWindowMemory(max_turns=recent_turns)
        self.summary = SummaryMemory(recent_turns=recent_turns, model=summary_model)
        self.vector = VectorMemory(
            embedding_model=embedding_model,
            max_retrieved=max_vector_results,
        )
        self.turn_counter = 0

    def add_turn(self, user_message: str, ai_response: str):
        """Add a conversation turn to all memory systems."""
        self.turn_counter += 1

        # Sliding window (for recent context)
        self.window.add_user_message(user_message)
        self.window.add_ai_message(ai_response)

        # Summary (for long-term compressed context)
        self.summary.add_user_message(user_message)
        self.summary.add_ai_message(ai_response)

        # Vector (for semantic retrieval of relevant past turns)
        self.vector.add_turn("human", user_message, self.turn_counter)
        self.vector.add_turn("ai", ai_response, self.turn_counter)

    def get_context(self, current_query: str) -> dict:
        """Get combined context for the current query."""
        return {
            "recent_messages": self.window.get_messages(),
            "summary": self.summary.summary,
            "relevant_past": self.vector.get_context_for_query(current_query),
        }

    def format_for_prompt(self, current_query: str) -> str:
        """Format combined memory for insertion into LLM prompt."""
        ctx = self.get_context(current_query)
        parts = []

        if ctx["summary"]:
            parts.append(f"Conversation summary: {ctx['summary']}")

        if ctx["relevant_past"]:
            parts.append(ctx["relevant_past"])

        # Recent turns are passed as messages, not text
        return "\n\n".join(parts) if parts else ""

    def clear(self):
        self.window.clear()
        self.summary.clear()
        self.vector.clear()
        self.turn_counter = 0
```

---

## Memory Store Implementations

### Redis

Fast, ephemeral, good for short-lived sessions:

```python
import redis
import json
from datetime import timedelta


class RedisMemoryStore:
    """Store conversation memory in Redis with TTL."""

    def __init__(
        self,
        redis_url: str = "redis://localhost:6379",
        session_ttl: timedelta = timedelta(hours=2),
    ):
        self.client = redis.from_url(redis_url)
        self.ttl = session_ttl

    def save_turn(self, session_id: str, turn: dict):
        key = f"chat:{session_id}:turns"
        self.client.rpush(key, json.dumps(turn))
        self.client.expire(key, self.ttl)

    def get_turns(self, session_id: str) -> list[dict]:
        key = f"chat:{session_id}:turns"
        turns = self.client.lrange(key, 0, -1)
        return [json.loads(t) for t in turns]

    def save_summary(self, session_id: str, summary: str):
        key = f"chat:{session_id}:summary"
        self.client.set(key, summary, ex=self.ttl)

    def get_summary(self, session_id: str) -> str:
        key = f"chat:{session_id}:summary"
        return (self.client.get(key) or b"").decode()

    def delete_session(self, session_id: str):
        keys = self.client.keys(f"chat:{session_id}:*")
        if keys:
            self.client.delete(*keys)
```

### PostgreSQL

Durable, queryable, good for persistent memory:

```python
import psycopg
from datetime import datetime


class PostgresMemoryStore:
    """Store conversation memory in PostgreSQL."""

    def __init__(self, connection_string: str):
        self.conn_str = connection_string
        self._init_schema()

    def _init_schema(self):
        with psycopg.connect(self.conn_str) as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS chat_turns (
                    id SERIAL PRIMARY KEY,
                    session_id TEXT NOT NULL,
                    turn_index INT NOT NULL,
                    role TEXT NOT NULL,
                    content TEXT NOT NULL,
                    embedding vector(1536),
                    created_at TIMESTAMPTZ DEFAULT now()
                );
                CREATE INDEX IF NOT EXISTS idx_chat_session
                    ON chat_turns(session_id, turn_index);

                CREATE TABLE IF NOT EXISTS chat_summaries (
                    session_id TEXT PRIMARY KEY,
                    summary TEXT NOT NULL,
                    updated_at TIMESTAMPTZ DEFAULT now()
                );
            """)

    def save_turn(self, session_id: str, turn_index: int,
                  role: str, content: str, embedding: list[float] | None = None):
        with psycopg.connect(self.conn_str) as conn:
            conn.execute(
                "INSERT INTO chat_turns (session_id, turn_index, role, content, embedding) "
                "VALUES (%s, %s, %s, %s, %s)",
                [session_id, turn_index, role, content, embedding],
            )

    def get_recent_turns(self, session_id: str, limit: int = 10) -> list[dict]:
        with psycopg.connect(self.conn_str) as conn:
            rows = conn.execute(
                "SELECT role, content, turn_index FROM chat_turns "
                "WHERE session_id = %s ORDER BY turn_index DESC LIMIT %s",
                [session_id, limit],
            ).fetchall()
            return [
                {"role": r[0], "content": r[1], "turn_index": r[2]}
                for r in reversed(rows)
            ]

    def search_similar_turns(
        self, session_id: str, query_embedding: list[float], limit: int = 5
    ) -> list[dict]:
        with psycopg.connect(self.conn_str) as conn:
            rows = conn.execute(
                "SELECT role, content, turn_index, "
                "1 - (embedding <=> %s::vector) as similarity "
                "FROM chat_turns WHERE session_id = %s AND embedding IS NOT NULL "
                "ORDER BY embedding <=> %s::vector LIMIT %s",
                [query_embedding, session_id, query_embedding, limit],
            ).fetchall()
            return [
                {"role": r[0], "content": r[1], "turn_index": r[2], "similarity": r[3]}
                for r in rows
            ]
```

### Zep

Purpose-built memory layer for LLM applications:

```python
from zep_cloud.client import Zep

zep = Zep(api_key="your-zep-key")


# Add messages to a session
async def add_to_zep(session_id: str, role: str, content: str):
    from zep_cloud.types import Message
    await zep.memory.add(
        session_id=session_id,
        messages=[Message(role_type=role, content=content)],
    )


# Get memory (Zep auto-summarizes and extracts entities)
async def get_zep_memory(session_id: str) -> dict:
    memory = await zep.memory.get(session_id=session_id)
    return {
        "summary": memory.summary.content if memory.summary else "",
        "messages": [
            {"role": m.role_type, "content": m.content}
            for m in memory.messages
        ],
        "facts": memory.facts,  # Extracted facts
    }


# Search memory by relevance
async def search_zep_memory(session_id: str, query: str) -> list:
    results = await zep.memory.search(
        session_id=session_id,
        text=query,
        search_type="similarity",
        limit=5,
    )
    return results
```

### mem0

AI-native memory layer with automatic fact extraction:

```python
from mem0 import Memory

m = Memory()

# Add memories from conversation
m.add(
    "User prefers PostgreSQL over MySQL for new projects",
    user_id="user_123",
    metadata={"source": "conversation", "session": "abc"},
)

# Search memories
results = m.search("database preferences", user_id="user_123")
for r in results:
    print(f"Memory: {r['memory']} (relevance: {r['score']:.3f})")

# Get all memories for a user
all_memories = m.get_all(user_id="user_123")
```

---

## LangChain Memory Types Comparison

| Memory Type | Class | Mechanism | Token Cost | Use Case |
|-------------|-------|-----------|------------|----------|
| Buffer | `ConversationBufferMemory` | Store all messages | Grows linearly | Short conversations |
| Buffer Window | `ConversationBufferWindowMemory` | Last K turns | Fixed (K turns) | Short to medium |
| Summary | `ConversationSummaryMemory` | LLM summarization | Fixed (summary size) | Long, single-topic |
| Summary Buffer | `ConversationSummaryBufferMemory` | Summary + recent | Fixed | Long, needs detail |
| Token Buffer | `ConversationTokenBufferMemory` | Trim by token count | Fixed | Token-constrained |
| Entity | `ConversationEntityMemory` | Track entities | Fixed | Entity-centric |
| Vector Store | `VectorStoreRetrieverMemory` | Embed + retrieve | Variable | Long, diverse topics |

---

## Common Pitfalls

1. **Using buffer memory for long conversations.** Buffer memory grows linearly and eventually exceeds the context window. Use summary or hybrid memory for conversations beyond 10 turns.
2. **Summarizing too aggressively.** A one-sentence summary of 20 turns loses critical details. Use summary + recent window to preserve recent detail.
3. **Not handling topic switches.** When the user switches topics, old summary context may confuse the model. Detect topic switches and reset or segment the summary.
4. **Embedding every message for vector memory.** Short messages like "yes", "ok", "thanks" produce meaningless embeddings. Only embed substantive messages (>20 tokens).
5. **Using the same memory strategy for all use cases.** A customer support bot (entity-centric, long) needs different memory than a code assistant (context-window focused, short). Match the strategy to the use case.
6. **Not setting session TTLs.** Abandoned sessions consume memory indefinitely. Set TTLs (2-24 hours) in Redis/Postgres and clean up expired sessions.
7. **Trusting entity extraction blindly.** LLM-based entity extraction makes mistakes. Use entity memory as supplementary context, not as the sole source of truth.

---

## References

- LangChain memory: https://python.langchain.com/docs/how_to/#memory
- LlamaIndex chat memory: https://docs.llamaindex.ai/en/stable/module_guides/storing/chat_stores/
- Zep: https://www.getzep.com/
- mem0: https://github.com/mem0ai/mem0
- Park et al. "Generative Agents: Interactive Simulacra of Human Behavior" (UIST 2023) -- pioneered reflective memory for LLM agents
