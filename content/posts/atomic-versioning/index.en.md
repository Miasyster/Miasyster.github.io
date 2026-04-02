---
title: "If Research Isn't Reproducible, It Isn't Research"
date: 2026-04-03
draft: false
tags: ["Reproducibility", "Versioning", "ML Research", "System Design"]
categories: ["Architecture Decisions"]
summary: "The most common lie in ML research is 'the results looked great last time.' What code was used last time? What data version? What parameters? Nobody can say. I used filesystem transactions (temp directory → atomic rename) to create immutable snapshots of every iteration, turning 'last time's results' from a memory into a queryable fact."
---

> **The most common lie in ML research is "the results looked great last time." What code was used last time? What data version? What parameters? Nobody can say. I used filesystem transactions (temp directory → atomic rename) to create immutable snapshots of every iteration, turning "last time's results" from a memory into a queryable fact.**

## "Last Time's Results" Is a Ghost

Everyone doing ML research has lived this scenario: two weeks ago you produced a strategy with a Sharpe Ratio of 1.8. Now you want to reproduce it, only to discover the code has changed, data has been updated, parameters are forgotten. You know the result once existed, but you can't prove it.

In an AI orchestration system, this problem is worse. AI auto-iterates 15 rounds, each generating different code, using different parameters, producing different metrics. Round 8 looked great, but round 12 overwrote round 8's code. You don't even know what code round 8 used — because nobody saved the intermediate state.

An irreproducible research result isn't a finding. It's an anecdote.

## The Structure, Not the Surface

The root issue isn't "forgot to save." It's that the system's information model only has "current state," not "historical state."

The traditional research workflow looks like this:

```
Run code → Check results → Modify code → Run again → Overwrite previous results
```

Every iteration is an in-place mutation. There's only one copy of the code file (the latest), one copy of the output (the latest). Want to get back to two iterations ago? Manual Ctrl+Z or dig through Git history.

But Git is a version management tool designed for humans — it assumes humans will manually commit at meaningful checkpoints. AI doesn't. AI runs 15 rounds in a loop with seconds between each. You can't expect AI to pause after each round and write a meaningful commit message.

This leads to a design requirement: versioning should be automatic system behavior, not operator initiative. Every state transition automatically produces a snapshot, without depending on anyone remembering to save.

## Approaches I Considered

### Approach A: Automatic Git Commits

Auto-run `git add && git commit` after each iteration. Use Git's version history to manage iteration state.

Where Git is right: human-driven development workflows, where commit granularity is "one meaningful change" and commit messages convey intent.

Why it doesn't fit AI iteration: three reasons.

First, granularity mismatch. One AI task might produce 20 versions, each containing code, execution results, metrics, and metadata. Git manages file changes, not "complete iteration snapshots." You'd need to scatter code, results, and metrics across different files, then link them via commit hash. Possible, but awkward.

Second, performance. `git add + commit` isn't zero-cost in large repositories. During rapid iteration, frequent Git operations become a bottleneck. Git's locking mechanism also means concurrent tasks block each other.

Third, query capability. "Give me round 8's metrics" — in Git, this requires `git log` to find the commit, then `git show` to extract file contents. Doable, but less intuitive than reading a directory.

### Approach B: Database Storage

Write each round's code, results, and metrics to PostgreSQL. Manage versions via relational tables.

Where databases are right: strong structured query needs, large data volumes, transactional consistency requirements.

Why I didn't fully adopt it: code is text, execution results are JSON, metrics are numbers, HTML reports are large text blobs. Cramming all of these into a relational database means either TEXT columns for large fields (poor query efficiency) or splitting across multiple tables (high join complexity). Databases excel at managing structured metadata but aren't great at storing heterogeneous artifacts.

Another consideration: the filesystem natively supports "open the file and look at it." When debugging, directly running `cat code.py` is far more intuitive than `SELECT code FROM versions WHERE ...`. Debuggability is an underrated requirement in research systems.

### Approach C: Filesystem Transactions + Immutable Snapshots (My Choice)

Design philosophy in one sentence: each iteration is an immutable directory, atomicity guaranteed by filesystem transactions.

## The Atomic Write Design

Each iteration's snapshot is a directory with a fixed file structure:

```
{base_dir}/{task_id}/v{version}/
    ├── code.py         # Complete code for this round
    ├── result.json     # Full execution result
    ├── metrics.json    # Extracted key metrics
    ├── metadata.json   # Auto-generated metadata (timestamp, task ID, version)
    └── report.html     # Optional visualization report
```

The critical design: the write process is atomic. Not file-by-file — that would leave half-written artifacts if the process crashes mid-write. Instead:

```
1. Create a temp directory alongside the target (prefix .v{n}_tmp_)
2. Write all files into the temp directory
3. After all files are written, rename() the temp directory to the official name
4. If any step fails, delete the temp directory
```

`rename()` on POSIX filesystems is atomic — it either succeeds completely or nothing happens. This follows the same logic as database transactions: either commit everything or rollback everything. No intermediate state where "code was saved but metrics were lost."

This pattern comes from database WAL (Write-Ahead Log) design. WAL's core principle: "write the log before writing the data" — if data writing crashes, recovery comes from the log. My pattern: "write to temp directory before renaming" — if file writing crashes, the temp directory gets cleaned up, never polluting official versions.

## The Key Judgment Call

The critical insight behind this design: version snapshots aren't a "developer tool" — they're "system infrastructure."

Many systems treat version management as an add-on feature for "developer convenience." Git, MLflow, Weights & Biases all occupy this position — they're standalone tools that researchers proactively use to track experiments.

My design integrates version snapshots into the system's state transition logic. Not "save a version after iteration completes," but "saving a version is part of iteration completion." If the version isn't saved, the iteration hasn't completed at the system level.

This applies the same framework discussed in a previous article about endgame thinking. Endgame thinking says: retrospection capability isn't bolted on after the fact — it's a design constraint. Versioning is the technical implementation of retrospection capability — if every iteration has a complete immutable snapshot, any "last time's results" can be precisely located and reproduced.

This judgment was also influenced by the Immutable Infrastructure philosophy. In containerized deployments, servers aren't "modified" — they're "replaced." Each version of a server image is immutable — problems are solved by rolling back to the previous image, not patching the current one. Same logic: each research snapshot is immutable — problems are solved by returning to a previous snapshot, not searching for diffs in the current state.

## The Cost and Benefit of Immutability

Immutability means storage grows linearly. Each iteration saves complete code and results, not incremental diffs. 20 iterations × 1MB each = 20MB per task. Thousand tasks = 20GB.

This is a conscious trade-off. Incremental storage (saving only diffs) uses less space, but querying requires rebuilding from version 1 forward — high complexity, high risk of errors. Complete snapshots waste space, but each version is self-contained — reading any version requires only reading one directory, with no dependency on other versions' integrity.

Disk is cheap. Engineers' debugging time is expensive. Simple economics.

Another benefit: automatic metadata injection. Each version's `metadata.json` automatically includes task ID, version number, and UTC timestamp. These fields aren't provided by callers — the storage layer fills them automatically, preventing human (or AI) errors from causing metadata inconsistencies.

## Results

After implementing versioning, "last time's results" stopped being a question requiring memory. Query the task ID, list all version directories, open the corresponding version's `metrics.json`. From "do you remember what parameters were used last time" to "check v8's metadata."

The more important effect is in iteration evaluation: the system can automatically compare current version metrics against the historical best. If metrics decline for 3 consecutive rounds, iteration terminates automatically. This logic depends on every version's metrics being fully preserved — if only "current version" and "previous version" exist, you can't identify trends.

Debugging also became simpler. Before: investigating "why round 12's results are worse than round 8" meant reading logs, comparing code diffs, guessing possible causes. Now: open the v8 and v12 directories, `diff code.py` for code differences, `diff metrics.json` for metric differences. All information is right there, no reconstruction needed.

## What This Decision Taught Me

I distilled one design principle from this practice:

> Any system that needs to answer "what was the previous state" should make state snapshots part of system behavior, not operator responsibility. Automated immutable snapshots eliminate "forgot to save" as a failure mode.

This principle extends far beyond ML research. Configuration management (Terraform's state file), database migrations (migration files' linear history), even document version control (each release is a snapshot, not a diff) all follow the same idea.

The core insight: reproducibility isn't a virtue — it's an architectural constraint. If your system needs to answer "what happened before," the architecture must guarantee "previous state was saved." Relying on operator discipline isn't enough — humans forget, and AI certainly won't save proactively.

---

*Sixth article in the series. Previous: [Let AI's Code Run — But Don't Let It Run Away](/en/posts/sandbox-defense-in-depth/). Fourth: [Endgame Thinking](/en/posts/design-for-the-endgame/). Third: [AI as Operator, Kernel as Law](/en/posts/ai-as-operator/). Second: [MCP's Problem Isn't the Protocol](/en/posts/mcp-semantic-gap/). First: [Why I Didn't Use Multi-Agent Architecture](/en/posts/why-not-multi-agent/).*
