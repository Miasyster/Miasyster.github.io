---
title: "MCP's Problem Isn't the Protocol — It's the Semantic Gap"
date: 2026-04-02
draft: false
tags: ["MCP", "AI Agent", "Vibe Coding", "Intent Validation"]
categories: ["Architecture Decisions"]
summary: "MCP's JSON-RPC transport works fine. The real problem: natural language rules have no code-level enforcement — the LLM can completely ignore your instructions. I designed the Intent Validator pattern to close this gap."
---

> **MCP's JSON-RPC transport works fine. The real problem: natural language rules have no code-level enforcement — the LLM can completely ignore your instructions. I designed the Intent Validator pattern to close this gap.**

## A Rule That Got Ignored

My system has a business rule: ML backtests must use the full data range, date restrictions are forbidden. The reason is that restricting the range reduces sample size, making results unreliable.

This rule was written in the MCP server's instructions, telling the agent in natural language: "NEVER restrict to test period only."

Then one day the agent ignored it. It filled in a date range, the request went through, the task completed, and the results looked normal. No errors anywhere. But the results were unreliable — nobody just knew.

This isn't an MCP bug. The protocol layer worked correctly. The problem is more fundamental: **across the entire MCP ecosystem, there is no mandatory binding between natural language rules and code execution.**

## The Structure of the Problem

Current AI tool-calling validation has three layers. The middle one is empty:

```
Natural language instructions (MCP instructions) → LLM "understands" → might comply, might not
            ↓
         ??? (empty)                              → no code-level check
            ↓
Type validation (JSON Schema / Pydantic)          → start_date is a string → type valid → passes
```

Layer 1 is advice. Layer 3 is type checking. The missing middle is **code-level enforcement of business semantics.**

This gap isn't unique to my system. Every MCP or function-calling application has it. MCP's tool schema can define parameter types, but can't express conditional constraints between parameters — "if A is empty then B must be non-empty," "when task type is X, field Y is forbidden." JSON Schema can't describe these.

The industry currently attacks this from both ends:

- Upstream: optimize prompts so the LLM better understands rules → has a ceiling, will never be 100% reliable
- Downstream: tighten JSON Schema with stricter type definitions → insufficient expressiveness for cross-parameter constraints

Both ends are being worked on. Nobody's building the middle layer.

## How I Got Here

My first instinct was to build a custom protocol to replace MCP. After analysis, I realized this was the wrong reaction — the protocol layer (JSON-RPC transport, tool discovery, serialization) isn't broken. Swapping protocols doesn't fix a semantic problem; it just moves the complexity.

Then I recognized the structural similarity to input validation in web development. Frontend forms have HTML5's type="email" validation (analogous to JSON Schema type checking), but real business validation ("email domain must be company domain," "amount can't exceed balance") happens on the backend. Nobody says "HTML5 validation is insufficient, I need to build a new HTTP protocol."

The correct approach is **adding an application-level business rule validation layer between LLM output and system execution.**

That was the design starting point for Intent Validator.

## Approaches Compared

### Approach A: Strengthen Prompt Instructions

Write rules more explicitly, add more WARNING markers, emphasize consequences. This is what most MCP applications currently do.

The problem: you're fundamentally gambling against probabilities. An LLM isn't a rule engine; it's a probability model. "Usually complies" is not "always complies." Occasional failures are acceptable in research experiments. In production, they're not.

### Approach B: Tighten Tool Schema

Remove start_date from the exposed fields entirely. But this means other task types that legitimately need dates can't use them either — a tool's schema is the union of all calling scenarios, not the intersection.

### Approach C: Intent Validator (My Choice)

Insert a code-level validation layer after LLM output is parsed, before it's sent to the execution layer. Each task type registers its own business rules; the validator runs them automatically.

Core design principles:

**Rules are code, not documentation.** "ml_backtest can't set dates" isn't a note in the README. It's a Python function that raises an error.

**Registry architecture.** Adding rules doesn't change framework code — just add a decorated function. Going from 0 rules to 100 rules requires zero changes to the validation engine.

**Actionable rejection messages.** A rejection isn't a bare "400 Bad Request." Each violation carries three fields: rule (machine-readable ID), error (what's wrong), fix (specific correction instructions). The agent can self-correct and resubmit without human intervention.

**Hard blocks and soft warnings, separated.** Some rules are mandatory (hard errors reject the submission). Some are advisory (warnings attached to the success response). Not everything is binary.

## The Key Judgment Call

The most critical design decision was: **where should this validation layer live?**

Option 1: Inside the MCP server. Covers only the agent calling path.

Option 2: Inside the REST API request models. Covers only the HTTP calling path.

Option 3: Extract into an independent module, called by both paths.

I chose option 3. The reasoning: a rule shouldn't be written twice. MCP and HTTP are two entry points into the same system; business rules don't change based on which door you walk through. Implementation: a model_validator in the request model base class automatically calls the shared rule engine. The MCP server's submit function also calls the same engine. Rules written once, all entry points covered.

The inspiration for this decision came from a corollary of DRY: **duplication in validation logic is more dangerous than duplication in business logic.** If business logic is duplicated, both copies behave the same. If validation logic is duplicated and the two copies drift out of sync, one entry point allows what another rejects — that's a security hole.

## Results

After deployment, I tested several scenarios:

- Agent submits ml_backtest with start_date → instant rejection with clear fix instructions, agent self-corrects and resubmits successfully
- Script curls the REST API directly with illegal parameter combination → Pydantic model_validator triggers the same rules, returns 422
- Adding a new business rule → write one decorated function, change zero existing code

From "rules in instructions, hoping the LLM complies" to "rules in code, non-compliance is an error." Reliability went from probabilistic to deterministic.

## What This Decision Taught Me

I updated a mental framework from this practice:

> **Every critical constraint between natural language and code needs a code-level enforcement point. If a rule exists only in documentation or prompts, it's not a constraint — it's a suggestion.**

This principle extends beyond MCP. It applies to any scenario where AI agents operate under business constraints: drug contraindication rules for medical agents, statute of limitations checks for legal agents, change window restrictions for ops agents. If these constraints rely solely on prompt-level soft control, you're using a probabilistic model to provide deterministic guarantees — that's logically unsound.

The core tension of vibe coding is the gap between natural language ambiguity and system operation precision. Intent Validator isn't the ultimate solution, but it points in the right direction: **don't try to make the LLM more rigorous — put a code-level gate behind it.**

---

*Second article in the series. Previous: [Why I Didn't Use Multi-Agent Architecture](/en/posts/why-not-multi-agent/).*
