---
title: "Agent-Native Architecture: Designing Systems for Agents, Not Humans"
date: 2026-05-01
draft: false
tags: ["AI Agent", "Architecture", "MCP", "System Design"]
categories: ["Architecture Decisions"]
summary: "When the system operator changes from a human to an LLM Agent, design principles need fundamental rethinking. Humans need GUIs and documentation. Agents need semantically clear tools and constraints that throw errors."
---

> **When the system operator changes from a human to an LLM Agent, design principles need fundamental rethinking. Humans need GUIs and documentation. Agents need semantically clear tools and constraints that throw errors.**

## The Operator Changed. So Must the Design.

Traditional software is designed for humans: GUIs guide workflows, documentation explains features, error messages help humans understand problems.

When the operator becomes an LLM Agent, this design breaks down. Agents don't look at GUIs, don't read documentation (at least not the way humans do), and don't need friendly error messages — they need **machine-parseable error information** to adjust their next action.

Agent-Native architecture isn't "add an API for the Agent." It's rethinking every design decision from the premise that the system's operator is an Agent.

## Principle 1: Tools Match the Caller's Cognitive Model

Humans excel at composing small tools to accomplish tasks (Unix philosophy: `cat | grep | sort`). LLMs don't — they excel at extracting key information from a single complete result.

QuantGPT's `run_backtest` returns everything in one call: Sharpe, IC, group returns, turnover, max drawdown, industry exposure, factor loadings. Not split into 7 small tools for the Agent to call one by one.

```python
# Human-friendly design: 7 small tools
get_sharpe(factor_id)
get_ic(factor_id)
get_turnover(factor_id)
# ...Agent needs 7 calls to see the full picture

# Agent-Native design: 1 complete tool
run_backtest(expression, universe) → {sharpe, ic, turnover, drawdown, ...everything}
# Agent sees the complete picture in one call, decides what to focus on
```

**Tool design should match the caller's cognitive model, not the designer's aesthetic preference.** "Small and focused" is a human aesthetic; "complete and self-contained" is an Agent's need.

## Principle 2: Errors Are Interface, Not Exceptions

Humans see error messages, think about the cause, and manually fix the problem. Agents parse error content and automatically adjust.

This means error message design shifts from "help humans understand" to "help Agents act":

```python
# Human-friendly error
raise ValueError("Expression syntax error")

# Agent-Native error
raise ValueError(
    "At character 23: 'ts_regrssion' is not a valid operator. "
    "Did you mean 'ts_regression'? "
    "Available time-series operators: ts_mean, ts_std, ts_corr, ts_regression, ..."
)
```

The second error contains three layers: **what's wrong** (location), **possible correction** (suggestion), **available alternatives** (operator list). The Agent can parse this and directly generate a corrected expression without an additional `list_operators` call.

## Principle 3: Constraints in Code, Not Documentation

Documentation constraints ("please call through the API") have zero binding force on Agents. Agents take the shortest path — if directly importing a function is faster, they'll bypass the API.

Agent-Native system constraints must be **runtime-enforced**:

```python
_api_context = threading.local()

def _require_api_context():
    if not getattr(_api_context, 'active', False):
        raise RuntimeError("Must be called through API")
```

Agents only respect rules that throw errors. Rules that don't throw errors don't exist.

Similarly, the expression parser's security limits:

```python
MAX_DEPTH = 100           # Recursion depth
MAX_WINDOW = 500          # Rolling window
MAX_EXPRESSION_LENGTH = 1000  # Expression length
```

Not advisory values — hard limits. Exceed them and an exception is thrown, forcing the Agent to revise its input.

## Principle 4: Stateless > Stateful

Stateful tools mean call order matters — you must call A before B, otherwise B can't read the state A set. This is an implicit constraint on the Agent, and it's a documentation-level one (you need to tell the Agent "call A first"), not a code-level one.

Stateless tools eliminate this problem. Every call is independently complete, and the Agent can call any tool in any order at any frequency.

```python
@mcp.tool()
async def run_backtest(expression: str, universe: str = "hs300", ...):
    # Reads no global state
    # Depends on no previous call's result
    # Returns a complete backtest report
    ...
```

The tool doesn't know if it's being called "for the first time" or "on iteration 15." It doesn't care about context. This means any decision path the Agent takes is valid — there's no such thing as "wrong call order."

## Principle 5: Semantic Naming Is Routing

Human systems need routers to dispatch requests. Agent-Native systems don't — the tool name itself is the routing.

```
run_backtest           — I want to know how this factor performs
score_factor           — I want a composite score
diagnose_factor        — I want to know why it's underperforming
run_anti_overfit       — I want to know if it's overfitting
```

LLMs select tools based on names and descriptions. The closer the name is to intent description, the higher the selection accuracy. No need to write `if intent == "diagnose": call diagnose_factor` in Agent code — the model does this mapping naturally.

## Testability: The Underestimated Advantage

Agent decisions are stochastic — same input, different runs may produce different outputs. You can't write unit tests for Agent behavior.

But the Agent-Native tool layer is deterministic. QuantGPT has 74 tests covering:

- Mathematical correctness of 80+ operators
- Cross-sectional/time-series grouping semantics
- Anti-overfit statistical tests
- WQ BRAIN metric calculations

Tests guarantee: regardless of how the Agent calls the tools, the returned results are correct. Agent decision quality depends on model capability; data correctness depends on infrastructure — the latter is what you can control and verify.

## One-Line Summary

> Designing for Agents and designing for humans are two different things. Complete returns > small-and-focused, runtime enforcement > documentation constraints, stateless > stateful, semantic naming > explicit routing. When your operator is an LLM, every design decision needs re-examination.

---

*Series: [Why Agent-Infra, Not Agents](/posts/why-agent-infra-not-agent/) · [Harness Is Governance](/posts/harness-is-governance/) · [Skill Orchestration > Agent Loop Chains](/posts/skill-orchestration-over-agent-loops/)*
