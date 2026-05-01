---
title: "Harness Is Governance: Constraining Agents with Code, Not Prompts"
date: 2026-05-01
draft: false
tags: ["AI Agent", "Governance", "Harness", "MCP", "Security"]
categories: ["Architecture Decisions"]
summary: "The mainstream approach to Agent governance is writing rules in prompts. But LLMs can ignore prompts. Real governance lives outside the Agent, in the Harness layer — tools define what's possible, errors define what's forbidden, code defines where the boundaries are."
---

> **The mainstream approach to Agent governance is writing rules in prompts. But LLMs can ignore prompts. Real governance lives outside the Agent, in the Harness layer — tools define what's possible, errors define what's forbidden, code defines where the boundaries are.**

## Why Prompt Governance Fails

When you write "backtests must go through the API" in a system prompt, you're making an assumption: the Agent will obey text instructions.

This assumption holds most of the time. But it fails at the worst moments — when the Agent finds a more efficient path, it tends to take shortcuts. Early instructions get diluted in long contexts, and Agents can "forget" constraints across many conversation turns.

The fundamental problem: **prompt-level constraints have no enforcement mechanism.** Violating a prompt produces no consequences — the Agent keeps running, having done something you didn't want. You might not discover it until you review the output.

This isn't an LLM bug. It's the inherent limitation of text-based constraints.

## The Harness Layer: Interface Between Agent and System

A harness isn't a framework — it's a position. The interface layer between the Agent and the domain system.

```
Agent (Claude / GPT / DeepSeek)
    │
    ├── Harness Layer ←── Governance lives here
    │   ├── MCP tool definitions (what's possible)
    │   ├── Runtime guards (what's forbidden)
    │   ├── Parameter validation (input boundaries)
    │   └── Return value design (output completeness)
    │
    └── Domain System
        ├── Backtest engine
        ├── Expression parser
        ├── Anti-overfit detection
        └── WQ BRAIN simulator
```

The Harness layer determines three things:

1. **What the Agent can do** — through which tools are exposed
2. **What the Agent cannot do** — through runtime guards and parameter validation
3. **What the Agent sees** — through return value design

Together, these three things constitute governance. Not a set of rule documents — a layer of code.

## Mechanism 1: Capability Boundaries

An Agent's capabilities are defined by the tools it can call. Tools you don't expose, the Agent simply cannot use.

QuantGPT exposes 8 MCP tools. The Agent can't directly access the database, can't directly download market data, can't directly modify the cache. It can only get backtest results through `run_backtest` — what the backtest engine does internally (data fetching, cache management, concurrency control) is none of the Agent's business.

This is the same idea as operating system syscalls. Userspace programs can't directly read/write disk — they must go through kernel syscalls. Agents can't directly operate on domain systems — they must go through the Harness's tools.

**Not exposed = doesn't exist.** This is infinitely more reliable than "please don't call this function."

## Mechanism 2: Runtime Enforcement

Exposed tools also need internal constraints. QuantGPT's three layers of runtime defense:

**API boundary guard:**
```python
_api_context = threading.local()

def _require_api_context():
    if not getattr(_api_context, 'active', False):
        raise RuntimeError("Must be called through API")
```

The first line of the backtest function is this guard. If the Agent tries to `import` and call directly (bypassing MCP tools), it crashes immediately.

**Expression safety limits:**
```python
MAX_DEPTH = 100
MAX_WINDOW = 500
MAX_EXPRESSION_LENGTH = 1000
```

Agent generates an expression exceeding limits? The parser refuses to execute. The Agent is forced to generate a simpler expression.

**Dual-mode compilation:**
```python
if mode == "wq" and operator not in WQ_COMPATIBLE_OPS:
    raise RuntimeError(f"'{operator}' not available in WQ BRAIN mode")
```

Agent wants to submit an expression with local-only operators to WQ BRAIN? Caught at compile time, not at platform submission.

Every layer is **code-level enforcement** — not suggestions, not documentation, not prompt instructions. Violations throw errors, and the Agent must adjust.

## Mechanism 3: Information Boundaries

Agent decision quality depends on the information it sees. The Harness controls this through **return value design**.

QuantGPT's `score_factor` returns 6-dimensional scoring: IC mean, IC_IR, stability, anti-overfit, group backtest, WQ alignment. The Agent doesn't see a vague "good/bad" — it sees **diagnostic information** that reveals which dimension is dragging the score down, enabling targeted improvements.

Similarly, `diagnose_factor` returns not just "score is low" but specific failure modes and improvement suggestions. The Agent doesn't need to infer "why is Sharpe low" — the tool tells it directly.

**Information richness and structure determine the Agent's decision ceiling.** Give an Agent a number, it can only compare. Give it a diagnostic table, it can reason.

## Mechanism 4: Cross-Review

Single-Agent research has a fundamental problem: the model generating hypotheses and the model evaluating them is the same one. Confirmation bias is structural.

QuantGPT solves this at the Harness layer: every research conclusion must go through independent review by a second LLM (DeepSeek).

```
Agent (Claude) makes judgment
    │
    ├── Collects factual data (backtest metrics)
    ├── Writes judgment + reasoning chain
    │
    └── Harness layer: invokes DeepSeek review
        ├── Agrees → output conclusion
        └── Disagrees → present both positions, take conservative option
```

This isn't the Agent choosing whether to seek a second opinion — the Harness **mandates** it. The Agent can't skip this step because the research conclusion output interface requires cross-review results.

## Why Harness = Governance

Summarizing the four mechanisms:

| Governance Dimension | Mechanism | Implementation Layer |
|:---------------------|:----------|:--------------------|
| What's possible | Tool exposure | MCP tool definitions |
| What's forbidden | Runtime guards | threading.local + parser limits |
| What's visible | Return value design | Tool output information structure |
| Decision reliability | Cross-review | Mandatory dual-LLM review |

All governance mechanisms live in the Harness layer — outside the Agent, outside the domain system, at the interface.

This position is crucial. If governance is inside the Agent (prompt), it's fragile. If governance is inside the domain system (business logic), it's coupled to the business. The Harness layer is the only position that's **independent of both the Agent and the domain system**.

**Put governance in the Harness, and you can swap Agents, modify domain systems, and governance rules remain unaffected.**

## One-Line Summary

> Agent governance isn't about writing better prompts. It's about designing better Harnesses — using code to define capability boundaries, runtime constraints, information structure, and review processes. Agents only respect rules that throw errors, so write your rules as code.

---

*Series: [Why Agent-Infra, Not Agents](/posts/why-agent-infra-not-agent/) · [Agent-Native Architecture](/posts/agent-native-architecture/) · [AI as Operator, Kernel as Law](/posts/ai-as-operator/)*
