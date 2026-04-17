# DSPy -- Optimization: BootstrapFewShot, MIPROv2, and BootstrapFinetune

## Overview

Optimization is DSPy's core value proposition. Instead of manually engineering prompts, you define a metric, provide training data, and let an optimizer compile your pipeline into a high-performing system. DSPy offers three primary optimizer families, each targeting a different optimization strategy:

| Optimizer | What it Optimizes | Cost | Best For |
|-----------|-------------------|------|----------|
| `BootstrapFewShot` | Few-shot examples in the prompt | Low (few LM calls) | Quick gains, small datasets |
| `MIPROv2` | Instructions + few-shot examples jointly | Medium (many LM calls) | Maximum prompt-based performance |
| `BootstrapFinetune` | Model weights via finetuning | High (finetuning job) | Production deployment, lower inference cost |

This guide covers each optimizer in depth with complete, runnable examples.

---

## Prerequisites for All Optimizers

Every optimizer needs three things:

```python
import dspy
from dspy.evaluate import Evaluate

# 1. A configured language model
lm = dspy.LM("openai/gpt-4o-mini", temperature=0.0)
dspy.configure(lm=lm)

# 2. A module (the pipeline to optimize)
class RAGModule(dspy.Module):
    def __init__(self):
        self.retrieve = dspy.Retrieve(k=5)
        self.answer = dspy.ChainOfThought("context, question -> answer")

    def forward(self, question: str) -> dspy.Prediction:
        context = self.retrieve(question).passages
        result = self.answer(context=context, question=question)
        return dspy.Prediction(answer=result.answer, context=context)

# 3. Training data with labeled examples
trainset = [
    dspy.Example(
        question="What is retrieval-augmented generation?",
        answer="RAG is a technique that combines information retrieval with "
               "language model generation to produce grounded answers.",
    ).with_inputs("question"),
    dspy.Example(
        question="What embedding model should I use for RAG?",
        answer="For English text, BGE-large or text-embedding-3-small offer "
               "a good balance of quality and speed.",
    ).with_inputs("question"),
    # ... 20-200 examples recommended
]

devset = trainset[:10]
trainset = trainset[10:]

# And a metric function
def answer_quality(example, prediction, trace=None):
    """LM-judge metric for answer quality."""
    judge = dspy.Predict(
        "question, reference_answer, predicted_answer -> score: float"
    )
    result = judge(
        question=example.question,
        reference_answer=example.answer,
        predicted_answer=prediction.answer,
    )
    score = min(max(float(result.score), 0.0), 1.0)
    if trace is not None:
        return score >= 0.7
    return score
```

---

## BootstrapFewShot

### How It Works

BootstrapFewShot is the simplest optimizer. It:

1. Runs your pipeline on training examples
2. Filters to examples where the metric passes (good demonstrations)
3. Inserts those demonstrations as few-shot examples into the prompt
4. Selects the subset of demonstrations that maximizes dev set performance

```
Training data -> Run pipeline -> Filter by metric -> Select best demos -> Compiled pipeline
```

### Basic Usage

```python
from dspy.teleprompt import BootstrapFewShot

optimizer = BootstrapFewShot(
    metric=answer_quality,
    max_bootstrapped_demos=4,   # max generated few-shot examples
    max_labeled_demos=4,        # max labeled examples from training set
    max_rounds=1,               # number of bootstrap rounds
    max_errors=5,               # max errors before stopping
)

compiled_rag = optimizer.compile(
    student=RAGModule(),
    trainset=trainset,
)

# Evaluate improvement
evaluator = Evaluate(devset=devset, metric=answer_quality, num_threads=4)
baseline = evaluator(RAGModule())
optimized = evaluator(compiled_rag)
print(f"Baseline: {baseline:.1f}% -> Optimized: {optimized:.1f}%")
```

### How Demonstrations Are Selected

The optimizer generates candidate demonstrations by running the pipeline:

```python
# Internally, BootstrapFewShot does something like this:
demos = []
for example in trainset:
    # Run the pipeline
    prediction = pipeline(question=example.question)

    # Check if the output passes the metric
    if metric(example, prediction, trace=trace):
        # This is a good demonstration -- save the full trace
        demos.append({
            "question": example.question,
            "answer": prediction.answer,
            "reasoning": prediction.reasoning,  # if ChainOfThought
        })

# Then it selects the best subset of demos for the prompt
```

### Teacher-Student Configuration

You can use a more powerful model as the "teacher" to generate demonstrations for a cheaper "student" model:

```python
from dspy.teleprompt import BootstrapFewShot

# Teacher: expensive but high-quality
teacher_lm = dspy.LM("openai/gpt-4o", temperature=0.0)

# Student: cheaper for production
student_lm = dspy.LM("openai/gpt-4o-mini", temperature=0.0)

# Configure with teacher
dspy.configure(lm=teacher_lm)

optimizer = BootstrapFewShot(
    metric=answer_quality,
    max_bootstrapped_demos=8,
    max_labeled_demos=4,
)

# Compile: teacher generates demos, student uses them
dspy.configure(lm=student_lm)
compiled = optimizer.compile(
    student=RAGModule(),
    teacher=RAGModule(),  # uses teacher_lm for generating demos
    trainset=trainset,
)
```

### Tuning BootstrapFewShot Parameters

```python
# Conservative: fewer demos, less prompt bloat
optimizer = BootstrapFewShot(
    metric=answer_quality,
    max_bootstrapped_demos=2,
    max_labeled_demos=2,
    max_rounds=1,
)

# Aggressive: more demos, higher chance of finding good ones
optimizer = BootstrapFewShot(
    metric=answer_quality,
    max_bootstrapped_demos=8,
    max_labeled_demos=8,
    max_rounds=2,  # two rounds of bootstrapping
    max_errors=20,
)
```

**Guidelines for parameter selection:**

| Parameter | Small dataset (<50) | Medium dataset (50-500) | Large dataset (>500) |
|-----------|--------------------|-----------------------|---------------------|
| `max_bootstrapped_demos` | 2-3 | 4-6 | 4-8 |
| `max_labeled_demos` | 2-3 | 4-6 | 4-8 |
| `max_rounds` | 1 | 1-2 | 2-3 |

---

## BootstrapFewShotWithRandomSearch

An extension that tries multiple random subsets of demonstrations and picks the best:

```python
from dspy.teleprompt import BootstrapFewShotWithRandomSearch

optimizer = BootstrapFewShotWithRandomSearch(
    metric=answer_quality,
    max_bootstrapped_demos=4,
    max_labeled_demos=4,
    num_candidate_programs=8,   # number of random subsets to try
    num_threads=4,              # parallel evaluation threads
    max_rounds=1,
)

compiled = optimizer.compile(
    student=RAGModule(),
    trainset=trainset,
    valset=devset,  # validation set for selecting the best program
)
```

This is strictly better than plain `BootstrapFewShot` at the cost of `num_candidate_programs` times more LM evaluations. Use it when you have the budget for more thorough search.

---

## MIPROv2

### How It Works

MIPROv2 (Multi-prompt Instruction Proposal Optimizer v2) is DSPy's most powerful prompt optimizer. Unlike BootstrapFewShot which only selects demonstrations, MIPROv2 jointly optimizes:

1. **Instructions**: The task description / system prompt for each module
2. **Few-shot demonstrations**: Which examples to include in the prompt
3. **Demonstration ordering**: How to arrange examples for best performance

MIPROv2 uses a Bayesian optimization approach (based on TPE -- Tree-structured Parzen Estimator) to efficiently search the joint space of instructions and demonstrations.

```
Generate instruction candidates -> Generate demo candidates -> Bayesian search over combinations -> Best compiled pipeline
```

### Basic Usage

```python
from dspy.teleprompt import MIPROv2

optimizer = MIPROv2(
    metric=answer_quality,
    num_candidates=7,       # instruction candidates to generate per module
    init_temperature=1.0,   # creativity for instruction generation
    verbose=True,           # show optimization progress
)

compiled = optimizer.compile(
    student=RAGModule(),
    trainset=trainset,
    num_batches=40,         # total Bayesian optimization trials
    max_bootstrapped_demos=4,
    max_labeled_demos=4,
    eval_kwargs=dict(
        num_threads=4,
        display_progress=True,
    ),
)
```

### Understanding MIPROv2 Stages

MIPROv2 runs in three stages:

```python
# Stage 1: Generate instruction candidates
# MIPROv2 uses the LM to propose different instructions for each Signature
# in your pipeline. It generates num_candidates instructions per module.

# For a RAG pipeline with two modules (retrieve + answer), it might generate:
# Module 1 instructions:
#   "Given context passages, answer the user's question accurately."
#   "You are a precise question-answering system. Use only the provided context."
#   "Answer the question based strictly on the retrieved passages. Be concise."
#   ... (7 candidates)

# Stage 2: Generate demonstration candidates
# Similar to BootstrapFewShot, but generates more candidates for the search

# Stage 3: Bayesian optimization
# Tries different combinations of (instruction, demos) across modules
# Uses TPE to focus on promising regions of the search space
```

### Advanced MIPROv2 Configuration

```python
from dspy.teleprompt import MIPROv2

optimizer = MIPROv2(
    metric=answer_quality,
    num_candidates=10,          # more candidates = better instructions, more cost
    init_temperature=1.4,       # higher = more diverse instructions
    verbose=True,
    track_stats=True,           # track optimization statistics
)

compiled = optimizer.compile(
    student=RAGModule(),
    trainset=trainset,
    num_batches=80,             # more batches = more thorough search
    max_bootstrapped_demos=4,
    max_labeled_demos=4,
    requires_permission_to_run=False,  # skip confirmation prompt
    eval_kwargs=dict(
        num_threads=8,
        display_progress=True,
    ),
)

# After optimization, inspect the best configuration
if optimizer.track_stats:
    print(f"Best score: {compiled._compiled_score}")
```

### Cost Estimation for MIPROv2

MIPROv2 makes many LM calls. Estimate cost before running:

```python
def estimate_mipro_cost(
    num_modules: int,
    num_candidates: int,
    num_batches: int,
    train_size: int,
    avg_tokens_per_call: int = 500,
    cost_per_1k_tokens: float = 0.00015,  # gpt-4o-mini input
):
    """Rough cost estimate for MIPROv2 optimization."""
    # Stage 1: instruction generation
    instruction_calls = num_modules * num_candidates
    # Stage 2: demo generation
    demo_calls = train_size * 2  # forward + evaluation
    # Stage 3: Bayesian search (each batch evaluates on dev set)
    search_calls = num_batches * train_size

    total_calls = instruction_calls + demo_calls + search_calls
    total_tokens = total_calls * avg_tokens_per_call
    estimated_cost = (total_tokens / 1000) * cost_per_1k_tokens

    print(f"Estimated LM calls: {total_calls:,}")
    print(f"Estimated tokens: {total_tokens:,}")
    print(f"Estimated cost: ${estimated_cost:.2f}")
    return estimated_cost

# Example: 2 modules, 7 candidates, 40 batches, 100 training examples
estimate_mipro_cost(
    num_modules=2,
    num_candidates=7,
    num_batches=40,
    train_size=100,
)
# Estimated LM calls: 4,214
# Estimated tokens: 2,107,000
# Estimated cost: $0.32 (with gpt-4o-mini)
```

### MIPROv2 vs. BootstrapFewShot

| Aspect | BootstrapFewShot | MIPROv2 |
|--------|-----------------|---------|
| Optimizes instructions | No | Yes |
| Optimizes demonstrations | Yes (random or greedy) | Yes (Bayesian) |
| LM calls | Low (1-2x training set) | High (10-100x training set) |
| Typical improvement | 5-15% | 10-30% |
| Time to run | Minutes | Minutes to hours |
| Min training examples | 10 | 20-50 |
| When to use | Quick iteration, small budgets | Final optimization before production |

---

## BootstrapFinetune

### How It Works

BootstrapFinetune takes optimization beyond prompting. It:

1. Generates high-quality demonstrations using your pipeline (like BootstrapFewShot)
2. Formats those demonstrations as finetuning data
3. Submits a finetuning job to the LM provider (OpenAI, etc.)
4. Returns a compiled pipeline that uses the finetuned model

This lets you "bake in" the learned behavior so you no longer need few-shot examples in the prompt, reducing inference cost and latency.

### Basic Usage

```python
from dspy.teleprompt import BootstrapFinetune

optimizer = BootstrapFinetune(
    metric=answer_quality,
    num_threads=4,
)

# This will:
# 1. Run the pipeline on trainset to generate demonstrations
# 2. Filter by metric
# 3. Format as finetuning data (OpenAI JSONL format)
# 4. Submit finetuning job
# 5. Wait for completion
# 6. Return compiled pipeline using finetuned model
compiled = optimizer.compile(
    student=RAGModule(),
    trainset=trainset,
    target="openai/gpt-4o-mini",   # model to finetune
    batchsize=50,                   # examples per finetuning batch
    epochs=2,                       # finetuning epochs
)

# The compiled pipeline now uses the finetuned model
result = compiled(question="What is RAG?")
```

### Teacher-Student Finetuning

The most powerful pattern: use a large model to generate demonstrations, then finetune a small model on them:

```python
from dspy.teleprompt import BootstrapFinetune

# Configure the teacher (expensive, high-quality)
teacher_lm = dspy.LM("openai/gpt-4o", temperature=0.7)
dspy.configure(lm=teacher_lm)

optimizer = BootstrapFinetune(
    metric=answer_quality,
    num_threads=4,
)

# Student model: smaller, cheaper
# Teacher generates the demonstrations, student learns from them
compiled = optimizer.compile(
    student=RAGModule(),
    teacher=RAGModule(),             # uses teacher_lm
    trainset=trainset,
    target="openai/gpt-4o-mini",     # finetune the cheap model
    epochs=3,
)

# Now gpt-4o-mini performs like gpt-4o on your task
# at a fraction of the cost
```

### Two-Stage Optimization: MIPROv2 + Finetune

For maximum performance, combine prompt optimization with finetuning:

```python
import dspy
from dspy.teleprompt import MIPROv2, BootstrapFinetune

# Stage 1: Optimize prompts with MIPROv2
lm = dspy.LM("openai/gpt-4o", temperature=0.0)
dspy.configure(lm=lm)

mipro = MIPROv2(
    metric=answer_quality,
    num_candidates=10,
    verbose=True,
)

prompt_optimized = mipro.compile(
    student=RAGModule(),
    trainset=trainset,
    num_batches=60,
    max_bootstrapped_demos=6,
    max_labeled_demos=4,
)

# Evaluate prompt-optimized version
evaluator = Evaluate(devset=devset, metric=answer_quality, num_threads=4)
mipro_score = evaluator(prompt_optimized)
print(f"MIPROv2 score: {mipro_score:.1f}%")

# Stage 2: Finetune using the optimized pipeline as teacher
finetune = BootstrapFinetune(
    metric=answer_quality,
    num_threads=4,
)

final = finetune.compile(
    student=RAGModule(),
    teacher=prompt_optimized,        # MIPROv2-optimized pipeline as teacher
    trainset=trainset,
    target="openai/gpt-4o-mini",
    epochs=3,
)

final_score = evaluator(final)
print(f"Finetuned score: {final_score:.1f}%")
```

---

## Writing Good Metrics

The metric is the most important part of DSPy optimization. Bad metrics lead to bad optimization.

### Exact Match (Simplest)

```python
def exact_match(example, prediction, trace=None):
    return example.answer.lower().strip() == prediction.answer.lower().strip()
```

### F1 Score (Token Overlap)

```python
def token_f1(example, prediction, trace=None):
    gold_tokens = set(example.answer.lower().split())
    pred_tokens = set(prediction.answer.lower().split())

    if not pred_tokens:
        return 0.0

    precision = len(gold_tokens & pred_tokens) / len(pred_tokens)
    recall = len(gold_tokens & pred_tokens) / len(gold_tokens)

    if precision + recall == 0:
        return 0.0

    f1 = 2 * precision * recall / (precision + recall)

    if trace is not None:
        return f1 >= 0.5  # binary for optimization
    return f1
```

### LM-as-Judge (Most Flexible)

```python
class JudgeSignature(dspy.Signature):
    """Evaluate whether the predicted answer correctly addresses the question
    given the reference answer. Score from 0 to 5."""

    question: str = dspy.InputField()
    reference_answer: str = dspy.InputField()
    predicted_answer: str = dspy.InputField()
    reasoning: str = dspy.OutputField(desc="why you gave this score")
    score: int = dspy.OutputField(desc="integer score from 0 to 5")


def llm_judge_metric(example, prediction, trace=None):
    judge = dspy.ChainOfThought(JudgeSignature)
    result = judge(
        question=example.question,
        reference_answer=example.answer,
        predicted_answer=prediction.answer,
    )
    score = min(max(int(result.score), 0), 5) / 5.0

    if trace is not None:
        return score >= 0.6
    return score
```

### Composite Metrics

```python
def composite_rag_metric(example, prediction, trace=None):
    """Combine multiple quality signals."""
    scores = {}

    # 1. Answer relevance (does it answer the question?)
    scores["relevance"] = token_f1(example, prediction)

    # 2. Faithfulness (is it grounded in context?)
    if hasattr(prediction, "context") and prediction.context:
        context_text = " ".join(prediction.context).lower()
        answer_tokens = prediction.answer.lower().split()
        grounded_tokens = sum(1 for t in answer_tokens if t in context_text)
        scores["faithfulness"] = grounded_tokens / max(len(answer_tokens), 1)
    else:
        scores["faithfulness"] = 0.0

    # 3. Completeness (is the answer detailed enough?)
    scores["completeness"] = min(len(prediction.answer.split()) / 30.0, 1.0)

    # Weighted average
    final = (
        0.5 * scores["relevance"]
        + 0.3 * scores["faithfulness"]
        + 0.2 * scores["completeness"]
    )

    if trace is not None:
        return final >= 0.5
    return final
```

---

## Optimization Best Practices

### Data Quality Over Quantity

```python
# BAD: vague, inconsistent examples
bad_example = dspy.Example(
    question="Tell me about RAG",
    answer="RAG is cool",  # too short, too vague
).with_inputs("question")

# GOOD: specific, detailed examples
good_example = dspy.Example(
    question="What are the main components of a RAG pipeline?",
    answer="A RAG pipeline has three main components: (1) a retriever that finds "
           "relevant passages from a knowledge base, (2) a context formatter that "
           "prepares retrieved passages for the language model, and (3) a generator "
           "that produces the final answer conditioned on the query and context.",
).with_inputs("question")
```

### Iterative Optimization Workflow

```python
import dspy
from dspy.evaluate import Evaluate
from dspy.teleprompt import BootstrapFewShot, MIPROv2

# Step 1: Establish baseline
rag = RAGModule()
evaluator = Evaluate(devset=devset, metric=answer_quality, num_threads=4)
baseline = evaluator(rag)
print(f"Step 1 - Baseline: {baseline:.1f}%")

# Step 2: Quick optimization with BootstrapFewShot
bs = BootstrapFewShot(metric=answer_quality, max_bootstrapped_demos=4)
bs_compiled = bs.compile(student=RAGModule(), trainset=trainset)
bs_score = evaluator(bs_compiled)
print(f"Step 2 - BootstrapFewShot: {bs_score:.1f}%")

# Step 3: If not good enough, try MIPROv2
if bs_score < 85.0:
    mipro = MIPROv2(metric=answer_quality, num_candidates=7)
    mipro_compiled = mipro.compile(
        student=RAGModule(),
        trainset=trainset,
        num_batches=40,
        max_bootstrapped_demos=4,
        max_labeled_demos=4,
    )
    mipro_score = evaluator(mipro_compiled)
    print(f"Step 3 - MIPROv2: {mipro_score:.1f}%")

# Step 4: If targeting production with cost constraints, finetune
# (see BootstrapFinetune section above)

# Step 5: Save the best pipeline
best = mipro_compiled  # or bs_compiled, depending on results
best.save("./compiled/best_rag.json")
```

### Avoiding Common Optimization Failures

1. **Metric too easy**: If baseline already scores 90%+, optimizers have little room. Make the metric harder (e.g., require citations, penalize verbosity).

2. **Metric too noisy**: LM-as-judge metrics with high temperature produce inconsistent scores. Use `temperature=0` for the judge LM.

3. **Too few training examples**: BootstrapFewShot needs at least 10-20 examples. MIPROv2 needs 20-50. BootstrapFinetune needs 50-200.

4. **Training data too homogeneous**: If all questions are about the same topic, the optimizer learns to parrot that topic. Include diverse question types.

5. **Overfitting to dev set**: If dev score is high but test score is low, you are overfitting. Increase dev set size or reduce `num_batches` in MIPROv2.

---

## Saving, Loading, and Deploying

```python
# Save compiled pipeline (JSON format)
compiled_rag.save("./compiled/rag_mipro_v1.json")

# Load into a fresh module
fresh_rag = RAGModule()
fresh_rag.load("./compiled/rag_mipro_v1.json")

# Use in production
result = fresh_rag(question="What is DSPy?")

# Serve via FastAPI
from fastapi import FastAPI

app = FastAPI()

rag = RAGModule()
rag.load("./compiled/rag_mipro_v1.json")

@app.post("/query")
async def query(question: str):
    result = rag(question=question)
    return {
        "answer": result.answer,
        "sources": result.context,
    }
```

---

## Common Pitfalls

1. **Running MIPROv2 without enough budget**: MIPROv2 with `num_batches=40` and 100 training examples makes thousands of LM calls. Estimate cost first (see cost estimation section above).

2. **Not comparing against baseline**: Always measure the unoptimized pipeline first. If your metric is bad, optimization may actually decrease quality.

3. **Using trace incorrectly in metrics**: When `trace is not None` (during optimization), return `bool`. When `trace is None` (during evaluation), return `float`. Getting this wrong makes optimizers fail silently.

4. **Finetuning on too little data**: BootstrapFinetune needs at least 50 high-quality demonstrations. With fewer, the finetuned model may be worse than the base model with good prompts.

5. **Not testing on held-out data**: Optimization can overfit. Always evaluate on a test set that was never seen during optimization.

---

## References

- DSPy Optimizers documentation: https://dspy.ai/learn/optimization/overview/
- MIPROv2 paper: "Optimizing Instructions and Demonstrations for Multi-Stage Language Model Programs"
- BootstrapFinetune: https://dspy.ai/learn/optimization/finetuning/
- DSPy GitHub examples: https://github.com/stanfordnlp/dspy/tree/main/examples
