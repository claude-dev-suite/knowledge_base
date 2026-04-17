# Personalization RAG -- User-Specific Retrieval and Generation

## TL;DR

Personalization RAG adapts retrieval-augmented generation to individual users by incorporating their preferences, interaction history, domain expertise level, and past queries into the retrieval and generation process. Instead of returning the same answer to every user who asks "how do I deploy my app," a personalized RAG system knows that User A is a DevOps engineer who prefers Terraform-based deployments while User B is a junior developer who needs step-by-step Docker instructions. This guide covers why personalization matters for production RAG systems, the three layers of personalization (retrieval filtering, context augmentation, and generation adaptation), memory system architectures for storing user state, and the privacy and compliance considerations that determine what you can and cannot store.

---

## Why Personalize RAG?

### The One-Size-Fits-All Problem

Standard RAG pipelines treat every query identically:

```
User A (senior backend engineer):
  "How do I handle database migrations?"
  --> Returns: beginner-friendly guide with basic ALTER TABLE examples

User B (junior frontend developer):
  "How do I handle database migrations?"
  --> Returns: the same beginner-friendly guide

Neither user gets the best answer for their context.
```

Personalized RAG adjusts both what is retrieved and how it is presented:

```
User A (senior backend engineer, uses PostgreSQL, prefers CLI tools):
  "How do I handle database migrations?"
  --> Retrieves: advanced migration patterns, zero-downtime strategies
  --> Generates: concise answer referencing pgloader, Flyway, custom scripts

User B (junior frontend developer, uses Supabase, prefers GUI):
  "How do I handle database migrations?"
  --> Retrieves: Supabase migration guides, getting-started docs
  --> Generates: step-by-step walkthrough with screenshots references
```

### When Personalization Adds Value

| Use Case | Personalization Signal | Impact |
|---|---|---|
| Internal knowledge base | Role, team, past searches | Surface team-relevant docs first |
| Customer support | Product tier, past tickets, feature usage | Avoid suggesting features user does not have |
| E-commerce search | Purchase history, browsing, preferences | Product recommendations in answers |
| Educational platform | Skill level, completed courses, learning pace | Adapt explanation complexity |
| Developer tools | Tech stack, IDE, OS, experience level | Match code examples to user's environment |
| Healthcare portal | Condition history, medications, care plan | Personalized health guidance (with strict privacy) |

### When NOT to Personalize

- **Small user base (<100 users)**: manual curation or simple role-based filtering is sufficient
- **Uniform query intent**: if all users want the same type of answer, personalization adds complexity without benefit
- **Privacy-sensitive domains without consent**: personalization requires storing user data; if you cannot get consent, do not personalize
- **Early-stage products**: get the base RAG pipeline right before adding personalization; it compounds, so base quality matters more

---

## The Three Layers of Personalization

### Layer 1: Personalized Retrieval

Filter and rerank retrieved documents based on user context.

```python
"""
Personalized retrieval: filter and rerank based on user profile.
"""
from dataclasses import dataclass, field


@dataclass
class UserProfile:
    """User profile for personalized retrieval."""
    user_id: str
    role: str = "general"
    expertise_level: str = "intermediate"  # beginner, intermediate, advanced
    team: str = ""
    tech_stack: list[str] = field(default_factory=list)
    preferred_languages: list[str] = field(default_factory=list)
    past_queries: list[str] = field(default_factory=list)
    entity_preferences: dict = field(default_factory=dict)


class PersonalizedRetriever:
    """
    Retriever that incorporates user profile into search.

    Three strategies, applied in order:
    1. Metadata filtering (hard filter by user's context)
    2. Query expansion (add user-relevant terms)
    3. Preference-aware reranking (boost user-relevant results)
    """

    def __init__(self, vector_store, reranker=None):
        self.vector_store = vector_store
        self.reranker = reranker

    def retrieve(
        self,
        query: str,
        user: UserProfile,
        k: int = 10,
        final_k: int = 5,
    ) -> list[dict]:
        """
        Personalized retrieval pipeline.

        Args:
            query: user's question
            user: user profile with preferences
            k: initial retrieval count (before reranking)
            final_k: final number of documents to return
        """
        # Step 1: Build metadata filters
        filters = self._build_metadata_filters(user)

        # Step 2: Expand query with user context
        expanded_query = self._expand_query(query, user)

        # Step 3: Retrieve with filters
        candidates = self.vector_store.similarity_search(
            query=expanded_query,
            k=k,
            filter=filters,
        )

        # Step 4: Fallback if too few results
        if len(candidates) < final_k:
            # Retry without filters to avoid empty results
            candidates = self.vector_store.similarity_search(
                query=expanded_query,
                k=k,
            )

        # Step 5: Preference-aware reranking
        reranked = self._preference_rerank(candidates, user, query)

        return reranked[:final_k]

    def _build_metadata_filters(self, user: UserProfile) -> dict:
        """
        Build metadata filters based on user profile.

        Only apply filters when the user's context strongly
        constrains the relevant document space.
        """
        filters = {}

        if user.tech_stack:
            # Filter to documents matching user's tech stack
            # Use OR logic: match any of the user's technologies
            filters["technology"] = {"$in": user.tech_stack}

        if user.expertise_level == "beginner":
            # Exclude advanced-only documentation
            filters["difficulty"] = {"$in": ["beginner", "intermediate"]}
        elif user.expertise_level == "advanced":
            # Include all, but prefer advanced content (handled in reranking)
            pass

        if user.team:
            # Boost team-specific documentation
            # (soft filter -- handled in reranking rather than hard filter)
            pass

        return filters if filters else None

    def _expand_query(self, query: str, user: UserProfile) -> str:
        """
        Expand the query with user-relevant context.

        This helps the embedding model retrieve more relevant results
        by adding terms the user implicitly means.
        """
        expansions = []

        # Add tech stack context if not already in query
        for tech in user.tech_stack[:3]:
            if tech.lower() not in query.lower():
                expansions.append(tech)

        if expansions:
            return f"{query} (context: {', '.join(expansions)})"
        return query

    def _preference_rerank(
        self,
        candidates: list[dict],
        user: UserProfile,
        query: str,
    ) -> list[dict]:
        """
        Rerank candidates based on user preferences.

        Boost documents that match the user's tech stack,
        expertise level, and team.
        """
        scored = []

        for doc in candidates:
            base_score = doc.get("score", 0.5)
            bonus = 0.0

            metadata = doc.get("metadata", {})

            # Tech stack match
            doc_tech = metadata.get("technology", "")
            if doc_tech and doc_tech in user.tech_stack:
                bonus += 0.10

            # Expertise level match
            doc_difficulty = metadata.get("difficulty", "intermediate")
            if doc_difficulty == user.expertise_level:
                bonus += 0.05
            elif (
                user.expertise_level == "advanced"
                and doc_difficulty == "intermediate"
            ):
                bonus += 0.02  # Advanced users can read intermediate content

            # Team-specific content
            doc_team = metadata.get("team", "")
            if doc_team and doc_team == user.team:
                bonus += 0.08

            # Freshness preference (advanced users prefer recent docs)
            if user.expertise_level == "advanced" and metadata.get("updated_date"):
                # Bonus for recently updated docs
                bonus += 0.03

            scored.append({
                **doc,
                "personalized_score": base_score + bonus,
                "preference_bonus": bonus,
            })

        scored.sort(key=lambda x: x["personalized_score"], reverse=True)
        return scored
```

### Layer 2: Context Augmentation

Inject user-specific context into the prompt alongside retrieved documents.

```python
"""
Context augmentation: add user memory to the generation context.
"""


class ContextAugmenter:
    """
    Augment retrieved context with user-specific information.

    Injects user preferences, past interactions, and entity memory
    into the generation prompt without modifying the retrieval step.
    """

    def augment(
        self,
        query: str,
        retrieved_contexts: list[str],
        user: UserProfile,
        user_memory: dict | None = None,
    ) -> str:
        """
        Build the augmented context for the generator.

        Includes:
        1. User profile summary
        2. Relevant user memory entries
        3. Retrieved documents
        """
        sections = []

        # User context section
        user_context = self._build_user_context(user)
        if user_context:
            sections.append(f"## User Context\n{user_context}")

        # Relevant memory entries
        if user_memory:
            memory_section = self._build_memory_section(query, user_memory)
            if memory_section:
                sections.append(f"## User History\n{memory_section}")

        # Retrieved documents
        docs_section = "\n\n---\n\n".join(retrieved_contexts)
        sections.append(f"## Retrieved Documentation\n{docs_section}")

        return "\n\n".join(sections)

    def _build_user_context(self, user: UserProfile) -> str:
        """Build a concise user context summary for the prompt."""
        parts = []

        if user.role:
            parts.append(f"Role: {user.role}")
        if user.expertise_level:
            parts.append(f"Experience level: {user.expertise_level}")
        if user.tech_stack:
            parts.append(f"Tech stack: {', '.join(user.tech_stack[:5])}")
        if user.preferred_languages:
            parts.append(f"Preferred languages: {', '.join(user.preferred_languages)}")

        return "\n".join(parts) if parts else ""

    def _build_memory_section(
        self,
        query: str,
        user_memory: dict,
    ) -> str:
        """Extract relevant memory entries for the current query."""
        relevant = []

        # Entity preferences
        if "preferences" in user_memory:
            for pref_key, pref_value in user_memory["preferences"].items():
                # Only include preferences relevant to the query
                if any(
                    term in query.lower()
                    for term in pref_key.lower().split("_")
                ):
                    relevant.append(f"User preference: {pref_key} = {pref_value}")

        # Past interactions on this topic
        if "past_answers" in user_memory:
            for past in user_memory["past_answers"][-3:]:
                if self._is_topically_related(query, past["query"]):
                    relevant.append(
                        f"Previously asked: '{past['query']}' "
                        f"-> answered with: '{past['answer'][:100]}...'"
                    )

        return "\n".join(relevant) if relevant else ""

    def _is_topically_related(self, query_a: str, query_b: str) -> bool:
        """Simple check for topical relatedness."""
        words_a = set(query_a.lower().split())
        words_b = set(query_b.lower().split())
        # Remove stop words
        stop_words = {"the", "a", "is", "in", "to", "how", "do", "i", "my", "what"}
        words_a -= stop_words
        words_b -= stop_words
        if not words_a or not words_b:
            return False
        overlap = len(words_a & words_b) / min(len(words_a), len(words_b))
        return overlap > 0.3
```

### Layer 3: Generation Adaptation

Adjust the system prompt and generation parameters based on the user.

```python
"""
Generation adaptation: adjust prompts and parameters per user.
"""
import anthropic


class PersonalizedGenerator:
    """
    Adapt generation to user preferences and expertise level.
    """

    def __init__(self, model: str = "claude-sonnet-4-20250514"):
        self.client = anthropic.Anthropic()
        self.model = model

    def generate(
        self,
        query: str,
        context: str,
        user: UserProfile,
    ) -> str:
        """Generate a personalized answer."""
        system_prompt = self._build_system_prompt(user)

        response = self.client.messages.create(
            model=self.model,
            max_tokens=1000,
            system=system_prompt,
            messages=[{
                "role": "user",
                "content": (
                    f"{context}\n\n"
                    f"---\n\n"
                    f"Question: {query}\n\n"
                    "Answer the question using the information above. "
                    "Adapt your response to the user's context."
                ),
            }],
        )

        return response.content[0].text

    def _build_system_prompt(self, user: UserProfile) -> str:
        """Build a system prompt adapted to the user."""
        base = (
            "You are a technical assistant. Answer questions accurately "
            "using the provided context. Only include information that "
            "is supported by the context."
        )

        adaptations = []

        # Expertise-level adaptation
        if user.expertise_level == "beginner":
            adaptations.append(
                "The user is a beginner. Explain concepts step by step. "
                "Define technical terms when first used. "
                "Provide complete code examples with comments. "
                "Avoid assuming prior knowledge."
            )
        elif user.expertise_level == "advanced":
            adaptations.append(
                "The user is an advanced practitioner. Be concise. "
                "Skip basic explanations they already know. "
                "Focus on edge cases, performance implications, and "
                "advanced patterns. Use precise technical terminology."
            )
        else:
            adaptations.append(
                "The user has intermediate experience. Provide clear "
                "explanations with practical examples. Include relevant "
                "code snippets."
            )

        # Tech stack adaptation
        if user.tech_stack:
            stack_str = ", ".join(user.tech_stack[:5])
            adaptations.append(
                f"The user's tech stack includes: {stack_str}. "
                "When multiple approaches exist, prefer solutions "
                "that work with their stack."
            )

        # Language adaptation
        if user.preferred_languages:
            lang_str = ", ".join(user.preferred_languages[:3])
            adaptations.append(
                f"The user prefers code examples in: {lang_str}."
            )

        if adaptations:
            return base + "\n\n" + "\n\n".join(adaptations)
        return base
```

---

## Architecture Patterns

### Pattern 1: Profile-Based Personalization

The simplest pattern. Store explicit user profiles and use them for retrieval filtering and prompt adaptation.

```
User registers --> Fill out profile (role, stack, level)
Query arrives --> Load profile --> Filter retrieval --> Adapt prompt
```

**Pros**: simple, explicit, no inference needed
**Cons**: requires upfront user effort, stale profiles, no learning

### Pattern 2: Behavior-Based Personalization

Learn user preferences from their interactions. No explicit profile needed.

```
User asks queries --> Log interactions --> Extract patterns
  --> Build implicit profile (preferred topics, complexity level, etc.)
Query arrives --> Load implicit profile --> Personalize
```

**Pros**: no user effort, adapts over time, captures real preferences
**Cons**: cold start problem, requires interaction history, inference may be wrong

### Pattern 3: Hybrid (Recommended)

Combine explicit profiles with learned preferences:

```python
"""
Hybrid personalization: explicit profile + learned preferences.
"""
from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class HybridUserProfile:
    """Combines explicit settings with learned behavior."""
    # Explicit (user-set)
    user_id: str
    role: str = ""
    tech_stack: list[str] = field(default_factory=list)
    expertise_level: str = "intermediate"

    # Learned (system-inferred)
    inferred_topics: list[str] = field(default_factory=list)
    inferred_complexity: float = 0.5  # 0 = simple, 1 = complex
    interaction_count: int = 0
    last_active: str = ""

    # Confidence in learned signals
    inference_confidence: float = 0.0  # 0-1, increases with interactions

    def effective_expertise(self) -> str:
        """
        Combine explicit expertise with inferred complexity preference.

        If the user set 'beginner' but consistently asks advanced questions,
        surface intermediate content.
        """
        if self.inference_confidence < 0.3:
            return self.expertise_level  # Trust explicit setting

        # Blend explicit and inferred
        explicit_score = {"beginner": 0, "intermediate": 0.5, "advanced": 1.0}
        blended = (
            explicit_score.get(self.expertise_level, 0.5) * 0.4
            + self.inferred_complexity * 0.6
        )

        if blended < 0.3:
            return "beginner"
        elif blended > 0.7:
            return "advanced"
        return "intermediate"
```

---

## Privacy and Compliance

### What You Can and Cannot Store

| Data Type | GDPR Classification | Storage Requirements |
|---|---|---|
| User ID | Pseudonymized identifier | Must be deletable on request |
| Role / team | Non-sensitive personal data | Consent required, purpose-limited |
| Query history | Behavioral data | Consent + retention limit |
| Preferences | Inferred personal data | Right to explanation + deletion |
| Answer feedback | Behavioral data | Consent required |
| Entity memory | May contain PII | Encryption + access controls |

### Privacy-Preserving Personalization

```python
"""
GDPR-compliant personalization patterns.
"""
import hashlib
import json
from datetime import datetime, timedelta
from pathlib import Path


class PrivacyAwareProfileStore:
    """
    Store user profiles with GDPR compliance.

    Features:
    - Data minimization (only store what is needed)
    - Right to deletion (complete profile removal)
    - Retention limits (auto-expire old data)
    - Pseudonymization (hash user IDs)
    - Consent tracking (record what the user agreed to)
    """

    def __init__(
        self,
        storage_dir: str,
        retention_days: int = 90,
    ):
        self.storage_dir = Path(storage_dir)
        self.storage_dir.mkdir(parents=True, exist_ok=True)
        self.retention_days = retention_days

    def _pseudonymize_id(self, user_id: str) -> str:
        """Hash user ID for pseudonymized storage."""
        return hashlib.sha256(
            f"personalization:{user_id}".encode()
        ).hexdigest()[:16]

    def store_profile(
        self,
        user_id: str,
        profile_data: dict,
        consent_scope: list[str],
    ) -> None:
        """
        Store user profile with consent tracking.

        consent_scope: list of data categories the user consented to
        e.g., ["role", "tech_stack", "query_history", "preferences"]
        """
        pseudo_id = self._pseudonymize_id(user_id)

        # Only store consented data categories
        filtered = {}
        for key, value in profile_data.items():
            if key in consent_scope:
                filtered[key] = value

        record = {
            "pseudo_id": pseudo_id,
            "data": filtered,
            "consent_scope": consent_scope,
            "consent_timestamp": datetime.now().isoformat(),
            "last_updated": datetime.now().isoformat(),
            "expires_at": (
                datetime.now() + timedelta(days=self.retention_days)
            ).isoformat(),
        }

        path = self.storage_dir / f"{pseudo_id}.json"
        with open(path, "w") as f:
            json.dump(record, f, indent=2)

    def load_profile(self, user_id: str) -> dict | None:
        """Load profile, respecting retention limits."""
        pseudo_id = self._pseudonymize_id(user_id)
        path = self.storage_dir / f"{pseudo_id}.json"

        if not path.exists():
            return None

        with open(path) as f:
            record = json.load(f)

        # Check expiration
        expires = datetime.fromisoformat(record["expires_at"])
        if datetime.now() > expires:
            path.unlink()  # Auto-delete expired profiles
            return None

        return record["data"]

    def delete_profile(self, user_id: str) -> bool:
        """
        Right to deletion: completely remove user profile.

        This must be called when a user requests data deletion
        under GDPR Article 17.
        """
        pseudo_id = self._pseudonymize_id(user_id)
        path = self.storage_dir / f"{pseudo_id}.json"

        if path.exists():
            path.unlink()
            return True
        return False

    def export_profile(self, user_id: str) -> dict | None:
        """
        Right to data portability: export user data.

        GDPR Article 20 requires providing data in a machine-readable format.
        """
        return self.load_profile(user_id)

    def cleanup_expired(self) -> int:
        """Remove all expired profiles. Run periodically."""
        removed = 0
        for path in self.storage_dir.glob("*.json"):
            with open(path) as f:
                record = json.load(f)
            expires = datetime.fromisoformat(record["expires_at"])
            if datetime.now() > expires:
                path.unlink()
                removed += 1
        return removed
```

---

## Evaluation Considerations

### How to Evaluate Personalized RAG

Standard RAG evaluation (RAGAS, DeepEval) does not account for personalization. A "correct" answer depends on who is asking.

```python
"""
Evaluation framework for personalized RAG.
"""


class PersonalizedRAGEvaluator:
    """
    Evaluate personalized RAG by scoring answer appropriateness
    for specific user profiles.
    """

    def evaluate(
        self,
        query: str,
        answer: str,
        user: "UserProfile",
        golden_answers: dict[str, str],
    ) -> dict:
        """
        Evaluate a personalized answer.

        Args:
            query: the user's question
            answer: the personalized answer
            user: the user profile used for personalization
            golden_answers: dict mapping expertise level to expected answer
                e.g., {"beginner": "...", "advanced": "..."}
        """
        # Match against the user-appropriate golden answer
        expected = golden_answers.get(user.expertise_level, "")

        # Standard correctness (against user-appropriate answer)
        correctness = self._score_correctness(answer, expected)

        # Complexity appropriateness
        complexity = self._score_complexity_match(answer, user)

        # Tech stack relevance
        stack_relevance = self._score_stack_relevance(answer, user)

        return {
            "correctness": correctness,
            "complexity_match": complexity,
            "stack_relevance": stack_relevance,
            "overall": (correctness + complexity + stack_relevance) / 3,
        }

    def _score_correctness(self, answer: str, expected: str) -> float:
        """Score factual correctness against golden answer."""
        if not expected:
            return 0.5  # No golden answer available
        # In practice, use an LLM judge or RAGAS faithfulness
        common_words = set(answer.lower().split()) & set(expected.lower().split())
        return min(len(common_words) / max(len(expected.split()), 1), 1.0)

    def _score_complexity_match(
        self, answer: str, user: "UserProfile",
    ) -> float:
        """Score whether answer complexity matches user level."""
        avg_word_length = (
            sum(len(w) for w in answer.split()) / max(len(answer.split()), 1)
        )
        sentence_count = answer.count(".") + answer.count("!")

        # Heuristic: longer words and more sentences = more complex
        complexity = min(avg_word_length / 8, 1.0)

        target = {"beginner": 0.3, "intermediate": 0.5, "advanced": 0.8}
        target_complexity = target.get(user.expertise_level, 0.5)

        # Score based on distance from target
        distance = abs(complexity - target_complexity)
        return max(0, 1 - distance * 2)

    def _score_stack_relevance(
        self, answer: str, user: "UserProfile",
    ) -> float:
        """Score whether answer references user's tech stack."""
        if not user.tech_stack:
            return 1.0  # No stack to match

        answer_lower = answer.lower()
        matches = sum(
            1 for tech in user.tech_stack if tech.lower() in answer_lower
        )

        return min(matches / max(len(user.tech_stack), 1), 1.0)
```

---

## Common Pitfalls

1. **Over-personalizing**: adding every piece of user context to the prompt confuses the LLM and increases latency. Include only the 2-3 most relevant personalization signals per query.

2. **Cold start with empty profiles**: new users have no history or preferences. Fall back to role-based defaults or ask 2-3 onboarding questions rather than providing generic answers.

3. **Stale profiles**: a user who was a "beginner" six months ago may now be "advanced." Include decay or re-inference mechanisms for learned preferences.

4. **Filter too aggressively**: hard metadata filters (only show Python docs to Python users) can exclude relevant results. Prefer soft reranking boosts over hard filters.

5. **Ignoring privacy from the start**: retrofitting GDPR compliance is painful. Design for consent, deletion, and data minimization from day one.

6. **Not measuring personalization impact**: always A/B test personalized vs non-personalized responses to verify that personalization actually improves user satisfaction. Personalization that hurts is worse than no personalization.

7. **Leaking user data across sessions**: ensure per-user memory is properly isolated. A memory system that accidentally surfaces User A's preferences for User B is both a privacy violation and a quality failure.

---

## References

- Mem0: https://docs.mem0.ai/ -- memory layer for personalized AI
- Zep: https://docs.getzep.com/ -- long-term memory for AI assistants
- Letta (MemGPT): https://docs.letta.com/ -- self-editing memory for LLM agents
- Salemi, A. et al. "LaMP: When Large Language Models Meet Personalization." arXiv 2023
- GDPR Article 17 (Right to Erasure): https://gdpr-info.eu/art-17-gdpr/
- GDPR Article 20 (Data Portability): https://gdpr-info.eu/art-20-gdpr/
