---
title: "API Guard Pattern: Why Calling Functions Directly Is Forbidden"
date: 2026-04-28
draft: false
tags: ["Architecture", "Security", "API Design", "Python"]
categories: ["Architecture Decisions"]
summary: "QuantGPT uses threading.local to enforce a runtime guard: all backtest calls must go through the API boundary. Direct function calls raise an exception. Not because the function is dangerous — but because a system without boundaries can't be audited."
---

> **QuantGPT uses threading.local to enforce a runtime guard: all backtest calls must go through the API boundary. Direct function calls raise an exception. Not because the function is dangerous — but because a system without boundaries can't be audited.**

## The Problem: LLM Agents Take Shortcuts

When you expose a Python project to an LLM Agent, the Agent tends to do the most efficient thing: directly `import` your functions, bypassing the API layer.

```python
# An Agent will easily write code like this
from quantgpt.backtest import run_factor_backtest
result = run_factor_backtest("rank(close)", "hs300")
```

This code **works**. The backtest engine doesn't care who calls it. But it creates an unauditable call path — no task ID, no logs, no rate limiting, no permission checks. In a system where an Agent can autonomously run dozens of iterations, losing auditability means losing control.

Documentation conventions ("please call through the API") have no binding force on LLM Agents. The Agent sees that the `import` path is shorter, and uses it.

## The Solution: Runtime Enforcement

```python
_api_context = threading.local()

def _require_api_context():
    if not getattr(_api_context, 'active', False):
        raise RuntimeError(
            "run_factor_backtest() must be called within api_context()"
        )

@contextmanager
def api_context():
    _api_context.active = True
    try:
        yield
    finally:
        _api_context.active = False
```

The first line of `run_factor_backtest()` is `_require_api_context()`. Any call not wrapped in `api_context()` immediately raises a `RuntimeError`.

Thread isolation uses `threading.local()` — different threads have independent contexts, and process pool workers are isolated from each other.

## Legitimate Call Sites

Only a few places in the system wrap `api_context()`:

```python
# task_executor.py — the entry point for all backtest tasks
def _run_backtest_in_process(expression, universe, ...):
    enable_api_context()
    try:
        return run_factor_backtest(expression, universe, ...)
    finally:
        disable_api_context()
```

API routes, MCP tools, the iteration engine — all submit tasks through `task_executor`, which handles enabling the context. The call chain becomes:

```
HTTP/MCP request → task_executor → enable_api_context → run_factor_backtest
```

Adding a new call site? You must explicitly wrap `api_context()`. Forget to wrap it? Runtime explosion. There's no way to "quietly bypass" the guard.

## What About Tests

Tests also go through this guard. Enabled globally via a pytest autouse fixture:

```python
@pytest.fixture(autouse=True)
def _enable_api_context():
    with api_context():
        yield
```

All tests automatically run inside `api_context()`. Tests cover the real call path, not some special "test mode."

## Why Not Decorators/Middleware

Common alternatives:

- **Decorators**: Marked at function definition, but callers are unaware — doesn't change calling habits
- **Middleware**: Only protects the HTTP layer, not internal Python calls
- **Documentation**: Effective for humans, ineffective for Agents

The `threading.local` guard's key property is that **callers must actively cooperate** — you must enter `api_context()` before calling. This turns a "convention" into a "constraint."

## The Deeper Principle

This pattern doesn't solve a security problem (the backtest function itself is harmless) — it solves a **boundary problem**.

In an environment where an LLM Agent can freely call arbitrary Python functions, without runtime-enforced boundaries, the system degrades into a big ball of mud — all call paths are legitimate, audit logs can't capture direct calls, and you can't distinguish "a research task initiated through the API" from "the Agent casually ran a backtest."

Boundaries don't restrict freedom — they make freedom trackable.

Code constraints > documentation constraints. Runtime enforcement > static analysis. This is especially true for LLM Agents — they only respect rules that throw errors.

---

*Series: [AI as Operator, Kernel as Law](/posts/ai-as-operator/) · [MCP's Problem Isn't the Protocol — It's the Semantic Gap](/posts/mcp-semantic-gap/) · [Skill Orchestration > Agent Loop Chains](/posts/skill-orchestration-over-agent-loops/)*
