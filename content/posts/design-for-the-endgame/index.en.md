---
title: "Endgame Thinking: Design for the Audit Before You Design the Feature"
date: 2026-04-02
draft: false
tags: ["System Design", "Auditability", "AI Agent", "Architecture"]
categories: ["Architecture Decisions"]
summary: "Most systems are designed to run first, then audited as an afterthought. I inverted the order — first define what questions the system must answer when things go wrong, then work backwards to what each layer must record. This inversion reshaped the entire architecture."
---

> **Most systems are designed to run first, then audited as an afterthought. I inverted the order — first define what questions the system must answer when things go wrong, then work backwards to what each layer must record. This inversion reshaped the entire architecture.**

## An Inverted Design Order

The natural order of building a system: make the feature work, add logging, then bolt on audit trails.

When designing an AI orchestration system, I flipped this: first list the questions the system must answer after something goes wrong, then work backwards to what each layer should record, and only then design how the feature runs.

This wasn't because I'm more "disciplined." It was because of an early failure — AI auto-iterated through 15 optimization rounds and produced a strategy with an impressive Sharpe Ratio. But when I tried to trace back "why did round 8 switch from momentum to mean reversion," there was nothing. AI made a decision, but the decision process had vanished.

An unexplainable good result is more dangerous than an explainable bad one. Because you don't know if it's genuinely good or just overfit.

## The Structure, Not the Surface

The core issue isn't "not enough logs." It's a more fundamental design flaw: the system's information flow was designed for execution, not for retrospection.

Execution-first systems look like this:

```
Input → Process → Output → (optional) log something
```

Retrospection-first systems look like this:

```
Input → Record input → Process → Record decision basis → Output → Record output → Link to one chain
```

The difference isn't how much you record. It's whether what you record can be threaded into a causal chain. Scattered log entries aren't an audit — they're noise. An audit answers "who, based on what information, made what decision, when, and what happened" as a complete chain.

This shares DNA with database transaction log design. A database's WAL (Write-Ahead Log) isn't a debugging tool bolted on after the fact — it's part of the architecture. Log first, then execute. The order cannot be reversed.

## Approaches I Considered

### Approach A: Retroactive Logging

Add logger.info calls throughout the existing code. Capture state at key points. grep logs when you need to trace something.

This is standard practice in ops troubleshooting. Something breaks, check the logs, find the timestamp, locate the error.

Why it fails for AI orchestration: AI's orchestration loop isn't linear. It's iterative — generate code → execute → evaluate → decide to continue or stop → back to generate. Each round's "decision" depends on the previous round's "evaluation," which depends on "execution results." With scattered log lines, you can see what happened at each step, but not the causal relationship between steps. "Why did round 8 change direction" — that answer is distributed across three different log lines with nothing linking them together.

### Approach B: Endgame-Backwards Design (My Choice)

Define "what questions must be answerable after the fact," then work backwards to what the system must record.

I listed five endgame questions:

1. Who initiated this task, in what context?
2. What code was generated in each iteration, with what metrics?
3. What was the evaluation decision in each round, based on what reasoning?
4. What version of data did the final result depend on?
5. If you rerun with the same inputs, do you get the same result?

Working backwards from these five questions, each maps to a required data structure:

- Question 1 → submission record (user ID, session, objective, timestamp)
- Question 2 → iteration snapshot (complete code + execution result + metrics per round, not summaries)
- Question 3 → evaluation record (decision + reasoning as structured fields, not log text)
- Question 4 → data version hash (SHA256 of input data, stored in task metadata)
- Question 5 → reproducibility check (code hash + data hash + engine version — the triple uniquely determines the result)

After this design, the information flow changed: each state transition isn't "execute then maybe record something." It's "recording is part of the state transition — without the record, the transition isn't complete."

## The Key Judgment Call

The turning point wasn't technical. It was a cognitive shift: audit isn't a system's add-on feature — it's a design constraint.

This recognition came from a financial industry convention. In compliance-heavy financial institutions, trading system audit requirements aren't "requested" by the compliance team after launch — they're part of the architecture from day one. Trade record completeness, timestamp immutability, decision chain traceability — these are system requirements equal in priority to "can place orders."

AI orchestration systems face the same problem. When AI automatically makes decisions that affect capital, "how was this decision made" isn't a debugging need — it's a production need.

This leads to a more general principle: any system that will be asked "why" should treat "answering why" as a design constraint from the start. This isn't solvable by adding logs — the information architecture must be designed for retrospection.

## Results

Endgame-backwards design had an unexpected benefit: it dramatically simplified debugging.

The old debugging flow: read logs → grep keywords → piece together a timeline → guess at causation.

The new flow: query task ID → get the complete iteration sequence → every step's input, output, decision, and reasoning are structured fields → directly locate the problematic step.

From "searching through unstructured text" to "querying structured data." This isn't a side effect of the audit system — it's the natural consequence of endgame thinking: data structures designed for retrospection are inherently queryable.

A subtler effect: when the system forces every evaluation decision to record its reasoning, the evaluation logic itself is forced to become more explicit. "Pass/fail" isn't enough — you must write "terminated because Sharpe < 1.0 and 3 consecutive rounds without improvement." Mandatory recording forces decision logic to crystallize.

## What This Decision Taught Me

I distilled one design principle from this practice:

> When designing a system, ask first: after this system fails, what questions must it answer? Then ensure the architecture itself can answer them — without depending on post-hoc log grepping.

This principle extends far beyond AI systems. Any system that runs long enough will eventually face the question "how did it end up like this." The difference: some architectures can directly answer that question. Others require an engineer to spend three days sifting through logs to assemble an uncertain answer.

The difference isn't who has more logs. It's who treated "traceable" as a design constraint equal to "functional" during the design phase.

The essence of endgame thinking: don't just design what the system looks like when it's running normally. Also design what it looks like when it's being examined. The latter often determines the architecture of the former.

---

*Fourth article in the series. Previous: [AI as Operator, Kernel as Law](/en/posts/ai-as-operator/). Second: [MCP's Problem Isn't the Protocol](/en/posts/mcp-semantic-gap/). First: [Why I Didn't Use Multi-Agent Architecture](/en/posts/why-not-multi-agent/).*
