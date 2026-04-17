# Query Rewriting for Conversational RAG

## TL;DR

Query rewriting transforms a context-dependent user message ("How do I configure it?") into a standalone query ("How do I configure row-level security in PostgreSQL?") by resolving coreferences, ellipsis, and implicit context from the chat history. This is the single most impactful component for multi-turn RAG quality -- without it, the retriever receives ambiguous queries and returns irrelevant documents. This article covers rewriting techniques, framework implementations (LangChain, LlamaIndex), custom rewriting with Claude, evaluation methods, and handling topic switches.

---

## Why Rewriting Matters

### The Problem Without Rewriting

```
Turn 1: User: "Tell me about PostgreSQL RLS"
         -> Retriever query: "Tell me about PostgreSQL RLS"
         -> Retrieves: RLS documentation (correct)

Turn 2: User: "How do I configure it?"
         -> Retriever query: "How do I configure it?"
         -> Retrieves: generic "configuration" docs (wrong)
         -> Expected: RLS configuration docs

Turn 3: User: "What about performance?"
         -> Retriever query: "What about performance?"
         -> Retrieves: generic performance docs (wrong)
         -> Expected: PostgreSQL RLS performance impact
```

### With Rewriting

```
Turn 2: User: "How do I configure it?"
         -> Rewritten: "How do I configure row-level security in PostgreSQL?"
         -> Retrieves: RLS configuration docs (correct)

Turn 3: User: "What about performance?"
         -> Rewritten: "What is the performance impact of row-level security in PostgreSQL?"
         -> Retrieves: RLS performance docs (correct)
```

### Types of Context Dependencies

| Dependency Type | Example | What Rewriting Must Do |
|----------------|---------|----------------------|
| Pronoun reference | "How do I configure **it**?" | Resolve "it" -> "RLS" |
| Ellipsis | "And for MySQL?" | Expand to "How does MySQL handle row-level security?" |
| Topic continuation | "What about performance?" | Add "of PostgreSQL RLS" |
| Implicit context | "Show me an example" | Determine what kind of example from history |
| Negation reference | "What if I don't want that?" | Resolve "that" from history |
| Attribute reference | "How about the newer version?" | Resolve which technology + version |

---

## Rewriting Techniques

### Technique 1: LLM-Based Rewriting (Recommended)

Use an LLM to rewrite the query based on chat history. Most flexible and accurate.

```python
import anthropic

client = anthropic.Anthropic()

REWRITE_PROMPT = """\
Given the conversation history and the user's latest message, rewrite the message as a standalone search query that incorporates all necessary context.

Rules:
- Resolve all pronouns (it, they, that, this) to their referents
- Expand elliptical questions to full questions
- Include the topic/subject from history if not in the current message
- Do NOT answer the question -- only rewrite it
- If the message is already standalone, return it unchanged
- Keep the rewritten query concise (under 30 words)

Conversation history:
{history}

Latest message: {message}

Standalone query:"""


def rewrite_query(
    message: str,
    chat_history: list[dict],
    model: str = "claude-3-5-haiku-20241022",
) -> str:
    """Rewrite a context-dependent message into a standalone query."""
    if not chat_history:
        return message

    # Format history
    history_text = "\n".join(
        f"{'User' if turn['role'] == 'user' else 'Assistant'}: {turn['content']}"
        for turn in chat_history[-10:]  # Last 10 messages
    )

    response = client.messages.create(
        model=model,
        max_tokens=100,
        messages=[{
            "role": "user",
            "content": REWRITE_PROMPT.format(
                history=history_text,
                message=message,
            ),
        }],
    )
    return response.content[0].text.strip()


# Example
history = [
    {"role": "user", "content": "Tell me about PostgreSQL row-level security"},
    {"role": "assistant", "content": "PostgreSQL RLS uses CREATE POLICY to define..."},
]

rewritten = rewrite_query("How do I configure it?", history)
# "How do I configure row-level security in PostgreSQL?"

rewritten = rewrite_query("What about performance impact?", history)
# "What is the performance impact of row-level security in PostgreSQL?"

rewritten = rewrite_query("And for MySQL?", history)
# "How does MySQL handle row-level security?"
```

### Technique 2: Template-Based Rewriting

For simpler applications, use templates with slot-filling:

```python
import re


def template_rewrite(
    message: str,
    chat_history: list[dict],
) -> str:
    """Simple template-based query rewriting."""

    # Extract the most recent topic from history
    recent_topic = extract_topic(chat_history)
    if not recent_topic:
        return message

    message_lower = message.lower()

    # Pattern 1: Pronoun resolution
    # "How do I configure it?" -> "How do I configure {topic}?"
    pronouns = ["it", "this", "that", "them", "those"]
    for pronoun in pronouns:
        pattern = rf'\b{pronoun}\b'
        if re.search(pattern, message_lower):
            return re.sub(pattern, recent_topic, message, flags=re.IGNORECASE)

    # Pattern 2: Short follow-up
    # "And for MySQL?" -> "{topic} for MySQL?"
    if message_lower.startswith(("and ", "what about ", "how about ")):
        prefix = re.match(r'^(and |what about |how about )', message_lower)
        remainder = message[len(prefix.group()):].strip()
        return f"{recent_topic} {remainder}"

    # Pattern 3: Missing subject
    # "What about performance?" -> "What about {topic} performance?"
    if len(message.split()) <= 5 and recent_topic.lower() not in message_lower:
        return f"{message.rstrip('?')} of {recent_topic}?"

    return message


def extract_topic(chat_history: list[dict]) -> str:
    """Extract the main topic from recent chat history."""
    # Simple: use the noun phrase from the first user message
    for turn in reversed(chat_history):
        if turn["role"] == "user":
            # Extract key noun phrase (simplified)
            text = turn["content"]
            # Remove common question words
            for prefix in ["tell me about", "what is", "how does", "explain"]:
                if text.lower().startswith(prefix):
                    return text[len(prefix):].strip().rstrip("?.")
            return text
    return ""
```

### Technique 3: Embedding-Based Rewriting

Concatenate the query with relevant history and let the embedding model handle context:

```python
def embedding_rewrite(
    message: str,
    chat_history: list[dict],
    max_history_chars: int = 500,
) -> str:
    """Create an enriched query by prepending relevant history."""
    if not chat_history:
        return message

    # Take the most recent exchange
    recent = chat_history[-4:]  # Last 2 turns
    history_text = " ".join(
        turn["content"] for turn in recent
    )[:max_history_chars]

    # Concatenate for embedding
    enriched = f"{history_text} {message}"
    return enriched
```

**Note**: this does not produce a human-readable rewrite. It works only when the "rewritten" query is used directly for embedding, not displayed to the user or used for BM25 keyword search.

---

## Framework Implementations

### LangChain: create_history_aware_retriever

```python
from langchain.chains import create_history_aware_retriever
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")

contextualize_prompt = ChatPromptTemplate.from_messages([
    ("system",
     "Given the chat history and the latest user question, "
     "formulate a standalone question that can be understood "
     "without the chat history. Do NOT answer the question. "
     "If it is already standalone, return it as-is."),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])

history_aware_retriever = create_history_aware_retriever(
    llm, retriever, contextualize_prompt
)

# Usage
from langchain_core.messages import HumanMessage, AIMessage

result = history_aware_retriever.invoke({
    "input": "How do I configure it?",
    "chat_history": [
        HumanMessage(content="Tell me about PostgreSQL RLS"),
        AIMessage(content="PostgreSQL RLS uses CREATE POLICY..."),
    ],
})
# Internally rewrites "it" -> "RLS in PostgreSQL" before retrieving
```

### LlamaIndex: CondensePlusContextChatEngine

```python
from llama_index.core.chat_engine import CondensePlusContextChatEngine
from llama_index.core.memory import ChatMemoryBuffer

memory = ChatMemoryBuffer.from_defaults(token_limit=3000)

chat_engine = CondensePlusContextChatEngine.from_defaults(
    retriever=index.as_retriever(),
    memory=memory,
    condense_prompt=(
        "Given the following conversation and a follow-up question, "
        "rephrase the follow-up question to be a standalone question.\n\n"
        "Chat History:\n{chat_history_str}\n\n"
        "Follow Up Input: {question}\n\n"
        "Standalone question:"
    ),
)

# Turn 1
response = chat_engine.chat("Tell me about PostgreSQL RLS")
# Turn 2 (auto-rewritten internally)
response = chat_engine.chat("How do I configure it?")
```

### Custom Claude Rewriter with Caching

For production, cache rewrites to avoid redundant LLM calls:

```python
from functools import lru_cache
import hashlib


class CachedQueryRewriter:
    """Query rewriter with caching for repeated patterns."""

    def __init__(self, model: str = "claude-3-5-haiku-20241022"):
        self.client = anthropic.Anthropic()
        self.model = model
        self._cache: dict[str, str] = {}

    def _cache_key(self, message: str, history_hash: str) -> str:
        combined = f"{history_hash}::{message}"
        return hashlib.md5(combined.encode()).hexdigest()

    def _history_hash(self, history: list[dict]) -> str:
        # Hash last 3 turns for cache key
        recent = history[-6:] if len(history) > 6 else history
        text = "|".join(f"{t['role']}:{t['content'][:50]}" for t in recent)
        return hashlib.md5(text.encode()).hexdigest()

    def rewrite(self, message: str, chat_history: list[dict]) -> str:
        """Rewrite with caching."""
        if not chat_history:
            return message

        # Check if message is likely standalone (no rewrite needed)
        if self._is_standalone(message):
            return message

        h_hash = self._history_hash(chat_history)
        cache_key = self._cache_key(message, h_hash)

        if cache_key in self._cache:
            return self._cache[cache_key]

        rewritten = rewrite_query(message, chat_history, self.model)
        self._cache[cache_key] = rewritten
        return rewritten

    def _is_standalone(self, message: str) -> bool:
        """Heuristic: check if a message is likely self-contained."""
        message_lower = message.lower()

        # Contains pronouns that likely need resolution
        context_dependent = [
            r'\bit\b', r'\bthis\b', r'\bthat\b', r'\bthem\b',
            r'\bthose\b', r'\bthe same\b', r'\babove\b',
        ]
        for pattern in context_dependent:
            if re.search(pattern, message_lower):
                return False

        # Very short follow-up
        if len(message.split()) <= 3:
            return False

        # Starts with continuation words
        continuation_prefixes = [
            "and ", "also ", "what about ", "how about ",
            "same for ", "now ", "then ",
        ]
        if any(message_lower.startswith(p) for p in continuation_prefixes):
            return False

        # Likely standalone if it is a full question with a subject
        if len(message.split()) >= 6 and "?" in message:
            return True

        return True  # Default: assume standalone
```

---

## Evaluating Rewrite Quality

### Automated Evaluation

```python
def evaluate_rewrite_quality(
    test_cases: list[dict],
    rewriter_fn,
) -> dict:
    """
    Evaluate rewrite quality.

    test_cases: [{
        "history": [...],
        "message": str,
        "expected_standalone": str,
        "expected_entities": list[str],  # entities that must appear in rewrite
    }]
    """
    results = {
        "entity_recall": [],     # Do expected entities appear in the rewrite?
        "semantic_similarity": [], # How similar is rewrite to expected?
        "is_standalone": [],     # Does the rewrite make sense without history?
        "not_an_answer": [],     # Does the rewrite NOT contain an answer?
    }

    for tc in test_cases:
        rewritten = rewriter_fn(tc["message"], tc["history"])

        # Entity recall: all expected entities should appear
        entities_found = sum(
            1 for e in tc["expected_entities"]
            if e.lower() in rewritten.lower()
        )
        entity_recall = entities_found / len(tc["expected_entities"]) if tc["expected_entities"] else 1.0
        results["entity_recall"].append(entity_recall)

        # Semantic similarity to expected standalone
        from sentence_transformers import SentenceTransformer
        model = SentenceTransformer("all-MiniLM-L6-v2")
        emb_rewritten = model.encode([rewritten])[0]
        emb_expected = model.encode([tc["expected_standalone"]])[0]
        import numpy as np
        similarity = np.dot(emb_rewritten, emb_expected) / (
            np.linalg.norm(emb_rewritten) * np.linalg.norm(emb_expected)
        )
        results["semantic_similarity"].append(similarity)

        # Check it is not an answer (should be a question/query)
        is_query = "?" in rewritten or len(rewritten.split()) < 20
        results["not_an_answer"].append(float(is_query))

    return {k: sum(v) / len(v) for k, v in results.items()}


# Test cases
test_cases = [
    {
        "history": [
            {"role": "user", "content": "Tell me about PostgreSQL RLS"},
            {"role": "assistant", "content": "PostgreSQL RLS uses CREATE POLICY..."},
        ],
        "message": "How do I configure it?",
        "expected_standalone": "How do I configure row-level security in PostgreSQL?",
        "expected_entities": ["PostgreSQL", "row-level security"],
    },
    {
        "history": [
            {"role": "user", "content": "What are Kafka retention settings?"},
            {"role": "assistant", "content": "Kafka retains messages for 7 days..."},
        ],
        "message": "And for RabbitMQ?",
        "expected_standalone": "What are the message retention settings in RabbitMQ?",
        "expected_entities": ["RabbitMQ", "retention"],
    },
    {
        "history": [
            {"role": "user", "content": "How does connection pooling work in PostgreSQL?"},
            {"role": "assistant", "content": "PgBouncer manages a pool of connections..."},
        ],
        "message": "What about performance impact?",
        "expected_standalone": "What is the performance impact of connection pooling in PostgreSQL?",
        "expected_entities": ["PostgreSQL", "connection pooling", "performance"],
    },
]

metrics = evaluate_rewrite_quality(test_cases, rewrite_query)
print(f"Entity recall: {metrics['entity_recall']:.3f}")
print(f"Semantic similarity: {metrics['semantic_similarity']:.3f}")
```

### End-to-End Retrieval Evaluation

The ultimate test: does the rewritten query retrieve better documents?

```python
def evaluate_rewrite_retrieval_impact(
    test_cases: list[dict],
    retriever,
    rewriter_fn,
) -> dict:
    """Compare retrieval quality with and without rewriting."""
    results_with_rewrite = []
    results_without_rewrite = []

    for tc in test_cases:
        relevant_docs = set(tc["relevant_doc_ids"])

        # Without rewrite: retrieve using raw message
        raw_docs = retriever.invoke(tc["message"])
        raw_ids = {doc.metadata.get("doc_id") for doc in raw_docs[:5]}
        recall_without = len(raw_ids & relevant_docs) / len(relevant_docs) if relevant_docs else 0
        results_without_rewrite.append(recall_without)

        # With rewrite
        rewritten = rewriter_fn(tc["message"], tc["history"])
        rewritten_docs = retriever.invoke(rewritten)
        rewritten_ids = {doc.metadata.get("doc_id") for doc in rewritten_docs[:5]}
        recall_with = len(rewritten_ids & relevant_docs) / len(relevant_docs) if relevant_docs else 0
        results_with_rewrite.append(recall_with)

    import numpy as np
    print(f"Recall@5 without rewriting: {np.mean(results_without_rewrite):.3f}")
    print(f"Recall@5 with rewriting:    {np.mean(results_with_rewrite):.3f}")
    print(f"Improvement:                {np.mean(results_with_rewrite) - np.mean(results_without_rewrite):.3f}")

    return {
        "recall_without": np.mean(results_without_rewrite),
        "recall_with": np.mean(results_with_rewrite),
    }
```

---

## Handling Topic Switches

### The Problem

```
Turn 1-3: Discussion about PostgreSQL RLS
Turn 4: "Actually, I want to know about Kafka message ordering"
         -> Rewriter should NOT add "in PostgreSQL RLS"
         -> Should recognize this is a new topic
```

### Topic Switch Detection

```python
def detect_topic_switch(
    message: str,
    chat_history: list[dict],
    model: str = "claude-3-5-haiku-20241022",
) -> bool:
    """Detect if the user is switching to a new topic."""
    if not chat_history:
        return False

    # Heuristic 1: Explicit switch signals
    switch_signals = [
        "actually", "by the way", "different question",
        "something else", "new topic", "unrelated",
        "switching to", "let's talk about", "moving on",
        "forget about", "different thing",
    ]
    message_lower = message.lower()
    if any(signal in message_lower for signal in switch_signals):
        return True

    # Heuristic 2: Semantic similarity between message and recent history
    # Low similarity = likely topic switch
    from sentence_transformers import SentenceTransformer
    import numpy as np

    model_st = SentenceTransformer("all-MiniLM-L6-v2")

    recent_text = " ".join(
        t["content"] for t in chat_history[-4:] if t["role"] == "user"
    )
    emb_message = model_st.encode([message])[0]
    emb_history = model_st.encode([recent_text])[0]

    similarity = np.dot(emb_message, emb_history) / (
        np.linalg.norm(emb_message) * np.linalg.norm(emb_history)
    )

    if similarity < 0.3:  # Low similarity = topic switch
        return True

    return False


def rewrite_with_topic_awareness(
    message: str,
    chat_history: list[dict],
) -> str:
    """Rewrite query with topic switch detection."""
    if detect_topic_switch(message, chat_history):
        # New topic: do not incorporate history context
        # But still clean up the query
        return message.lstrip("actually, ").lstrip("by the way, ").strip()
    else:
        # Same topic: full rewriting with history context
        return rewrite_query(message, chat_history)
```

### Handling Partial Topic Switches

Sometimes the user shifts a specific aspect while keeping the general topic:

```
Turn 1: "How does PostgreSQL RLS work?"
Turn 2: "How does the same thing work in MySQL?"
         -> Not a full topic switch -- still about row-level security
         -> Rewrite: "How does row-level security work in MySQL?"
```

The LLM-based rewriter handles this naturally because it understands that "the same thing" refers to RLS and "in MySQL" is the new context.

---

## Performance Optimization

### Skip Rewriting When Not Needed

```python
def needs_rewriting(message: str) -> bool:
    """Quick check if a message needs rewriting."""
    message_lower = message.lower()

    # Explicit pronouns/references
    if re.search(r'\b(it|this|that|them|those|the same|above)\b', message_lower):
        return True

    # Short follow-ups
    if len(message.split()) <= 4:
        return True

    # Continuation words
    if message_lower.startswith(("and ", "also ", "what about ", "how about ")):
        return True

    # Full standalone question with specific subject
    if len(message.split()) >= 8 and "?" in message:
        return False

    return True  # Default: rewrite to be safe


def conditional_rewrite(
    message: str,
    chat_history: list[dict],
) -> str:
    """Only rewrite when necessary."""
    if not needs_rewriting(message):
        return message
    return rewrite_query(message, chat_history)
```

### Batch Rewriting

When processing multiple queries (e.g., in evaluation):

```python
async def batch_rewrite(
    messages: list[str],
    histories: list[list[dict]],
    model: str = "claude-3-5-haiku-20241022",
) -> list[str]:
    """Rewrite multiple queries in parallel."""
    import asyncio

    async def single_rewrite(msg, hist):
        if not needs_rewriting(msg) or not hist:
            return msg
        return rewrite_query(msg, hist, model)

    tasks = [single_rewrite(m, h) for m, h in zip(messages, histories)]
    return await asyncio.gather(*tasks)
```

---

## Common Pitfalls

1. **Rewriting standalone queries.** Rewriting "What are the ACID properties of PostgreSQL?" is wasteful and can introduce errors. Use the `needs_rewriting()` heuristic to skip standalone queries.
2. **Using the entire history for rewriting.** Only the last 3-5 turns are typically relevant. Passing 50 turns adds latency and can confuse the rewriter.
3. **Rewriter generating an answer instead of a query.** The rewriting prompt must explicitly say "Do NOT answer the question." Without this, the LLM sometimes generates a full answer.
4. **Not handling topic switches.** Carrying PostgreSQL context into a Kafka question produces wrong retrieval. Detect and handle topic switches explicitly.
5. **Ignoring rewrite quality evaluation.** A bad rewrite is worse than no rewrite. Measure entity recall and retrieval impact to ensure your rewriter is helping.
6. **Expensive models for rewriting.** Rewriting is a simple task -- Haiku-class models perform nearly as well as Sonnet for 10x less cost. Use the smallest model that maintains quality.
7. **Not preserving the original query.** Always keep the original user message alongside the rewrite. The original is needed for chat display, logging, and debugging. The rewrite is only for retrieval.

---

## References

- Vakulenko et al. "Question Rewriting for Conversational Question Answering" (WSDM 2021)
- Anantha et al. "Open-Domain Question Answering Goes Conversational via Question Rewriting" (NAACL 2021)
- Elgohary et al. "Can You Unpack That? Learning to Rewrite Questions-in-Context" (EMNLP 2019)
- LangChain history-aware retriever: https://python.langchain.com/docs/how_to/qa_chat_history_how_to/
- LlamaIndex chat engines: https://docs.llamaindex.ai/en/stable/module_guides/deploying/chat_engines/
