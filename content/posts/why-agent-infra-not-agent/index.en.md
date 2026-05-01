---
title: "Why I Build Agent Infrastructure, Not Agents"
date: 2026-05-01
draft: false
tags: ["AI Agent", "Architecture", "MCP", "Strategy"]
categories: ["Architecture Decisions"]
summary: "While everyone is building Agent frameworks, I chose a different path: build infrastructure for Agents, not the Agent itself. Not because I can't build Agents, but because the Agent layer is a consumable — infrastructure is an asset."
---

> **While everyone is building Agent frameworks, I chose a different path: build infrastructure for Agents, not the Agent itself. Not because I can't build Agents, but because the Agent layer is a consumable — infrastructure is an asset.**

## A Counterintuitive Choice

LangChain, CrewAI, AutoGen, Dify, Coze — a new Agent framework ships every month. They all solve the same problem: **how to make LLMs complete multi-step tasks**. Planner decides what to do, executor does it, reflector evaluates results, then loop.

QuantGPT also uses an AI Agent. But I made a decision opposite to the mainstream:

**I didn't write a single line of Agent code.**

All Agent logic — planning, decision-making, iteration, reflection — is handled entirely by Claude itself. I did one thing: give it a set of good tools. 8 MCP tools, each a stateless pure function. No planner, no router, no reflection loop.

This isn't laziness. It's a strategic choice.

## How Hard Building Agents Really Is

"Building an Agent" sounds easy. Call an API, write a prompt, add a loop. But that's a demo.

### Replicating a domain-depth Agent rivals building Claude Code from scratch

Claude Code looks like "just API calls plus tool use," but its engineering depth goes far beyond the surface: context engineering (when to compress history, when to discard), tool orchestration (priorities, conflict handling, failure retry), safety boundaries (which operations need confirmation), self-correction (how to backtrack from wrong paths).

Building an Agent of equivalent depth for quantitative research means building a "quant Claude Code" from scratch. That's not a workload an open-source project can absorb.

### The Agent frontier moves too fast

Early 2024 best practices (ReAct prompting + fixed tool chains) were replaced by MCP + native tool calling by year's end. The 2025 consensus (multi-Agent collaboration) is being questioned — single Agent + good tools may be more effective.

Agent logic you spend three months building today may become redundant in three months as model capabilities improve. Planner? The model plans on its own. Reflector? The model reflects on its own. Router? The model selects tools on its own.

**You're racing against the speed of model capability growth, and you can't win.**

### The Agent layer is commoditized

Everyone uses the same LLMs — Claude, GPT, DeepSeek. Agent frameworks are fundamentally wrappers around these models. Differentiation isn't in the Agent layer — it's in **how good the tools you feed the Agent are**.

A semantically correct expression parser, a statistically rigorous anti-overfit detection system — these represent months of domain accumulation, not something a prompt swap can replicate.

**Agents are generic; tools are proprietary. Investment should go to the proprietary layer.**

## Riding the Elevator: The Core Advantage of Agent-Infra

The biggest benefit of Agent-Infra isn't "saving effort" — it's that **you automatically benefit from every improvement in Agent capabilities**.

```
LLM Agent (Claude / GPT / DeepSeek)
    │
    └── 8 MCP Tools (Infrastructure Layer)
        ├── run_backtest         Full-market group backtest
        ├── score_factor         0-100 composite scoring
        ├── diagnose_factor      Failure mode diagnosis
        ├── run_anti_overfit     4-layer anti-overfit testing
        ├── run_rolling_validation Walk-forward validation
        ├── validate_expression  Syntax validation (80+ operators)
        ├── list_operators       Operator documentation
        └── list_universes       Universes and benchmarks
```

When Claude upgraded from 3.5 to Opus 4, I didn't change a line of code, but research quality visibly improved. When Claude Code added the skill system, I wrote a `/factor-mine` skill and the Agent immediately gained structured research capabilities — still no infrastructure code changes.

**If you built your own Agent, model upgrades require rewriting Agent logic to leverage new capabilities. If you built infrastructure, model upgrades require nothing — your tools get used by a smarter caller, naturally producing better results.**

You're standing on a rising elevator, powered by the world's largest AI labs.

## Composability: Not Locked to Any Agent

What if tomorrow GPT-5 surpasses Claude at tool calling? **Zero changes, switch directly.**

Tools are stateless pure functions — receive parameters, return results, unaware of the caller's identity. MCP protocol itself is model-agnostic.

If you built your own Agent, your Agent logic is bound to a specific model's prompt format, API structure, and context window assumptions. Switching models = rewriting the Agent. Infrastructure builders don't need to make that choice.

## Testability: You Can't Test Agents, But You Can Test Tools

Agent decisions are stochastic — same input, different runs may produce different outputs. You can't write unit tests for Agent behavior.

But tools are deterministic. QuantGPT has 74 tests covering: mathematical correctness of 80+ operators, cross-sectional/time-series grouping semantics, anti-overfit statistical tests, WQ BRAIN metric calculations, and API boundary guards.

**Regardless of how the Agent calls these tools, the results are correct.** Agent decision quality depends on the model; data correctness depends on infrastructure — the latter is what you can control.

## The Moat Is in Domain Knowledge

Agent framework moats are nearly nonexistent. LangChain users today can migrate to CrewAI tomorrow at minimal cost.

Agent-Infra moats are in **domain knowledge encoding density**:

The expression parser encodes cross-sectional vs. time-series grouping semantic differences, mathematical implementations and edge cases for 80+ operators, WQ BRAIN compatibility dual-mode compilation, and LLM input safety constraints. The anti-overfit system encodes IC stability statistical tests, bull/bear/sideways sub-sample splits, placebo tests, and signal half-life fitting.

**These can't be replicated by prompt engineering.** Every operator's grouping semantics, every statistical test's threshold, every safety limit's value has a concrete failure case behind it.

Switching Agent frameworks is easy. Switching a battle-tested domain toolchain is hard.

## Governance Is Built In

Building your own Agent faces an eternal question: **how to control Agent behavior**.

The mainstream approach is prompt-level constraints. The problem: LLMs can ignore prompts. With infrastructure, governance mechanisms live in code:

```python
# Agent tries to call backtest directly? RuntimeError.
def _require_api_context():
    if not getattr(_api_context, 'active', False):
        raise RuntimeError("Must be called through API")

# Agent generates overly deep expressions? Parser refuses.
MAX_DEPTH = 100
MAX_WINDOW = 500
MAX_EXPRESSION_LENGTH = 1000
```

Agents only respect rules that throw errors. **The Harness is governance.** The tool layer defines what Agents can and can't do — more reliable than any prompt.

## When You Should Build Your Own Agent

Some scenarios genuinely need custom Agents:

- **Extreme determinism requirements** — medical, legal, trade execution, where LLM randomness is intolerable
- **Ultra-high throughput** — thousands of requests per second, where LLM inference latency and cost are unacceptable
- **Insufficient model capabilities** — target users can only use GPT-3.5-level models, requiring external scaffolding to compensate

But if your scenario is: **tasks require flexible judgment, intermediate steps are unpredictable, and sufficiently capable LLMs are available** — building infrastructure is smarter than building Agents.

## One-Line Summary

> Don't compete with the speed of LLM evolution. Build infrastructure, so every time the model gets smarter, your system gets better. Agents are consumables — frameworks become obsolete, prompts lose effectiveness, orchestration logic gets replaced by native model capabilities. Agent-Infra is an asset — domain knowledge doesn't expire, tool correctness doesn't expire, statistical rigor doesn't expire. Build assets, not consumables.

---

*Series: [Agent-Native Architecture](/posts/agent-native-architecture/) · [Harness Is Governance](/posts/harness-is-governance/) · [Skill Orchestration > Agent Loop Chains](/posts/skill-orchestration-over-agent-loops/)*
