---
title: "AI as Operator, Kernel as Law — Why AI Shouldn't Have Architectural Authority"
date: 2026-04-02
draft: false
tags: ["AI Agent", "Architecture", "System Design", "Separation of Concerns"]
categories: ["Architecture Decisions"]
summary: "Letting AI drive research workflows doesn't mean letting AI decide how the system runs. I made a key separation: AI is just the operator, the execution engine is the law. This decision came from a failure."
---

## A Failure That Clarified Things

In the early days of the system, I gave AI a lot of freedom — it could connect directly to databases, generate scripts in any directory, bypass APIs to instantiate internal components and run tasks.

The result: AI was indeed more "efficient." But the cost — some task results existed only in the process AI spawned, lost when the script exited. Temporary scripts scattered across the root directory with no one knowing if they were still needed. Some operations bypassed the audit trail, making it impossible to trace who did what after the fact.

The system "worked," but couldn't be trusted.

This forced a fundamental question: in an AI-driven system, where should AI's authority boundary be drawn?

## The Structure, Not the Surface

This isn't about "AI isn't capable enough" or "AI is unreliable." It's a classic system design problem: separating the mutable from the immutable.

Every system has two types of components:

- Immutable infrastructure: data access, execution engine, audit records, resource control. These are the system's "laws of physics" — they don't change based on whether the user is human or AI.
- Mutable operations layer: research strategies, parameter choices, iteration decisions. These are "operator judgment" — should be flexible, meant to vary.

The problem: without an explicit boundary, AI naturally blends the two. It doesn't distinguish between "I'm making an operational decision" and "I'm bypassing infrastructure." To AI, connecting directly to a database and calling an API are both just "means to complete the task."

This is the same problem as human engineers taking shortcuts around process. The difference is that human engineers usually know they're "bending the rules." AI doesn't.

## Approaches I Considered

### Approach A: Permission Checklist

Give AI a detailed "can do / cannot do" list, written in prompts or config files. Similar to Claude Code's CLAUDE.md rules.

Where this works: few rules, low usage frequency, controllable failure consequences. Like letting AI write code but not execute it — one rule, simple enough.

Why it's insufficient here: my system has dozens of task types, each with different execution constraints. The checklist would expand to a point where AI can't reliably follow it. As discussed in the previous article — natural language rules have no enforcement power. More rules means higher probability of being ignored.

### Approach B: Sandbox Isolation

Put AI in a restricted sandbox — can only access specific files, call specific functions, everything executes in an isolated environment.

This is standard practice in security. Docker containers, WebAssembly sandboxes, browser same-origin policy — all the same idea.

Why I didn't fully adopt it: sandboxes solve security problems, not architecture problems. Even inside a sandbox, AI can still write code that bypasses audit trails and generates untraceable results. A sandbox can restrict what resources AI accesses, but not how AI organizes its output.

### Approach C: Three-Layer Separation + API as the Only Channel (My Choice)

The design philosophy in three sentences:

```
Kernel is law.   — The execution engine defines what the system can do and how
AI is operator.  — AI can only invoke the engine through APIs, never touch the internals
UI is display.   — The presentation layer is read-only, never triggers execution
```

The core constraint isn't "AI can't do X." It's "everyone (including AI) can only do things through the same channel." That channel is the execution engine's API.

## The Key Judgment Call

The pivotal insight came from an operating system analogy.

In an OS, user-space programs can't directly manipulate hardware — they must go through system calls (syscalls). Not because user programs are "untrustworthy," but because direct hardware access breaks resource management consistency. Whether you're root or a regular user, disk operations go through the filesystem, network operations go through the protocol stack.

My system adopted the same pattern: the execution engine is the "kernel," AI is the "user-space program." AI submits tasks through the API (analogous to syscalls). The API handles audit records, resource control, and result persistence internally. AI can't bypass this layer, just as user programs can't bypass syscalls to write directly to disk.

This analogy has a corollary: the kernel should exist independently of its users. Even if the AI layer is completely removed, the execution engine still runs — humans can call the same APIs directly. The system doesn't depend on AI to function. AI is just a more efficient way to operate it.

This matters. Many AI systems are designed with "AI at the center, other components serving AI." My design inverts this: the execution engine is at the center, AI is one way to access it. This means:

- AI goes down, the system doesn't. Humans can take over.
- AI gets swapped (GPT to Claude, Claude to a local model), the execution engine needs zero changes.
- The audit trail doesn't depend on AI's "honesty" — because all operations must go through the API, and the API records automatically.

## Three Concrete Constraints

From this philosophy, three inviolable constraints:

First, AI never connects directly to data engines. All data access goes through the execution engine's data API. AI doesn't know where data is stored or in what format — it only knows "give me prices for these symbols in this time range."

Second, AI-generated code executes in a controlled sandbox. Code first passes AST scanning (forbids importing internal modules), then runs under resource limits (CPU, memory, timeout), and output must conform to a standard format. Not "asking AI to be careful" — making unsafe behavior structurally impossible at the code level.

Third, all tasks must be submitted through the API. AI cannot locally instantiate execution components. This ensures every task has an audit record, appears in task history, and can be queried through the interface. No "shadow tasks."

## Results

This separation has been running for months. There's one intuitive way to verify it: I can shut down the AI orchestration layer at any time and the rest of the system is completely unaffected. The execution engine keeps running, the interface keeps displaying data, existing task results aren't lost. AI simply stops "proactively initiating new research."

The reverse — if AI and the engine were coupled, shutting down AI would stop the entire system. That's the essential difference between "operator" and "infrastructure."

Another result: when switching from one LLM provider to another, changes were entirely contained within the orchestration layer — update routing config, swap API call patterns. Execution engine, data layer, interface layer: zero changes. This validated the "AI is replaceable" design goal.

## What This Decision Taught Me

I distilled one design principle from this practice:

> In any AI-driven system, ask first: if you remove AI, does the system still run? If not, you've coupled AI with infrastructure.

This is a more fundamental question than "AI safety." Not "will AI do something bad," but "does the architecture allow AI to do something bad."

If the architecture is right — AI is just the operator, the execution engine is the law — then AI's "unreliability" stops being a systemic risk. AI's action space is constrained to deterministic channels. Within those channels, behavior is auditable, reversible, and traceable.

This mental framework extends from traditional systems design: we never let any single user bypass the operating system, no matter how "smart" that user is. AI is no different. Its capability shouldn't be a reason to bypass system constraints.

---

*Third article in the series. Previous: [MCP's Problem Isn't the Protocol — It's the Semantic Gap](/en/posts/mcp-semantic-gap/). First: [Why I Didn't Use Multi-Agent Architecture](/en/posts/why-not-multi-agent/).*
