---
title: "Let AI's Code Run — But Don't Let It Run Away"
date: 2026-04-03
draft: false
tags: ["Sandbox", "AI Agent", "Security", "System Design"]
categories: ["Architecture Decisions"]
summary: "AI-generated code must be executed — otherwise it's just text. But execution means risk. I didn't choose container isolation or RestrictedPython. Instead I designed a three-layer defense: reject dangerous structures at compile time via AST, replace the entire builtins at runtime, and enforce OS-level resource limits as a backstop. Each layer handles a different class of risk. Overlapping but not redundant."
---

> **AI-generated code must be executed — otherwise it's just text. But execution means risk. I didn't choose container isolation or RestrictedPython. Instead I designed a three-layer defense: reject dangerous structures at compile time via AST, replace the entire builtins at runtime, and enforce OS-level resource limits as a backstop. Each layer handles a different class of risk. Overlapping but not redundant.**

## An Unavoidable Contradiction

The core value of an AI orchestration system: AI generates code, the system executes it, results feed back into the next iteration. If generated code can't run, the loop breaks.

But executing AI-generated code has a fundamentally different risk profile from executing human-written code. A human engineer understands side effects — they know what `import os; os.system('rm -rf /')` means. AI doesn't have that awareness. Its goal is "complete the task." If connecting directly to a database gets data faster, it'll try. Not out of malice — because it doesn't distinguish "legitimate means" from "unauthorized means."

So the question isn't "should we sandbox" but "what should the sandbox defend against, and at which layer."

## The Structure, Not the Surface

Break down "AI code execution risk" and you get three distinct categories:

The first is structural danger — code contains syntactic constructs that shouldn't exist. `import subprocess`, `eval()`, `__import__()`. These are patterns identifiable at parse time. Their danger doesn't depend on runtime context.

The second is runtime privilege escalation — the syntax is fine, but legitimate builtins are used to do illegitimate things. `open('/etc/passwd')` is syntactically valid but semantically unauthorized. Or chaining `getattr()` calls on dunder attributes to escape the sandbox.

The third is resource abuse — logic is correct, permissions are fine, but resource consumption is unacceptable. Infinite loops, allocating a 10GB list, maxing out CPU for 30 minutes. Not a security issue — a resource management issue.

Three categories need three layers. Trying to solve all with one layer either leaves gaps (too permissive) or blocks legitimate code (too restrictive).

## Approaches I Considered

### Approach A: Container Isolation

Docker containers or WebAssembly sandboxes. Spin up an isolated environment per execution, code runs freely inside, destroy it when done.

This is standard practice for multi-tenant SaaS. Replit, CodeSandbox, every online IDE uses this. Maximum security — OS-level isolation, code can't affect the host regardless of what it does.

Why it doesn't fit an AI iteration loop: latency. The core cycle is "generate → execute → evaluate → iterate," and a research task might run 10-20 iterations. Each iteration: start container, load pandas and numpy, initialize data context, execute, extract results, destroy container. Cold start overhead is 2-5 seconds. Multiply by 20 and that's 40-100 extra seconds. For a system that needs fast iteration, that latency is unacceptable.

There's also a practical issue: data transfer. Code inside the container needs market data, but the data can't be copied in (too large) and the container shouldn't connect directly to the database (violates architecture principles). You'd need a serialize → transfer → deserialize pipeline, which itself introduces complexity and performance cost.

### Approach B: RestrictedPython

The Python community has a mature solution called RestrictedPython. It rewrites Python's compilation at the bytecode level, replacing all attribute access and function calls with interceptable proxy functions. Security sits between "bare exec" and "containers."

Why I didn't use it: RestrictedPython was designed for "running untrusted user code in a multi-tenant environment" — a relic of the Zope/Plone CMS era. Its security model is extremely strict. Strict enough that many pandas and numpy operations get intercepted. `df.groupby()` triggers attribute access interception. `np.array()`'s internal C extension calls bypass the Python-level restrictions. Making RestrictedPython coexist with data science libraries requires extensive whitelisting and monkey-patching.

This is fundamentally a use-case mismatch. RestrictedPython assumes code from untrusted external users. My scenario has code from a controlled AI model — risk exists but is predictable. I don't need bytecode-level comprehensive interception, just rejection of known dangerous patterns and resource caps.

### Approach C: Three-Layer Defense — AST Scan + Runtime Whitelist + Resource Limits (My Choice)

Design philosophy in one sentence: each layer handles one class of risk, layers are orthogonal, no layer tries to solve everything.

## The Three Layers in Detail

Layer one: AST static scanning. Before code executes, parse the source into an abstract syntax tree, walk every node, check for forbidden structures.

```
Forbidden imports:  os, sys, subprocess, socket, shutil, ctypes, importlib...
Forbidden calls:    eval(), exec(), compile(), __import__(), open(), globals()...
Forbidden attrs:    __subclasses__, __bases__, __globals__, __code__
```

The key property of AST scanning is determinism — same code, same result, always. No runtime state dependency, no input data sensitivity. If code contains `import os`, regardless of variable values, the AST scan rejects it.

This layer addresses the first risk category: structural danger. It's a compile-time "preflight check" — finding engine trouble before takeoff is far cheaper than finding it mid-flight.

Layer two: runtime builtins replacement. Not blacklist filtering on Python's default builtins — a complete replacement.

Default Python builtins include 150+ functions and types. My whitelist keeps 53: math operations (abs, min, max, sum, round), type constructors (int, float, str, dict, list), iteration (enumerate, filter, map, zip, range), safe reflection (isinstance, len, type).

Key exclusions: `open()`, `getattr()`, `setattr()`, `delattr()`, `__import__()`. These are classic Python sandbox escape paths — chaining `getattr` to access dunder attributes lets you climb from any object all the way to `os.system`.

Safe scientific computing libraries are pre-injected. When code executes, the global namespace already contains `pd` (pandas) and `np` (numpy), no import needed. This avoids the dilemma of "opening import permissions just so AI can use pandas."

Layer three: OS-level resource limits. Using POSIX `resource` module to set hard caps:

```
Memory:     2048 MB (RLIMIT_AS)
CPU:        120 seconds (RLIMIT_CPU)
Wall clock: 300 seconds (monotonic clock)
Output:     50 MB
```

Memory and CPU limits are kernel-enforced. Code allocates more than 2GB, the kernel kills the process — not a Python-level MemoryError, a SIGKILL. This guarantees that even if both previous layers are bypassed, resource abuse can't affect the host.

Wall clock timeout uses `time.monotonic()` instead of system clock, because monotonic isn't affected by NTP time adjustments — more reliable for timing.

## The Key Judgment Call

The core judgment behind this design: the code source is a controlled AI model, not an untrusted external user.

This judgment shifts the security model's center of gravity. Against untrusted code, you must assume attackers actively seek escape paths — bytecode injection, C extension vulnerabilities, race conditions. Against AI-generated code, the primary risk is "unintentional overreach" not "deliberate attack." AI won't deliberately construct `"".__class__.__bases__[0].__subclasses__()` to escape a sandbox, but it might try `import os` to read files, because that's a common pattern in its training data.

This means the defense should focus on "rejecting known dangerous patterns" (AST scanning) and "limiting the available toolset" (whitelist), rather than "defending against unknown escape vectors" (container isolation). The former is lighter, faster, and less disruptive to legitimate operations.

The code includes a comment that makes this positioning explicit: "Uses AST scanning + restricted globals to sandbox exec() calls. In production this should be replaced with container-based isolation for untrusted input." Acknowledges limitations while clarifying appropriateness for the current scenario.

This judgment also draws from a cross-domain analogy: airport security. Airport security doesn't examine every cell of every passenger — it uses metal detectors (AST scanning), a prohibited items list (builtins whitelist), and capacity limits (resource limits). Each layer has blind spots, but combined they cover the vast majority of real threats. If you wanted to transport each passenger in an isolation capsule (container isolation), security would indeed be higher, but no flight would ever depart on time.

## Results

The three-layer defense has been running for months, processing thousands of AI-generated code executions. AST scan rejection rate is roughly 5% — mostly AI attempting to import system modules. Runtime whitelist intercepted zero privilege escalations — because the AST layer already filtered out most dangerous code, the whitelist serves as backstop. Resource limits triggered dozens of times — mainly AI-generated code with inefficient loops exceeding CPU time limits.

Per-execution sandbox overhead is in milliseconds (AST parse + globals construction), three orders of magnitude faster than container-based cold starts measured in seconds.

## What This Decision Taught Me

I distilled one design principle from this practice:

> Security defense should be layered by risk category, not stacked into one layer that tries to handle everything. Each layer only needs to handle the risk class it's good at. Overlap between layers is a feature, not a bug.

This principle extends well beyond code sandboxes. Network security's "defense in depth" is the same idea — firewalls, intrusion detection, application-layer filtering each handle one layer. Database security follows the same pattern — connection encryption, SQL injection filtering, row-level permissions each handle one layer.

The key insight: each additional layer has diminishing returns. The first layer (AST scanning) blocks 95% of risk at minimal cost. The second (whitelist) blocks 4% at moderate cost. The third (resource limits) blocks 1% at highest cost. Adding a fourth layer (container isolation) to block the remaining 0.1% might cost more than the first three combined.

When designing security systems, ask first: how much is the residual risk worth? If the answer is "not worth another layer," the current defense is sufficient.

---

*Fifth article in the series. Previous: [Endgame Thinking: Design for the Audit Before You Design the Feature](/en/posts/design-for-the-endgame/). Third: [AI as Operator, Kernel as Law](/en/posts/ai-as-operator/). Second: [MCP's Problem Isn't the Protocol](/en/posts/mcp-semantic-gap/). First: [Why I Didn't Use Multi-Agent Architecture](/en/posts/why-not-multi-agent/).*
