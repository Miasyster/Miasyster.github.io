---
title: "Cross-Model Review Is Architecture, Not Heuristic"
date: 2026-05-08
draft: false
tags: ["AI Agent", "LLM", "DeepSeek", "Architecture", "Quant Research", "MCP"]
categories: ["Architecture Decisions"]
summary: "Single-LLM self-reflection has a structural blind spot — the model tends to confirm its own output. QuantGPT enforces a hard rule in factor-mine SKILL Phase 0.5: Claude must consult DeepSeek before designing a new factor family. Not a suggestion, a hard rule. This isn't redundancy — it's the antidote to structural bias."
---

> **Single-LLM self-reflection has a structural blind spot — the model tends to confirm its own output. QuantGPT enforces a hard rule in factor-mine SKILL Phase 0.5: Claude must consult DeepSeek before designing a new factor family. Not a suggestion, a hard rule. This isn't redundancy — it's the antidote to structural bias.**

## 1. The Structural Blind Spot of Single-LLM Self-Reflection

Letting Claude reflect on its own output is a standard pattern in LLM applications. Reflection, Self-Critique, Chain-of-Thought — different names, same essence: **let the model evaluate itself with its own capabilities**.

This pattern works on small tasks. But on domain-deep tasks it has a fundamental limitation: **a model's reflection still falls within its own training distribution**.

Concrete scenario. Claude designs a factor expression:

```
-1 * rank(ts_av_diff(close, 10)) + rank(debt / enterprise_value)
```

Now you ask Claude to review this expression. It will output something like: "This factor combines price reversal with a fundamental signal, theoretically reasonable, aligned with WQ BRAIN style."

Sounds right. But this is the same model evaluating the same model's output — **the cognitive toolkit it uses is the one that produced the original**. If the original design has a bias (e.g., overfitting to a specific factor structure), the reflection carries that same bias.

Worse: RLHF dialogue training pushes models toward "completing the task". Asked to "reflect on the factor I just designed", a model leans toward supporting itself — because rejecting itself means rewriting, extending the conversation, and delaying delivery.

**This is a structural problem, not something prompt engineering fixes.**

## 2. Cross-Model Review ≠ "AI Checking AI"

The first instinct is naive: just have another LLM check it. But **not any LLM**.

The effectiveness of cross-model review comes from three differences:

1. **Training data distribution** — different models train on different corpora, with different blind spots and strengths
2. **RLHF trajectory** — different human-feedback data shapes different judgments of "what counts as correct"
3. **Reasoning style** — different chain-of-thought tendencies

If you use GPT-4o to review Claude's output: both are English-dominant, both aligned to general helpfulness, both with similar reasoning styles. The differences aren't large enough; the complementary value is limited.

Engineering cross-model review requires **picking a model with real distributional differences and stronger capability in the target domain**. For quant factor research, that model is **DeepSeek**.

## 3. Why DeepSeek, Specifically

I've used DeepSeek for review for several months. Here's why **it's the optimal choice for the quant scenario right now** — not because it's cheap, but because the distribution aligns.

### It Comes From a Quant Firm

DeepSeek's parent company is **High-Flyer Quantitative Investment** — one of China's tens-of-billions-RMB AUM quant hedge funds.

This isn't just a corporate-lineage label. It means:

- The team has first-hand understanding of quant research workflow
- The training corpus naturally contains a high proportion of finance, statistics, and derivatives texts
- Training density on financial mathematics, factor analysis, and backtest semantics is far higher than general-purpose models

Having Claude's factor expression reviewed by **a model trained by a team that does quant for a living** is distributional alignment, not a gimmick.

### The Best Choice for Chinese Financial Reasoning

Claude's training data is English-dominant. It can handle phrases like "CSI 500 industry-neutralized", but not as naturally as it handles "S&P 500 sector neutralized".

DeepSeek's density on Chinese financial corpora is far higher than general-purpose models. When factor design touches A-share market structure (price-limit rules, ST/halt status, two-sided market behavior, Wind industry classification), DeepSeek's feedback is often more concrete and accurate.

This isn't a performance difference (both have strong reasoning). It's a **domain grounding** difference.

### R1 Exposes Reasoning Traces

DeepSeek-R1 exposes a `reasoning_content` field — you see the model's complete chain-of-thought:

```python
result = ask_deepseek(prompt, model="deepseek-reasoner")
print(result["content"])     # final answer
print(result["reasoning"])   # full reasoning trace
```

For review scenarios, **the reasoning trace matters more than the answer**. The reasoning path explaining "why this factor might overfit" is 10x more useful than a one-liner saying "consider simplifying" — the former teaches Claude something it can act on, the latter is noise.

OpenAI o1 hides reasoning traces behind the product. DeepSeek exposes them directly. For developers this is a quality difference; for autonomous Agent systems this is a **usability difference**.

### Reasoning Depth Comparable to o1, Pricing an Order of Magnitude Lower

DeepSeek-R1 benchmarks comparably to OpenAI o1 on AIME / MATH-500 / GPQA reasoning tests.

Pricing comparison (early 2026):

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|------------------------|
| DeepSeek-R1 | ~$0.55 | ~$2.19 |
| OpenAI o1 | ~$15 | ~$60 |

The gap is about **27x**.

Each Phase 0.5 review consumes around 5K-10K tokens, costing ~$0.003-0.007 per call. **Cheap enough to make "review" the default behavior, not a luxury**. The price isn't the cause — but it's what makes "mandatory review" a viable engineering design.

## 4. The Implementation in QuantGPT

QuantGPT enforces DeepSeek consultation in factor-mine SKILL's `Phase 0.5: DeepSeek Design Consultation`.

### Trigger Conditions

```
- Starting a new research direction
- Existing signal family has reached SC saturation, needs new structure
- No reusable expression template found in the knowledge base
```

Any one of these triggers it. **Cannot be skipped within the SKILL flow.**

### MCP Tool Implementation

The DeepSeek MCP server is a 196-line stdio JSON-RPC service (`scripts/mcp_deepseek.py`). Core definition:

```python
TOOLS = [{
    "name": "ask_deepseek",
    "description": (
        "Send a prompt to DeepSeek LLM. "
        "Use for: Chinese financial reasoning, factor expression generation, "
        "alternative perspectives, or tasks benefiting from "
        "DeepSeek Reasoner's chain-of-thought."
    ),
    "inputSchema": {
        "properties": {
            "prompt": {"type": "string"},
            "model": {"type": "string", "default": "deepseek-reasoner"},
            "system": {"type": "string"},
            "temperature": {"type": "number", "default": 0.7},
        },
    }
}]
```

Registered to Claude Code via `.mcp.json`. Claude Agent sees the tool and can invoke it.

### Engineering the Review Prompt

Not a simple "what do you think of this factor". Phase 0.5 feeds DS a **structured payload**:

```
Facts layer:
  - Current research direction (from research_notes/archive/)
  - Verified rules (from knowledge/rules/)
  - Falsified paths (from knowledge/failures/)
  - Claude's draft design + reasoning

Request layer:
  - Evaluate whether this design has unidentified risks
  - Suggest 1-2 alternative structures
  - Flag potential conflicts with known failures
```

DS returns reasoning_content + content. Claude must **explicitly accept/reject/adjust**, and write the decision into research notes. This isn't a discussion — it's part of the engineering pipeline.

## 5. Why "Mandatory", Not "Suggested"

Making cross-model review optional doesn't work. Two reasons.

### Agents Skip Non-Required Steps

LLM dialogue training pushes Agents toward the shortest path to task completion. "Suggest you ask DeepSeek" written a thousand times in the prompt — when the Agent is racing to finish, it'll skip. Because one more API call, 30s of waiting, and the writeback is all latency.

### Prompt-Level Constraints Have No Enforcement

I argued in detail in [Harness Is Governance](/en/posts/harness-is-governance/): violating a prompt produces no consequences. The Agent skips a "suggestion", the flow continues, and there's no feedback signal telling it that this was wrong.

### Solution: Hard Rules + Flow Dependency

QuantGPT writes Phase 0.5 as a hard rule in the SKILL. Not a "suggestion" in a prompt — a flow dependency. The subsequent Phase 1 prompt template **explicitly references the DS review result** as context. Skip 0.5, and Phase 1's context is missing, the flow fails.

This is Harness Is Governance applied: **constrain Agents with code, not prompts**.

## 6. Trade-offs and Boundaries

Cross-model review isn't a free lunch.

### Cost

- Per Phase 0.5: ~$0.003-0.007 + 30s latency
- One full research cycle (4-8 Phase 0.5 calls): ~$0.03-0.05
- About 30%-50% higher token cost vs. a pure single-LLM pipeline

### Both Models Failing Together

If a bias exists in **both** Claude's and DeepSeek's training distributions (e.g., a shared interpretation of certain classical factors as overfitted), both models pass it together — and the review fails.

This "distribution overlap" blind zone has no perfect solution. Mitigations:

- Add a third model (GPT-4o / Qwen / Llama 3) as arbiter
- When review pass rates exceed 90% over time, inject adversarial prompts ("assume this factor is overfitted, find evidence")
- Maintain a hard-coded blacklist for historically falsified factor structures (not relying on any LLM judgment)

### When It Doesn't Apply

- Task domain has only one model with real knowledge (e.g., niche medical database) — cross-model review has no signal
- Latency-sensitive task (millisecond range) — extra API call is a deal breaker
- Team already has human review process — adding LLM review is redundant

## Conclusion

Cross-model review is an **architecture-level** design choice, not a prompt asking the model to "think more carefully".

Single-LLM self-reflection's limit comes from training distribution — and training distribution can only be cut by **another distribution**. When the review target is quant factors, that "other distribution" has essentially one choice: DeepSeek. It comes from a quant firm, trains on heavy Chinese financial corpora, reasons at o1 depth, and costs an order of magnitude less — cheap enough to make it default behavior.

QuantGPT enforces Phase 0.5 not because I believe DeepSeek is always right — it's wrong sometimes. It's because **the probability of two models with real distributional difference being wrong simultaneously is significantly lower than one model being wrong on its own**. Architecture's job isn't to eliminate errors, it's to reduce error rates to acceptable levels.

**Cross-Model Review Is Architecture, Not Heuristic.**
