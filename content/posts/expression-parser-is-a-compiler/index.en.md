---
title: "The Expression Parser Is a Compiler, Not eval()"
date: 2026-04-29
draft: false
tags: ["Compiler", "Expression Parser", "Architecture", "Security"]
categories: ["Architecture Decisions"]
summary: "QuantGPT's core is an 870+ line hand-written recursive descent parser supporting 80+ operators, automatic cross-sectional/time-series grouping, and dual-mode compilation. Not because I didn't know eval() is simpler — but because what eval() can't do happens to be what matters most."
---

> **QuantGPT's core is an 870+ line hand-written recursive descent parser supporting 80+ operators, automatic cross-sectional/time-series grouping, and dual-mode compilation. Not because I didn't know eval() is simpler — but because what eval() can't do happens to be what matters most.**

## Why eval() Won't Work

Factor expressions look like math formulas:

```
-1 * rank(ts_av_diff(close, 10)) + rank(debt / enterprise_value)
```

The intuitive implementation: define `rank`, `ts_av_diff` as Python functions, then `eval()` the entire string. Three lines of code, done.

But this system's caller is an LLM Agent. Agent-generated expressions are unpredictable. `eval()` means arbitrary code execution — not just a "security concern," but **a systematic inability to constrain the Agent's behavior space**. You can't distinguish "a legitimate factor expression" from "arbitrary Python hallucinated by the Agent."

Even ignoring security, `eval()` can't solve factor computation's core challenge: cross-sectional and time-series operators have completely different grouping semantics.

## Cross-Sectional vs Time-Series: Same Parentheses, Different Semantics

```
rank(close)              — grouped by trade_date, ranks across all stocks on the same day
ts_mean(close, 20)       — grouped by stock_code, rolling average along a single stock's history
```

`rank()` is a cross-sectional operator: all stocks on the same day form a cross-section, and rank is computed within it. `ts_mean()` is a time-series operator: a single stock's price history is a time series, and the mean rolls along the time axis.

Both have similar function signatures (receiving a data column), but the underlying `groupby` is completely different. `eval()` can't infer this semantic distinction from syntax. A hand-written parser can:

```python
# When the parser encounters rank(), it injects cross-sectional grouping
s.groupby(df['trade_date']).rank(pct=True)

# When it encounters ts_mean(), it injects time-series grouping
_apply_ts_op_per_stock(df, lambda g: g.rolling(window).mean())
```

**Grouping logic is fully transparent to the expression author.** Someone writing `rank(close)` doesn't need to know it's grouped by date under the hood — the parser handles it automatically based on operator type. This is what "compilation" means: translating high-level intent into correct low-level operations.

## Dual-Mode Compilation

The same expression can compile to two targets:

- **`mode="local"`**: All 80+ operators available, including local extensions like `tanh`, `sigmoid`, `rsi`, `macd`. For rapid experimentation and free exploration.
- **`mode="wq"`**: Only the WorldQuant BRAIN-compatible operator subset. For pre-submission validation — if an expression uses an unsupported operator, it errors at compile time, not at platform submission.

28 WQ-only remote operators (`vector_neut`, `ts_regression`, `bucket`, etc.) are registered as stubs in local mode — calling them raises a `RuntimeError` explaining "this operator is only available on the WQ BRAIN platform."

This lets the Agent explore freely (local mode), then switch to wq mode for compliance checks before submission. Both stages use the same parser — only the compilation target differs.

## Security Constraints

LLM Agents generate all kinds of edge cases. The parser hard-codes three defensive limits:

```python
MAX_WINDOW = 500         # Rolling window cap
MAX_DEPTH = 100          # Recursion depth limit
MAX_EXPRESSION_LENGTH = 1000  # Expression character count cap
```

- **Window cap**: Prevents `ts_mean(close, 99999)` from consuming all memory
- **Recursion depth**: Prevents `rank(rank(rank(rank(...))))` infinite nesting
- **Expression length**: Prevents the Agent from generating excessively long composite expressions

These aren't advisory values — they're runtime-enforced. Exceeding limits throws an exception immediately, forcing the Agent to revise its expression.

## 80+ Operators by Category

| Category | Count | Examples |
|:---------|:------|:---------|
| Unary | 8 | `log`, `abs`, `sign`, `scale`, `sqrt` |
| Cross-sectional | 4 | `rank`, `zscore`, `group_rank`, `group_zscore` |
| Time-series | 18+ | `ts_mean`, `ts_std`, `ts_corr`, `decay_linear`, `ts_av_diff` |
| Technical indicators | 8 | `rsi`, `macd`, `ema`, `sma`, `atr`, `obv`, `boll_*` |
| Conditional/Special | 5 | `where`, `trade_when`, `clip`, `indneutralize`, `ternary` |
| WQ Remote | 28 | `vector_neut`, `ts_regression`, `bucket`, `quantile` |
| Arithmetic | 11 | `+`, `-`, `*`, `/`, `^`, `>`, `<`, `==`, `!=`, `and`, `or` |

Each operator's grouping semantics (cross-sectional/time-series/scalar) is explicitly registered in the parser. There are no "generic operators" — every operator must declare its computation domain.

## Why Not Use an Existing Parser Framework

Python has `lark`, `ply`, `pyparsing`. Why hand-write?

1. **Semantic actions are deeply coupled with parse depth.** The parser doesn't just produce an AST — it directly generates a Pandas computation graph during parsing. `lark`'s Transformer can do this, but the code wouldn't be shorter, and adds an abstraction layer.
2. **Dual-mode switching requires parse-time decisions.** `mode="wq"` needs to error immediately on encountering an incompatible operator, not parse-then-check. Embedding this in recursive descent logic is most natural.
3. **Error messages are for the Agent.** "At character 23: 'ts_regrssion' is not a valid operator, did you mean 'ts_regression'?" This kind of error message with correction suggestions is hard to achieve with generic parser frameworks.

870 lines sounds like a lot, but it contains parsing, compilation, safety checks, error handling, and complete implementations of 80+ operators. It's the most stable module in the system — across 3 factors formally submitted to WorldQuant BRAIN (all IS tests passed), the parser had zero bugs.

## One-Line Summary

> The parser isn't "input handling" — it's the unified implementation of the type system, security boundary, and compilation target. When your caller is an LLM, the parser is your first and most important line of defense.

---

*Series: [API Guard Pattern](/posts/api-guard-pattern/) · [Sandbox Defense in Depth](/posts/sandbox-defense-in-depth/) · [Skill Orchestration > Agent Loop Chains](/posts/skill-orchestration-over-agent-loops/)*
