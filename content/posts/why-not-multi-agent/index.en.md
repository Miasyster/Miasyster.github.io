---
title: "Why I Didn't Use Multi-Agent Architecture for My Quant Research System"
date: 2026-04-03
draft: false
tags: ["AI Agent", "Multi-Agent", "Quant", "Architecture"]
categories: ["Architecture Decisions"]
summary: "Multi-agent is the hot paradigm in AI engineering. But when building an AI-driven quantitative research system, I chose single agent + state machine. Here's why."
---

## Context

I built a quantitative research system from scratch, covering the full pipeline from data ingestion to paper trading:

```
Data → Factor Research → Anti-Overfit Testing → ML Training → Leakage Detection → Backtest → Signal Generation → Paper Trading
```

The system has three layers: Research Kernel (execution engine), AI Orchestration (coordination), and Human Interface (visualization). The AI's job is to drive automated research iterations — generate factor expressions, submit backtests, evaluate metrics, decide whether to keep optimizing.

When I added the AI orchestration layer, the first architecture question was: multi-agent or single agent?

## The Multi-Agent Temptation

Multi-agent is the hot paradigm in AI engineering right now. Intuitively, quant research seems like a good fit:

- **Research Agent**: generates factor expressions and strategy code
- **Backtest Agent**: submits backtests, collects results
- **Evaluation Agent**: analyzes metrics, checks for overfitting
- **Risk Agent**: validates constraints

Sounds reasonable — clear separation of concerns.

## Why I Didn't Do It

I chose single agent + state machine orchestration. Not because "multi-agent is too hard to implement," but because the preconditions for multi-agent don't hold in my scenario.

### Precondition 1: Different tasks need different toolsets

A core premise of multi-agent is that each agent has its own exclusive tools with minimal overlap.

In quant research, every operation points to the same Kernel API:

```
Research Agent → calls Kernel API to submit factor research
Backtest Agent → calls Kernel API to submit backtest
Eval Agent    → calls Kernel API to query metrics
Risk Agent    → calls Kernel API to check constraints
```

Four agents, one hammer. Splitting them doesn't add capability, only coordination overhead.

### Precondition 2: Tasks can run in parallel

The second premise is that agents can work concurrently, achieving 1+1>2 through coordination.

Quant research is **strictly sequential**:

```
Train → Backtest (needs model) → Evaluate (needs backtest results) → Decide (needs evaluation)
```

You can't backtest without a trained model. You can't evaluate without backtest results. Each step's input is the previous step's output. There's no room for parallelism.

### Precondition 3: Adversarial or debate dynamics add value

Some systems use multi-agent for "red team / blue team" — one agent proposes, another pokes holes. This works for open-ended tasks like writing or design, where quality is subjective.

Quant research evaluation is **numerically deterministic**. A Sharpe Ratio of 1.2 is 1.2. An IC of 0.03 is 0.03. You don't need another agent to "debate" whether the number is right. Anti-overfit testing is a deterministic 4-point checklist, not subjective judgment.

The adversarial mechanism degenerates into if-else logic, which a single agent handles internally.

## What I Chose: Single Agent + State Machine

```
INIT → PLAN → GENERATE_CODE → EXECUTE → EVALUATE → ITERATE/FINISH
                                  ↑          │
                                  └──────────┘
```

One agent drives the entire loop. The state machine controls behavioral boundaries:

- **GENERATE_CODE**: Agent calls LLM to generate factor/strategy code
- **EXECUTE**: Submits to Kernel API for backtesting
- **EVALUATE**: Compares metrics against thresholds, makes continue/stop decision
- **ITERATE**: If unsatisfied, feeds metrics back into GENERATE_CODE

Every iteration has a complete record: code, metrics, evaluation decision, reasoning. Any decision can be traced.

### Why State Machine Beats Multi-Agent Here

| Dimension | Multi-Agent | Single Agent + State Machine |
|-----------|-------------|------------------------------|
| Context | Must be explicitly passed between agents; easy to lose | Naturally shared across all states |
| Debugging | Cross-agent log tracing | Linear trace within single process |
| Latency | Inter-agent communication overhead | Zero communication overhead |
| Accountability | Which agent made the bad call? | State machine records every decision |
| Complexity | N agents × M communication patterns | 1 agent × K states |

The core argument: **when the pipeline is sequential, the toolset is shared, and evaluation is deterministic, multi-agent adds complexity that isn't offset by any gain.**

## When Multi-Agent IS the Right Choice

I'm not saying multi-agent has no value. It's correct in these scenarios:

1. **Naturally parallel tasks**: e.g., simultaneously searching multiple information sources, each agent handling one source, then aggregating. Different toolsets, real parallelism gains.

2. **Subjective adversarial judgment**: e.g., writing tasks — one agent writes, another critiques from the reader's perspective. Evaluation criteria are qualitative, not quantitative. Adversarial dynamics add value.

3. **Non-shareable toolsets**: e.g., one agent can only access the codebase, another can only access production. Permission isolation requires physical separation.

4. **Scale-out**: Same task type needs to process many instances simultaneously. This is essentially concurrency, not "multi-agent," but implementing it with a multi-agent framework is reasonable.

My quant research system **satisfies none of these**.

## On the Multi-Agent Hype

There's a current industry tendency to treat multi-agent as a "more advanced" architecture paradigm, as if more agents = more sophisticated. This mirrors the microservices evolution — around 2015, everyone was splitting into microservices, until many teams realized their systems didn't need it.

The criterion for architecture choice isn't "new" or "trendy," but "does the problem structure match?"

My problem structure is: sequential pipeline, shared toolset, deterministic evaluation. The answer is single agent + state machine.

If your problem structure is different, your answer might be different too. The key is: analyze the problem structure first, then choose the architecture. Not the other way around.

---

*This is the first article in a series documenting architecture decisions I made while building an AI + quant research system. Upcoming topics: MCP's semantic gap problem, AI permission boundary design.*
