---
title: "Why I Didn't Use LangChain — The Design Logic Behind a Custom FSM Orchestration Engine"
date: 2026-04-03
draft: false
tags: ["AI Agent", "FSM", "Orchestration", "LangChain", "System Design"]
categories: ["Architecture Decisions"]
summary: "LangChain, LangGraph, CrewAI, PydanticAI — no shortage of AI orchestration frameworks. I evaluated all of them and built my own. Not NIH syndrome. When you need failure-mode-driven mutation strategies, phase-aware multi-model routing with different temperatures, and adaptive evolution based on trajectory analysis, the abstraction layers of general-purpose frameworks become obstacles to route around."
---

> **LangChain, LangGraph, CrewAI, PydanticAI — no shortage of AI orchestration frameworks. I evaluated all of them and built my own. Not NIH syndrome. When you need failure-mode-driven mutation strategies, phase-aware multi-model routing with different temperatures, and adaptive evolution based on trajectory analysis, the abstraction layers of general-purpose frameworks become obstacles to route around.**

## Too Many Frameworks to Justify Writing Your Own

The AI orchestration ecosystem in 2024-2025 has the highest framework density in software engineering history. LangChain has the largest community. LangGraph offers graph-based state machines. CrewAI does multi-agent collaboration. PydanticAI takes structured output to its logical extreme.

A rational engineering decision would be: evaluate these frameworks, pick the closest fit, extend it. Building an orchestration engine from scratch in 2025 looks like reinventing the wheel.

But my scenario has a critical characteristic: the orchestration goal isn't "complete a task" — it's "find the optimal solution through iterative search." AI isn't executing a preset process chain. It's doing evolutionary optimization in a search space — generate code, execute, evaluate, then decide whether to refine, explore, recombine, or simplify based on evaluation results.

That distinction changes everything.

## The Structure, Not the Surface

Break "AI orchestration" apart and it needs to answer questions at four levels:

Level one: state management. What phase are we in? Where can we go? When should we roll back?

Level two: model routing. What model for each phase? What temperature? If a model call fails, where's the fallback?

Level three: iteration strategy. Metrics declining for 3 consecutive rounds — keep refining or change direction? When to try recombining successful segments from history? When to simplify complexity?

Level four: safety mechanisms. What if metrics cliff-dive? What if duplicate code is generated? What if too many rounds pass without convergence?

General-purpose frameworks typically cover level one and the basics of level two. Levels three and four — evolution strategies, disaster rollback, anti-duplication detection — are deeply coupled to the business scenario. No framework will build these for you.

## Frameworks I Evaluated

### LangChain

LangChain's core value is ecosystem integration — unified interfaces for nearly every LLM provider, vector database, and document loader. If your task is "retrieve information from a document store and generate answers" (RAG), LangChain is the right choice.

Its problem in my scenario: too many abstraction layers. A single LLM call passes through Chain → LLM → Prompt Template → Output Parser — four layers. When I need the `generate` phase to use deepseek-reasoner (temperature 0.8) while the `fix` phase uses deepseek-chat (temperature 0.1), I need to bypass LangChain's LLM abstraction to inject phase-aware routing. Bypassing a framework's abstractions is more complex than not using the framework.

A practical issue: debugging. When an AI-generated factor expression fails execution, evaluation metrics come back empty, and the rollback mechanism triggers — I need to know which step broke. In LangChain's call chain, exceptions get wrapped layer by layer, with the real error buried under three levels of traceback. In my own engine, every state transition is explicit `if/elif`, and where an exception occurs is immediately visible.

### LangGraph

LangGraph is closer to my needs — it uses graph structures for state transitions, supporting cycles and conditional branches. If I drew my FSM as a graph, LangGraph could theoretically express it.

But LangGraph's graph state machine differs fundamentally from what I need. LangGraph nodes are "execution functions," edges are "transition conditions." State management and execution logic are intertwined — a node both defines "what phase am I in" and "what does this phase do."

My design deliberately separates the two. The state machine is pure logic — it only answers "can A transition to B," performing zero I/O. A frozenset dictionary, 7 lines:

```
INIT      → {PLAN, FAIL}
PLAN      → {GENERATE_CODE, FAIL}
GENERATE  → {EXECUTE, FAIL}
EXECUTE   → {EVALUATE, ITERATE, FAIL}
EVALUATE  → {ITERATE, FINISH, FAIL}
ITERATE   → {PLAN, GENERATE_CODE, EXECUTE, FAIL}
FINISH    → {}
FAIL      → {}
```

This state machine has no side effects, no dependencies, and can be tested independently. The orchestration loop calls `can_transition()` externally for legality checks, then handles execution itself. State logic decoupled from execution logic means I can replace execution logic without touching state definitions, or use mock execution in tests to verify state flows.

LangGraph's design can't achieve this separation. Its graph is the execution flow, not a pure state definition. For simple linear processes this doesn't matter, but for an iterative loop requiring frequent rollback, jumping, and recovery, a pure state machine is cleaner.

### CrewAI

CrewAI is a multi-agent collaboration framework — multiple Agents with distinct roles cooperating through message passing.

Scenario mismatch. My system is single-Agent multi-phase, not multi-Agent. The first article in this series discussed why I didn't use multi-agent architecture in detail — the core reason being that single Agent + FSM state transitions are deterministic, traceable, and auditable, while multi-Agent message passing is non-deterministic, making behavior chain reproducibility impossible to guarantee.

### PydanticAI

PydanticAI takes structured output to its extreme — defining LLM output format via Pydantic models with automatic validation and retry.

On the single dimension of output validation, it's genuinely more elegant than my custom approach. But it doesn't cover my other dimensions: phase-aware model routing, evolution strategy selection, failure mode detection. If I introduced PydanticAI solely for structured output, my model routing, mutation engine, and trajectory analysis would all need rewriting to adapt to its interfaces. A framework that solves 10% of the problem but demands 60% of the system be refactored to adapt — that trade isn't worth it.

## Three Core Designs of the Custom Engine

### Design One: Phase-Aware Multi-Model Routing

Different orchestration phases have different LLM requirements. Planning needs high creativity (high temperature), code generation needs diversity (medium-high temperature), fixing needs precision (low temperature).

My router uses a three-level fallback strategy:

```
Lookup order: task_type:phase → phase → default

Example: factor_calc:generate → generate → fallback config

Actual config:
  plan:     deepseek-chat,     temp 0.7 (creative planning)
  generate: deepseek-reasoner, temp 0.8 (diverse generation)
  iterate:  deepseek-reasoner, temp 0.9 (exploratory iteration)
  fix:      deepseek-chat,     temp 0.1 (precise fixing)
```

Three-level fallback means: by default all task types share the same phase routing, but specific task types (say, factor research vs. strategy backtesting) can have different models and parameters configured without code changes.

No mainstream framework natively supports this routing granularity. LangChain has Router Chain, but its routing is based on semantic matching of input content, not deterministic routing based on orchestration phase.

### Design Two: Adaptive Evolution Strategies

This is the engine's core competitive advantage, and the area general-purpose frameworks don't touch at all.

The orchestration loop's iteration isn't simply "tell AI to improve the previous round's code." It's a directional search process, with direction determined by four evolution strategies:

EXPLOIT (refine): current score is decent, fine-tune within current direction. For local optimization.

EXPLORE (explore): current direction isn't working, need a completely different approach. For escaping local optima.

RECOMBINE (recombine): combine successful segments from historical iterations. For late-stage convergence.

SIMPLIFY (simplify): complexity is too high causing overfitting, reduce nested operations while preserving core logic.

Strategy selection isn't random — it's adaptive decision-making based on trajectory analysis:

```
High score + low diversity     → EXPLOIT (keep refining, don't jump)
2+ consecutive declines + enough iterations → RECOMBINE (recover from history)
Low score + early iteration    → EXPLORE (wrong direction, change approach)
High diversity + low convergence → EXPLORE (scattered, need focus)
Medium score + high stability  → EXPLOIT (steady progress)
Large score gap + iteration budget → RECOMBINE (combine strengths)
```

Trajectory analysis operates on four dimensions: exploration diversity (score variance), convergence rate (linear regression slope of scores), stability (recent score consistency), and semantic diversity (AST-based code similarity).

Each strategy maps to different mutation types. Under the EXPLOIT strategy, the mutation engine selects specific operations based on failure mode:

```
Wrong signal direction (IC < -0.01) → flip signal direction
Zero predictive power (|IC| < 0.01) → swap operators
Inconsistent (|ICIR| < 0.3) → add normalization
Over-complex → simplify structure
```

The essence of this system is embedding evolutionary algorithm concepts (selection, mutation, crossover, fitness evaluation) into the AI orchestration loop. AI handles generation and mutation. The engine handles direction selection and fitness evaluation.

### Design Three: Disaster Rollback and Safety Boundaries

Iterative search carries a risk: AI might severely regress in one iteration, producing results far worse than before.

The engine uses three safety layers:

Layer one: best-version tracking. After each evaluation, if the current score exceeds historical best, update the best record (version number, score, code, metrics). If the score declines, increment the consecutive decline counter.

Layer two: disaster rollback. If the current score drops more than 30% from historical best, classify it as catastrophic decline, automatically roll back to the best version's code, reset the decline counter, and restart iteration from the best version. This guarantees that even when AI goes the wrong direction, the system doesn't lose the best solution found so far.

Layer three: early stopping. 3 consecutive declines with 5+ iterations completed and best score above 0.15 — classify as converged, stop iterating, return the best version's results. This prevents wasting compute in already-converged regions.

Additionally, every generated code undergoes anti-duplication detection — first text normalization for exact matching, then AST semantic similarity for structural duplicate detection (threshold 0.85). If duplication is detected, an explicit prompt demands structurally different code.

## The Key Judgment Call

The turning point in choosing to build custom was a realization: general-purpose frameworks solve "how to call LLMs." I need to solve "how to search for optimal solutions."

These two problems have different complexity centers of gravity. "How to call LLMs" has complexity in the connection layer — adapting different APIs, handling retries and rate limits, managing context windows. Frameworks have clear advantages here.

"How to search for optimal solutions" has complexity in the decision layer — selecting strategies based on historical trajectories, choosing mutations based on failure modes, deciding rollback vs. continue based on score trends. This decision logic is deeply coupled to the business scenario and can't be abstracted away by general-purpose frameworks.

If I had used LangGraph, I'd get a graph state machine managing INIT → PLAN → GENERATE → EXECUTE → EVALUATE → ITERATE transitions. But evolution strategies, mutation engine, disaster rollback, trajectory analysis — all of these would still need custom implementation, routed around LangGraph's node abstractions. Total code might exceed pure custom implementation, because of the added adaptation layer.

This mirrors a classic trade-off in operating system design: microkernel vs. monolithic kernel. A microkernel (framework) provides minimal core mechanisms, with functionality extended via plugins. A monolithic kernel (custom) puts core functionality inside the kernel. When your "plugins" are more complex than the "core," the microkernel's architectural advantage vanishes — you've just added communication overhead between kernel and plugins.

## Results

The custom engine has been running for months, processing hundreds of research tasks with 5-20 iterations each. The adaptive evolution strategy improved factor research convergence speed by roughly 30% over fixed strategies — EXPLOIT refining details in high-score regions, EXPLORE escaping local optima in low-score regions, RECOMBINE recovering from stagnation using historical successes.

Disaster rollback triggered a dozen times, successfully preserving the historical best solution every time. Without it, those tasks would have lost all accumulated optimization gains after a single bad AI iteration.

The entire engine is approximately 2,000 lines of code (state machine 50, orchestration loop 500, evolution strategies 200, mutation engine 200, model routing 150, evaluator 300, other utilities 600). Using LangGraph plus custom plugins, LangGraph's own abstraction code plus adaptation code would likely exceed 1,500 lines, with harder debugging.

Protocol-based DI makes the entire engine testable and swappable. The Kernel client is a Protocol interface — inject a mock for testing, an HTTP client for production. Switching LLM providers requires only routing config changes, zero engine code modifications.

## What This Decision Taught Me

I distilled one design principle from this practice:

> When your core complexity is in decision logic rather than the connection layer, a general-purpose framework's abstraction layer is a cost, not a benefit. The price of building custom is higher initial investment. The return is complete control over decision logic — no translation needed between the framework's abstractions and your business logic.

This principle doesn't say "never use frameworks." If your core complexity is in the connection layer — interfacing with 10 LLM APIs, 5 vector databases, 3 document formats — LangChain's ecosystem integration is genuine value, and building custom is a waste.

The test: do you spend more time "calling LLMs" or "making decisions based on results"? If the former, use a framework. If the latter, frameworks can't help much — they'll just add a layer of indirection between you and your decision logic.

More generally, a framework's value = complexity it abstracts away / adaptation complexity it introduces. Use the framework when the numerator exceeds the denominator. When the denominator exceeds the numerator — as in my scenario — building custom is the more economical choice.

---

*Seventh article in the series. Previous: [If Research Isn't Reproducible, It Isn't Research](/en/posts/atomic-versioning/). Fifth: [Let AI's Code Run — But Don't Let It Run Away](/en/posts/sandbox-defense-in-depth/). Fourth: [Endgame Thinking](/en/posts/design-for-the-endgame/). Third: [AI as Operator, Kernel as Law](/en/posts/ai-as-operator/). Second: [MCP's Problem Isn't the Protocol](/en/posts/mcp-semantic-gap/). First: [Why I Didn't Use Multi-Agent Architecture](/en/posts/why-not-multi-agent/).*
