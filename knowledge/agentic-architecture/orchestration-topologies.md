# Agentic System Orchestration Topologies

## Overview

Agentic systems use LLMs to take actions through tools, sometimes deciding their
own next step. The architecture is about choosing the simplest control structure
that accomplishes the task and bounding the agent's autonomy, cost, and blast
radius.

## First decision: workflow vs autonomous agent

- **Workflow** — fixed, code-orchestrated steps with LLM calls at each. It is
  predictable, cheap, and debuggable. **Prefer it whenever the steps are known.**
- **Autonomous agent** — the LLM decides the next action in a loop. It is
  flexible and handles open-ended tasks but is less predictable and costlier.
  Use it only when the path genuinely cannot be predefined.

## Orchestration topologies

| Topology | Shape | Fits |
|----------|-------|------|
| Single agent + tools | One loop with a toolbox | Most tasks — start here |
| Supervisor / sub-agents | An orchestrator delegates to specialists with their own context | Decomposable tasks; context isolation |
| Pipeline / chain | Staged hand-offs | Known multi-stage transforms |
| Network / peer agents | Agents message one another | Rarely needed; high complexity and cost |

Bias toward the simplest topology that works, and isolate context with
sub-agents when a subtask would otherwise flood the main context.

## Cross-cutting concerns

- **Memory and context:** short-term (conversation), long-term (a vector store),
  and scratch; summarize/compact to fit the window; decide what persists across
  runs.
- **Tool layer:** typed tools with clear contracts, least privilege, and
  validated inputs and outputs — tools are the agent's blast radius, so scope
  them.
- **Control and safety:** human-in-the-loop approval for irreversible or
  outward-facing actions, step/turn budgets, and explicit termination conditions.
- **Determinism and cost:** cap iterations, cache, prefer workflows for the
  deterministic parts, and trace every step (tool calls, tokens, cost).
- **Failure handling:** retries, fallbacks, and a defined give-up/escalate path
  so an agent never loops forever.

## Guidance

Known steps → a workflow. Open-ended but decomposable → a supervisor with
sub-agents. One coherent task → a single agent with tools. Reach for multi-agent
networks only when the simpler shapes demonstrably fail.
