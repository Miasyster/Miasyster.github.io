---
title: "MCP's Problem Isn't the Protocol — It's the Semantic Gap"
date: 2026-04-02
draft: false
tags: ["MCP", "AI Agent", "Vibe Coding", "Intent Validation"]
categories: ["Architecture Decisions"]
summary: "MCP's JSON-RPC transport works fine. The real problem: there's a semantic gap between natural language instructions and structured system operations, and the current MCP ecosystem doesn't address it at all. I built an Intent Validator to fix this in my system."
---

## The Problem

I have an AI-driven quantitative research system. An AI agent submits various research tasks through MCP tools: ML training, backtesting, factor research, etc. Each task type has dozens of parameters with complex business constraints between them.

Here's one critical rule for ML backtesting:

> **Never set start_date or end_date.** Always use full-period OOF (Out-of-Fold) backtesting — in-sample uses cross-validated predictions, out-of-sample uses the final model. Restricting the date range gives too few data points, making results unreliable.

This rule lives in the MCP server's instructions, told to the agent in natural language:

```
ml_backtest: ALWAYS use full period (do NOT set start_date/end_date).
Empty dates = OOF backtest: in-sample uses cross-validated predictions,
out-of-sample uses final model.
NEVER restrict to test period only — too few data points.
```

The problem: **the agent can completely ignore this.**

The LLM reads the instruction and usually complies. But occasionally — especially when the user provides an explicit date range — it fills in the dates anyway. Nothing stops it. The request passes through MCP legitimately, reaches the REST API, passes Pydantic validation (because start_date is a legal string field), gets submitted, and produces unreliable results with no error.

**This is not a bug in MCP.** The JSON-RPC transport works fine. The tool schema definition is fine. The problem is: natural language rules have no code-level enforcement.

## Three Validation Layers, with a Gap in the Middle

My system had three validation layers, but the critical middle one was empty:

```
Layer 1: MCP instructions (natural language)  → Agent can ignore
Layer 2: ???                                   → Empty
Layer 3: Pydantic model validation (type check) → start_date is a valid string, passes
```

Layer 1 is "advice." Layer 3 is "type checking." The missing middle layer is "code-level enforcement of business rules."

This gap exists in every MCP application. MCP's tool schema can describe parameter types (string, int, array), but cannot describe cross-parameter business constraints ("if symbols is empty, universe must be non-empty", "ml_backtest must not set dates").

## The Solution: Intent Validator

I added an Intent Validator between the MCP tool layer and the HTTP dispatch:

```
Agent calls MCP submit()
  → JSON parsing
    → Intent Validator ← NEW: code-level business rule enforcement
      → HTTP call to Orchestration API
        → Pydantic type validation
```

### 1. Registry-Based Rule Architecture

Each task type can register multiple validation rules. Adding rules requires zero changes to MCP server code:

```python
@_register("ml_backtest")
def _no_date_restriction(config, objective):
    violations = []
    if config.get("start_date"):
        violations.append({
            "rule": "ml_backtest_no_start_date",
            "error": "start_date is set but ML backtest requires full period.",
            "fix": "Remove start_date for full-period OOF backtest.",
        })
    return violations
```

### 2. Hard Blocks + Soft Warnings

Each rule has a severity level. Hard errors reject the submission with fix instructions. Warnings are attached to the success response:

```python
# Hard error: reject
{"rule": "ml_backtest_no_start_date", "error": "...", "fix": "..."}

# Soft warning: allow but inform
{"rule": "ml_backtest_split_dates_recommended", "severity": "warning",
 "error": "split_dates not provided", "fix": "Pass split_dates from training result"}
```

### 3. Actionable Error Messages

Every violation has three fields: `rule` (machine-readable ID), `error` (what's wrong), `fix` (how to correct it). When rejected, the agent can directly follow the `fix` field to adjust parameters and resubmit, with no human intervention needed.

### 4. Unified Coverage Across Both Entry Points

The same rules cover both MCP and HTTP entry paths. The MCP path calls the validator inside `submit()`. The HTTP path triggers it through Pydantic's `model_validator` during request deserialization:

```python
class IntentValidatedModel(BaseModel):
    _intent_task_type: str = ""

    @model_validator(mode="after")
    def _run_intent_validation(self):
        result = validate_intent(self._intent_task_type, self.model_dump(), ...)
        if hard_errors:
            raise ValueError(f"Intent validation failed: {msg}")
        return self

class MLBacktestRequest(IntentValidatedModel):
    _intent_task_type: str = "ml_backtest"
    model_id: str
    # ...
```

Whether the agent submits through MCP or a script hits the REST API directly, the same business rules are enforced.

## Registered Rules

| Task Type | Rule | Level |
|-----------|------|-------|
| ml_training | Empty symbols requires universe | Hard block |
| ml_training | model_type whitelist | Hard block |
| ml_backtest | No start_date/end_date allowed | Hard block |
| ml_backtest | model_id required | Hard block |
| ml_backtest | Universe or symbols required | Hard block |
| ml_backtest | split_dates recommended | Warning |
| ml_predict | model_id + universe required | Hard block |
| rolling_factor_research | Date range >= window total | Hard block |
| data_update | update_mode enum validation | Hard block |
| Common | Date format YYYY-MM-DD | Hard block |
| Common | start_date < end_date | Hard block |

## How This Differs from a Harness

Claude Code has permission modes and hooks. Cursor has rules files. These are harness patterns — external constraints on agent behavior.

Intent Validator is not a harness. The distinction:

- **Harness** controls "can the agent call this tool?" — tool-level granularity
- **Intent Validator** controls "when the agent calls this tool, does the parameter combination satisfy business semantics?" — parameter-level granularity

A harness can "prohibit the agent from calling submit." It cannot "allow submit, but reject if task_type is ml_backtest and start_date is non-empty." Cross-parameter conditional constraints must be implemented at the application layer.

## Implications for Vibe Coding

The core tension of vibe coding is: **natural language ambiguity vs. system operation precision.** A user says "run a backtest for me" — that maps to dozens of possible parameter combinations, only a few of which are reasonable.

Current approaches concentrate on two extremes:

- **Improve LLM understanding**: make the model better at following instructions → has a ceiling, will never be 100%
- **Tighten tool schemas**: use stricter parameter definitions → JSON Schema's expressiveness is limited, can't describe cross-parameter constraints

The missing middle layer is Intent Validation: **after LLM output, before system execution, enforce business semantics with code.** Doesn't depend on LLM comprehension ability. Not limited by JSON Schema expressiveness.

This pattern applies to any scenario where agent operations have business constraints — not just quant research. A medical agent shouldn't prescribe contraindicated drug combinations. A legal agent shouldn't cite repealed statutes. An ops agent shouldn't run full deployments during peak hours. These constraints all need code-level enforcement, not prompt-level suggestions.

---

*Second article in the series. Previous: [Why I Didn't Use Multi-Agent Architecture for My Quant Research System](/en/posts/why-not-multi-agent/).*
