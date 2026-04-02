---
title: "Why I Didn't Use Multi-Agent Architecture for My Quant Research System"
date: 2026-04-02
draft: false
tags: ["AI Agent", "Multi-Agent", "Architecture", "State Machine"]
categories: ["Architecture Decisions"]
summary: "Multi-agent is the hot paradigm in AI engineering. I chose single agent + state machine for my AI-driven quant research system. Not because multi-agent is too hard, but because the problem structure doesn't match."
---

## A Counterintuitive Choice

Since 2024, multi-agent has become almost the default architecture for AI systems. CrewAI, AutoGen, MetaGPT — every framework tells you: split tasks across multiple agents, let them collaborate, get better results.

When designing an AI-driven quantitative research system, I seriously evaluated this path and ultimately rejected it. I chose what looks like a "dated" approach: single agent + finite state machine.

This wasn't a resource constraint or technical limitation. It was a deliberate choice after analyzing the problem structure.

## The Structure, Not the Surface

A quant research workflow has many apparent "roles": someone generates strategies, someone runs backtests, someone evaluates results, someone checks risk. It's natural to map each role to an agent.

But this is reasoning by analogy from org charts to system architecture — a surface-level inference.

The real questions to analyze are structural:

**Are tasks parallel or sequential?**

The core quant research pipeline is strictly sequential: train model → backtest (needs model output) → evaluate (needs backtest results) → decide (needs evaluation). Each step's input is the previous step's output. Zero parallelism. The coordination benefit of multi-agent is zero in a serial pipeline.

**Do different agents need different toolsets?**

A core assumption of multi-agent is that each agent has exclusive tools with minimal overlap. In quant research, "generate strategy," "submit backtest," "query metrics," and "check constraints" all point to the same execution engine API. Four agents, one hammer. Splitting them adds communication overhead without adding capability.

**Does evaluation require subjective debate?**

Some systems use multi-agent for red team / blue team — one proposes, one attacks. This works for subjective tasks like writing or product design. But quant evaluation is numerically deterministic: a Sharpe Ratio of 1.2 is 1.2. An IC of 0.03 is 0.03. No agent needs to "debate" whether the number is trustworthy. Anti-overfit testing is a deterministic checklist, not a judgment call. The adversarial mechanism degenerates into if-else logic.

All three conditions unmet. Multi-agent adds complexity here without adding value.

## The Approaches I Considered

### Approach A: Multi-Agent Collaboration

A CrewAI-style setup — define researcher, backtester, evaluator, risk officer as four roles, coordinate through message passing.

Where it's correct: naturally parallel tasks (searching multiple sources simultaneously), non-shareable toolsets (codebase vs. production environment permission isolation), multi-perspective debate needed (product design).

Why it doesn't fit my case: serial pipeline means zero parallelism gain; shared toolset makes splitting pointless; numerical evaluation needs no debate. Extra cost: serialization overhead for inter-agent context passing and cross-agent log tracing during debugging.

### Approach B: Single Agent + Finite State Machine

One agent drives the entire research loop. The state machine controls behavioral boundaries and transitions:

```
INIT → GENERATE → EXECUTE → EVALUATE ──→ FINISH
                     ↑          │
                     └── ITERATE ┘
```

The core idea in one sentence: **replace "coordination protocol between multiple agents" with "state transition rules within a single agent."**

Each state transition is deterministic: if EVALUATE metrics meet the threshold, transition to FINISH; if not, transition to ITERATE back to GENERATE. No inter-agent "discussion" about what to do next.

## The Key Judgment Call

The pivotal insight wasn't a technical comparison. It was recognizing a deeper distinction: **multi-agent frameworks solve "coordination" problems, but my system doesn't have a "coordination" problem.**

What my system has is an "orchestration" problem — a deterministic pipeline that needs to be automated, with LLM generation capability inserted at specific points. This is closer to workflow engine territory, not multi-agent collaboration territory.

This recognition came from an analogy: traditional CI/CD pipelines also have multiple stages (build → test → deploy), and could theoretically be mapped to multiple "agents." But nobody builds CI/CD with multi-agent frameworks, because everyone intuitively knows it's a serial orchestration problem. The quant research iteration loop is structurally isomorphic to a CI/CD pipeline.

## Results

The state machine approach has been running for several months. A few quantifiable comparison points:

- Zero context loss: single agent naturally shares the full research history, no intermediate results need to be passed between agents
- Linear traceability: every state transition has a complete record (input code, output metrics, evaluation decision, decision reasoning) — just walk the timeline when debugging
- Debugging time went from "piecing together logs across multiple agents" to "locating within a single state sequence"

Haven't encountered a scenario requiring parallelism. If I ever need to run 10 independent factor research tasks simultaneously, that's a concurrency scheduling problem — asyncio.gather handles it fine, no need for inter-agent communication.

## What This Decision Taught Me

I distilled one judgment principle from this practice:

> **Before choosing an architecture, identify the problem's structural type. Not every system with "multiple apparent roles" needs multi-agent.**

The method is three questions:
1. Is there parallelism between tasks?
2. Do toolsets need isolation?
3. Does evaluation require subjective debate?

If none are satisfied, don't use multi-agent. Even if one is satisfied, evaluate whether the coordination complexity introduced is covered by the gains.

This mirrors the microservices trajectory exactly. Around 2015, everyone was splitting into microservices, until many teams discovered their systems didn't need it — their problem was a deployment problem, not a service boundary problem. Multi-agent is currently going through the same inflation phase. The hype will pass. Problem structure analysis won't.

---

*First article in the series. Next: [MCP's Problem Isn't the Protocol — It's the Semantic Gap](/en/posts/mcp-semantic-gap/).*
