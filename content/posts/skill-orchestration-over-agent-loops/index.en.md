---
title: "Skill Orchestration > Agent Loop Chains: Why Dumb Pipelines + Smart Tools Beat Smart Pipelines + Dumb Tools"
date: 2026-04-30
draft: false
tags: ["AI Agent", "MCP", "Orchestration", "Architecture", "LLM"]
categories: ["Architecture Decisions"]
summary: "Current AI Agent frameworks obsess over building complex loop chains: Planner → Executor → Reflector → Re-planner. I chose the opposite: tools are stateless pure functions, and the LLM decides the call sequence itself. Not because loop chains aren't cool — but because they put decision authority in the wrong place."
---

> **Current AI Agent frameworks obsess over building complex loop chains: Planner → Executor → Reflector → Re-planner. I chose the opposite: tools are stateless pure functions, and the LLM decides the call sequence itself. Not because loop chains aren't cool — but because they put decision authority in the wrong place.**

## Two Architectures, One Fundamental Disagreement

In the [previous post](/posts/why-not-multi-agent/), I explained why I didn't use a multi-agent architecture. The conclusion was "use a single Agent + state machine." But that only answered "how many Agents" — it didn't answer a more important question: **where should the Agent's decision boundary be?**

Mainstream Agent frameworks (LangGraph, CrewAI, AutoGen) are all doing the same thing: building an increasingly complex **decision pipeline**. The planner decides what to do, the executor does it, the reflector evaluates the result, then back to the planner. The pipeline itself is "smart" — it knows when to loop, when to exit, when to backtrack.

I made the opposite choice: **the pipeline is "dumb," the tools are "smart."**

Concretely: QuantGPT's MCP server exposes 10 independent tools (run_backtest, score_factor, diagnose_factor, run_anti_overfit, etc.), each a stateless pure function — receives parameters, returns complete results, records no call history, has no knowledge of what was called before.

Where's the "pipeline"? It doesn't exist. The LLM Agent (Claude) sees all tool descriptions directly and decides what to call next. No planner, no router, no reflection loop — all of that is handled by the LLM's own reasoning capability.

This isn't laziness. It's an architecture decision.

## The Hidden Assumption in "Smart Pipelines"

When you build a Planner → Executor → Reflector loop chain, you're implicitly making an assumption: **you know better than the LLM how decisions should flow.**

This assumption was reasonable in 2023. GPT-3.5's reasoning was limited, and it genuinely needed external scaffolding to guide it — telling it to "plan first, then execute," "reflect after execution," "decide whether to retry after reflection." The framework's value was compensating for model limitations.

By 2025, this assumption is increasingly questionable.

Claude, GPT-4o, and DeepSeek-V3 have powerful tool selection and multi-step reasoning capabilities. Give them 10 tools and a goal, and they can plan the call sequence themselves. They can even dynamically adjust strategy based on intermediate results — which is exactly what loop chains *cannot* do, because the branching logic is pre-coded by you.

**The problem with smart pipelines isn't that they don't work — it's that they freeze a specific decision flow, and that flow is likely suboptimal.**

A concrete example. In my factor research scenario, a standard Agent loop chain would look like this:

```
Generate factor → Backtest → Score → Score high enough?
                                      ├── Yes → Anti-overfit test → Pass? → Submit
                                      └── No  → Mutate → Back to generate
```

Looks reasonable. But real research produces situations the loop chain can't handle:

- The Agent finds a factor with mediocre scores, but diagnostics show the window parameter is just too small — a simple parameter tweak would suffice, no need for the full mutation-regeneration flow
- The Agent discovers two independent factors each with Sharpe 1.3 and wants to try combining them directly, but the loop chain has no branch for that
- While running anti-overfit tests, the Agent notices IC suddenly decays in a specific subsample and wants to run diagnostics to check if it's an industry exposure issue

These are **researcher's improvisational judgments**, not pre-codable branches. Every if-else in a loop chain requires the developer to foresee the scenario. What you can't foresee, you can't handle.

## Three Principles for MCP Tool Design

Since we're not using loop chains, tool design becomes critical. I followed three principles:

### 1. Stateless: Every Tool Call Is Self-Contained

```python
@mcp.tool()
async def run_backtest(expression: str, universe: str = "hs300", ...):
    # Reads no global state
    # Depends on no "previous call result"
    # Returns a complete backtest report (metrics + diagnostics + scores)
    ...
```

A tool doesn't know whether it's being called "for the first time" or "on iteration 15." It doesn't care about context. This means the Agent can call any tool in any order, at any frequency, with no state-inconsistency issues.

Compare with loop chains: if the Executor depends on global variables set by the Planner, the Agent must strictly follow the Planner → Executor sequence. Breaking the sequence breaks consistency.

### 2. Complete Returns: Results Are Self-Contained

Every tool's return value includes all relevant information. The Agent doesn't need to call another tool to "supplement" the result.

`run_backtest` returns not just Sharpe and Returns — it simultaneously returns group returns, turnover, max drawdown, industry exposure, and factor loadings. The Agent sees the complete picture and decides what to do next.

This contradicts the Unix philosophy of "small and focused." Small and focused works well for humans composing pipelines (`cat | grep | sort`), but LLMs aren't good at combining ten calls to piece together a complete result — they're good at extracting key information from a single complete result. **Tool design should match the caller's cognitive model, not the designer's aesthetic preference.**

### 3. Semantic Naming: Tool Names Describe Intent

```
run_backtest           — I want to know how this factor performs
score_factor           — I want a composite score for this factor
diagnose_factor        — I want to know why this factor performs poorly
run_anti_overfit       — I want to know if this factor is overfitting
run_rolling_validation — I want to know this factor's stability across time periods
```

The LLM selects tools based on natural language descriptions. The closer the tool name is to intent description, the higher the LLM's selection accuracy. No router needed — semantic matching *is* the best routing.

## "But You Need a Planner to..."

The typical objection goes: "Without a planner, how does the Agent know what to do first?"

The answer: **it already knows.**

Give Claude a goal — "discover a factor with Fitness > 1.0" — and 10 tool descriptions, and it will spontaneously:

1. Check available operators (`list_operators`)
2. Design a factor expression
3. Validate syntax (`validate_expression`)
4. Backtest (`run_backtest`)
5. Score (`score_factor`)
6. If the score is low, diagnose the problem (`diagnose_factor`)
7. Modify the expression based on diagnostics, back to step 3
8. If the score is high, test for overfitting (`run_anti_overfit`)
9. Submit if passed

Nobody taught it this workflow. It derived it from the tools' semantic descriptions.

More importantly, it will **deviate from this workflow** based on intermediate results. If diagnostics reveal the problem isn't the expression but the stock universe, it switches universes and re-runs instead of continuing to iterate in the same universe. If anti-overfit testing shows IC decay concentrated in H2 2022, it hypothesizes a market structure change and proactively re-tests with a shorter window.

**This flexibility cannot be pre-coded in a loop chain.** You can write if-else for 10 scenarios, but the 11th requires a code change. An LLM can handle arbitrarily many scenarios, as long as the tools' capabilities cover them.

## This Isn't "No Architecture"

Some will say: "You're just pushing architecture decisions to the LLM — that's not good engineering practice."

No. Architecture still exists — it's just in a different place.

Loop chain architecture constrains at the **pipeline layer** — the pipeline defines which steps are possible, where to branch, when to loop. The Agent's freedom is bounded by the pipeline.

My architecture constrains at the **tool layer** — tools define what the Agent can do, what information each operation returns, and that operations have no implicit dependencies. The Agent's freedom is bounded by the capability set.

An operating system analogy: loop chains are a monolithic kernel, all decision logic compiled together; MCP toolsets are a microkernel, exposing only system calls, with scheduling logic in userspace (the LLM).

Both architectures have constraints. The difference: pipeline constraints are **process constraints** (you must follow this sequence), tool constraints are **capability constraints** (you can only use these operations).

Process constraints go stale easily — when the research paradigm changes, you have to rewrite the pipeline. Capability constraints are more stable — as long as the operations' semantics don't change, the Agent can freely compose new workflows.

## Results in Practice

This architecture has been running on QuantGPT for several months, producing 3 factors formally submitted to WorldQuant BRAIN (best Fitness 1.26, Sharpe 1.77), all passing IS tests.

A few specific observations:

**The Agent invented research paths I hadn't anticipated.** For example, it discovered that `ts_av_diff` and `rank(debt/enterprise_value)` each performed mediocrely, but proactively tried an additive combination that jumped Fitness from 0.7 to 1.26. No code told it to "try combining" — it inferred from `score_factor` results that the two signals were complementary.

**Debugging shifted from "tracing pipeline state" to "reading Agent logs."** Every tool call and return value is an independent JSON entry — arranged chronologically, they form a complete research record. No need to understand the pipeline's internal state machine.

**Adding tools doesn't affect Agent behavior.** I later added a `wq_brain_batch_submit` tool, and the Agent automatically discovered and started using it — no "pipeline logic" to update, because there's no pipeline logic to begin with.

## When This Approach Is Wrong

Honestly, there are scenarios where this approach shouldn't be used:

- **When model capability is insufficient.** If your LLM is GPT-3.5 level, it genuinely needs external scaffolding to guide decisions. Skill orchestration implicitly depends on a sufficiently intelligent caller.
- **When the process is deterministic.** If your workflow has no branching judgment (ETL pipelines, data cleaning), writing code directly is more reliable and cheaper than having an LLM decide.
- **When throughput is high.** For scenarios processing thousands of requests per second, LLM inference latency and cost are unacceptable. Hard-coded pipelines are more appropriate.

But these exceptions actually clarify the decision criteria: **when the task requires flexible judgment, intermediate steps are unpredictable, and you have a sufficiently intelligent caller, Skill orchestration beats Agent loop chains.**

## One-Line Summary

> Don't use code to freeze decision capabilities the LLM already has. Give it good tools and let it decide how to use them.

In 2023, we needed frameworks to compensate for model shortcomings. In 2025, we need to step back and return pre-coded decision logic to the model.

Smart pipelines + dumb tools are a relic of the previous era. Dumb pipelines + smart tools are the LLM-native architecture.

---

*Previous posts in the series: [Why I Didn't Use Multi-Agent Architecture for Quant Research](/posts/why-not-multi-agent/) · [MCP's Problem Isn't the Protocol — It's the Semantic Gap](/posts/mcp-semantic-gap/)*
