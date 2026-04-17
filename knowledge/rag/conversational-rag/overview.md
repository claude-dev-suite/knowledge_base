# Conversational RAG -- Multi-Turn Architecture

## TL;DR

Conversational RAG extends single-turn RAG to multi-turn dialogues where context accumulates across turns. The core challenge: when a user says "How do I configure it?", the retriever must understand that "it" refers to an entity mentioned three turns ago. This requires query rewriting (resolving references using chat history), memory management (deciding what to keep from previous turns), and context budget management (fitting history, retrieved documents, and instructions into a limited context window). This overview covers the architecture, the two main approaches (history-aware retrieval and standalone question reformulation), and integration patterns with LangChain and LlamaIndex.

---

## The Multi-Turn Problem

### Single-Turn RAG (No History)

```
User: "How does PostgreSQL handle row-level security?"
  -> Retrieve docs about PostgreSQL RLS
  -> Generate answer

Works perfectly. One query, one retrieval, one answer.
```

### Multi-Turn RAG (With History)

```
Turn 1:
  User: "How does PostgreSQL handle row-level security?"
  -> Retrieve docs about PostgreSQL RLS
  -> Generate answer about RLS

Turn 2:
  User: "How do I configure it?"
  -> Retrieve docs about... "it"???
  -> "it" = RLS, but the retriever does not know that
  -> Retrieves generic "configuration" docs -> bad answer

Turn 3:
  User: "What about performance impact?"
  -> "What" about "performance impact" of what?
  -> Need to know: performance impact of PostgreSQL RLS
  -> Without history context, retrieves generic performance docs
```

### Why This Is Hard

| Challenge | Description |
|-----------|------------|
| Pronoun resolution | "it", "that", "this", "they" refer to entities from prior turns |
| Ellipsis | "And for MySQL?" (ellipsis of "How does MySQL handle row-level security?") |
| Topic continuity | Subsequent questions assume the topic from prior turns |
| Topic switches | User may change topic abruptly; system must detect this |
| Context accumulation | Each turn may add information the user expects to be "remembered" |
| Context window limits | History + retrieved docs + system prompt must fit in the LLM's context |

---

## Architecture Options

### Option A: Standalone Question Reformulation

Rewrite the user's message into a self-contained query before retrieval.

```
Chat History:
  User: "How does PostgreSQL handle row-level security?"
  Assistant: "PostgreSQL uses CREATE POLICY..."

New User Message: "How do I configure it?"

Rewritten Query: "How do I configure row-level security in PostgreSQL?"

-> Retrieve using the rewritten query
-> Generate answer
```

**Pros**: clean separation between history processing and retrieval. Retriever operates on well-formed queries.

**Cons**: rewrites can lose nuance or misinterpret the reference. Extra LLM call per turn.

### Option B: History-Aware Retrieval

Pass both the query and history to the retriever (or embed the combined text).

```
Retrieval input: "How do I configure it?" + [history about PostgreSQL RLS]
-> Embedding captures RLS context
-> Retrieves relevant docs
```

**Pros**: no information loss from rewriting. Simpler pipeline (no extra LLM call).

**Cons**: history makes the query longer, which may dilute the embedding. Works better with models trained for conversational context.

### Option C: Hybrid (Recommended)

Rewrite the query for the retriever AND pass history to the generator.

```
1. Rewrite query: "How do I configure it?" -> "How to configure PostgreSQL RLS?"
2. Retrieve using rewritten query
3. Generate answer using: rewritten query + history + retrieved docs

The generator sees history for continuity.
The retriever sees a clean query for precise retrieval.
```

---

## Implementation with LangChain

### create_history_aware_retriever

LangChain provides a built-in function that rewrites queries using chat history:

```python
from langchain.chains import create_history_aware_retriever, create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings


# Setup
llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")
vectorstore = Chroma.from_documents(documents, OpenAIEmbeddings())
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})


# Step 1: History-aware retriever (query rewriting)
contextualize_prompt = ChatPromptTemplate.from_messages([
    ("system",
     "Given the chat history and the latest user question, "
     "reformulate the question to be a standalone question that "
     "can be understood without the chat history. Do NOT answer the question, "
     "just reformulate it if needed. If it is already standalone, return it as-is."),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])

history_aware_retriever = create_history_aware_retriever(
    llm, retriever, contextualize_prompt
)


# Step 2: Answer generation chain
answer_prompt = ChatPromptTemplate.from_messages([
    ("system",
     "You are a helpful assistant. Answer the question based on the "
     "provided context. If you cannot answer from the context, say so.\n\n"
     "Context:\n{context}"),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])

question_answer_chain = create_stuff_documents_chain(llm, answer_prompt)


# Step 3: Full conversational RAG chain
rag_chain = create_retrieval_chain(
    history_aware_retriever, question_answer_chain
)


# Usage
from langchain_core.messages import HumanMessage, AIMessage

chat_history = []

# Turn 1
response = rag_chain.invoke({
    "input": "How does PostgreSQL handle row-level security?",
    "chat_history": chat_history,
})
print(response["answer"])

chat_history.extend([
    HumanMessage(content="How does PostgreSQL handle row-level security?"),
    AIMessage(content=response["answer"]),
])

# Turn 2
response = rag_chain.invoke({
    "input": "How do I configure it?",
    "chat_history": chat_history,
})
print(response["answer"])
# The history-aware retriever rewrites "it" -> "RLS in PostgreSQL"
```

### Managing Chat History Size

As conversations grow, history exceeds the context window. Use a sliding window or summary:

```python
from langchain_core.messages import BaseMessage


def manage_history(
    chat_history: list[BaseMessage],
    max_turns: int = 10,
    max_tokens: int = 2000,
) -> list[BaseMessage]:
    """Keep only the most recent turns within token budget."""
    # Simple: sliding window
    if len(chat_history) > max_turns * 2:  # Each turn = human + AI
        return chat_history[-(max_turns * 2):]
    return chat_history


def summarize_old_history(
    chat_history: list[BaseMessage],
    recent_turns: int = 5,
    llm=None,
) -> list[BaseMessage]:
    """Summarize old turns, keep recent ones verbatim."""
    if len(chat_history) <= recent_turns * 2:
        return chat_history

    old_messages = chat_history[:-(recent_turns * 2)]
    recent_messages = chat_history[-(recent_turns * 2):]

    # Summarize old messages
    old_text = "\n".join(
        f"{'User' if isinstance(m, HumanMessage) else 'Assistant'}: {m.content}"
        for m in old_messages
    )

    summary_response = llm.invoke(
        f"Summarize this conversation history in 2-3 sentences, "
        f"focusing on the key topics and facts discussed:\n\n{old_text}"
    )

    from langchain_core.messages import SystemMessage
    summary_message = SystemMessage(
        content=f"Previous conversation summary: {summary_response.content}"
    )

    return [summary_message] + recent_messages
```

---

## Implementation with LlamaIndex

### CondensePlusContextChatEngine

LlamaIndex's `CondensePlusContextChatEngine` combines query condensation (rewriting) with context retrieval:

```python
from llama_index.core import VectorStoreIndex
from llama_index.core.chat_engine import CondensePlusContextChatEngine
from llama_index.core.memory import ChatMemoryBuffer

# Build index
index = VectorStoreIndex.from_documents(documents)

# Create chat engine with memory
memory = ChatMemoryBuffer.from_defaults(token_limit=3000)

chat_engine = CondensePlusContextChatEngine.from_defaults(
    index.as_retriever(similarity_top_k=5),
    memory=memory,
    llm=llm,
    context_prompt=(
        "You are a helpful assistant. Use the following context to answer.\n"
        "Context:\n{context_str}\n\n"
    ),
    condense_prompt=(
        "Given the conversation so far and the latest message, "
        "rewrite the message as a standalone question.\n\n"
        "Conversation:\n{chat_history_str}\n\n"
        "Latest message: {question}\n\n"
        "Standalone question:"
    ),
)

# Multi-turn conversation
response1 = chat_engine.chat("How does PostgreSQL handle row-level security?")
print(response1.response)

response2 = chat_engine.chat("How do I configure it?")
print(response2.response)
# Internally: "it" is resolved to "RLS in PostgreSQL" before retrieval

response3 = chat_engine.chat("What about performance impact?")
print(response3.response)
# "performance impact of PostgreSQL RLS"
```

### ContextChatEngine (No Rewriting)

For simpler use cases where you trust the embedding model to handle context:

```python
from llama_index.core.chat_engine import ContextChatEngine

chat_engine = ContextChatEngine.from_defaults(
    retriever=index.as_retriever(),
    memory=memory,
    llm=llm,
)
# Passes raw query + history to retriever; no rewriting step
```

---

## Token Budget Management

### The Budget Constraint

A typical LLM context window (128K tokens for Claude, 128K for GPT-4) must contain:

```
[System prompt]         ~200-500 tokens
[Chat history]          Variable (grows with conversation)
[Retrieved documents]   Variable (k docs * avg_doc_size)
[Current query]         ~20-100 tokens
[Space for generation]  ~500-2000 tokens (max_tokens)

Total = system + history + docs + query + generation <= context_window
```

### Budget Allocation Strategy

```python
def allocate_token_budget(
    context_window: int = 128000,
    system_prompt_tokens: int = 300,
    max_generation_tokens: int = 1000,
    current_query_tokens: int = 50,
    num_docs: int = 5,
    avg_doc_tokens: int = 500,
) -> dict:
    """Calculate available token budget for chat history."""
    reserved = (
        system_prompt_tokens
        + max_generation_tokens
        + current_query_tokens
        + (num_docs * avg_doc_tokens)
    )
    available_for_history = context_window - reserved

    return {
        "context_window": context_window,
        "system_prompt": system_prompt_tokens,
        "retrieved_docs": num_docs * avg_doc_tokens,
        "current_query": current_query_tokens,
        "generation_reserve": max_generation_tokens,
        "available_for_history": available_for_history,
        "max_history_turns": available_for_history // 200,  # ~200 tokens per turn
    }

budget = allocate_token_budget()
# With 128K context:
# available_for_history ≈ 124,150 tokens
# max_history_turns ≈ 620 turns
#
# In practice, you should cap much lower to leave room for longer
# documents and edge cases
```

### Dynamic Budget Management

```python
import tiktoken


class TokenBudgetManager:
    """Manage token budget across conversation components."""

    def __init__(
        self,
        context_window: int = 128000,
        system_prompt: str = "",
        max_generation: int = 1000,
        doc_budget_pct: float = 0.30,   # 30% for retrieved docs
        history_budget_pct: float = 0.50, # 50% for history
    ):
        self.context_window = context_window
        self.tokenizer = tiktoken.encoding_for_model("gpt-4")
        self.system_tokens = len(self.tokenizer.encode(system_prompt))
        self.max_generation = max_generation

        available = context_window - self.system_tokens - max_generation
        self.doc_budget = int(available * doc_budget_pct)
        self.history_budget = int(available * history_budget_pct)

    def count_tokens(self, text: str) -> int:
        return len(self.tokenizer.encode(text))

    def fit_history(self, messages: list[dict]) -> list[dict]:
        """Trim history to fit within budget."""
        total = 0
        fitted = []
        for msg in reversed(messages):  # Keep most recent first
            msg_tokens = self.count_tokens(msg["content"])
            if total + msg_tokens > self.history_budget:
                break
            fitted.insert(0, msg)
            total += msg_tokens
        return fitted

    def fit_documents(self, documents: list[str]) -> list[str]:
        """Trim documents to fit within budget."""
        total = 0
        fitted = []
        for doc in documents:
            doc_tokens = self.count_tokens(doc)
            if total + doc_tokens > self.doc_budget:
                # Truncate this document to fit remaining budget
                remaining = self.doc_budget - total
                if remaining > 100:
                    tokens = self.tokenizer.encode(doc)[:remaining]
                    fitted.append(self.tokenizer.decode(tokens))
                break
            fitted.append(doc)
            total += doc_tokens
        return fitted
```

---

## End-to-End Conversational RAG Pipeline

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder


class ConversationalRAG:
    """Complete conversational RAG pipeline."""

    def __init__(
        self,
        retriever,
        llm_model: str = "claude-3-5-sonnet-20241022",
        max_history_turns: int = 10,
    ):
        self.retriever = retriever
        self.llm = ChatAnthropic(model=llm_model)
        self.max_history_turns = max_history_turns
        self.chat_history: list = []

    def _rewrite_query(self, user_message: str) -> str:
        """Rewrite user message to standalone query using history."""
        if not self.chat_history:
            return user_message

        prompt = ChatPromptTemplate.from_messages([
            ("system",
             "Rewrite the user's latest message as a standalone search query "
             "that incorporates relevant context from the chat history. "
             "Output ONLY the rewritten query, nothing else."),
            MessagesPlaceholder("history"),
            ("human", "{message}"),
        ])

        history = self.chat_history[-(self.max_history_turns * 2):]
        response = self.llm.invoke(
            prompt.format_messages(history=history, message=user_message)
        )
        return response.content.strip()

    def _retrieve(self, query: str) -> list:
        """Retrieve documents using the rewritten query."""
        return self.retriever.invoke(query)

    def _generate(
        self, user_message: str, rewritten_query: str, documents: list
    ) -> str:
        """Generate answer using history, docs, and query."""
        context = "\n\n".join([doc.page_content for doc in documents])

        prompt = ChatPromptTemplate.from_messages([
            ("system",
             "You are a helpful assistant. Answer based on the provided context. "
             "If the context does not contain the answer, say so.\n\n"
             f"Retrieved context:\n{context}"),
            MessagesPlaceholder("history"),
            ("human", "{message}"),
        ])

        history = self.chat_history[-(self.max_history_turns * 2):]
        response = self.llm.invoke(
            prompt.format_messages(history=history, message=user_message)
        )
        return response.content

    def chat(self, user_message: str) -> dict:
        """Process a single chat turn."""
        # Step 1: Rewrite query
        rewritten = self._rewrite_query(user_message)

        # Step 2: Retrieve
        documents = self._retrieve(rewritten)

        # Step 3: Generate
        answer = self._generate(user_message, rewritten, documents)

        # Step 4: Update history
        self.chat_history.append(HumanMessage(content=user_message))
        self.chat_history.append(AIMessage(content=answer))

        return {
            "answer": answer,
            "rewritten_query": rewritten,
            "sources": [doc.metadata for doc in documents],
        }

    def reset(self):
        """Clear chat history."""
        self.chat_history = []


# Usage
rag = ConversationalRAG(retriever=retriever)

r1 = rag.chat("How does PostgreSQL handle row-level security?")
print(f"Turn 1: {r1['answer'][:100]}...")

r2 = rag.chat("How do I configure it?")
print(f"Rewritten: {r2['rewritten_query']}")
print(f"Turn 2: {r2['answer'][:100]}...")

r3 = rag.chat("What about performance impact?")
print(f"Rewritten: {r3['rewritten_query']}")
print(f"Turn 3: {r3['answer'][:100]}...")
```

---

## Common Pitfalls

1. **Not rewriting queries before retrieval.** Sending "How do I configure it?" directly to the retriever returns generic configuration docs. Always rewrite using history.
2. **Rewriting every query, even standalone ones.** Some queries are already self-contained. Unnecessary rewriting adds latency and can introduce errors. Detect standalone queries and skip rewriting.
3. **Unlimited history growth.** Without a cap, history eventually exceeds the context window. Implement sliding window or summarization.
4. **Ignoring topic switches.** When the user says "Actually, let's talk about Kafka instead," the system must detect the switch and not carry over PostgreSQL context.
5. **Using the same context window budget for all turns.** Early turns have short history (more room for docs). Later turns have long history (less room for docs). Dynamically adjust the number of retrieved documents based on available budget.
6. **Not testing with realistic conversation flows.** Single-turn tests pass but multi-turn tests fail because of accumulated errors in reference resolution. Test with 5-10 turn conversations.

---

## References

- LangChain conversational RAG: https://python.langchain.com/docs/tutorials/qa_chat_history/
- LlamaIndex chat engines: https://docs.llamaindex.ai/en/stable/module_guides/deploying/chat_engines/
- Anantha et al. "Open-Domain Question Answering Goes Conversational via Question Rewriting" (NAACL 2021)
- Vakulenko et al. "Question Rewriting for Conversational Question Answering" (WSDM 2021)
