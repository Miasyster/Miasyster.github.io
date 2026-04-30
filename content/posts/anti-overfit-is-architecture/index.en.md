---
title: "Anti-Overfit Is Architecture, Not a Plugin"
date: 2026-04-27
draft: false
tags: ["Anti-Overfit", "Architecture", "Factor Research", "Statistical Testing"]
categories: ["Architecture Decisions"]
summary: "Most backtest systems treat anti-overfit as an optional add-on check — run the backtest, then test for overfitting if you feel like it. QuantGPT builds it into the scoring system and evolution engine: anti-overfit results directly affect factor scores, and the evolution engine reads anti-overfit metrics to decide its next strategy. Factors that haven't proven robustness don't even qualify for iteration."
---

> **Most backtest systems treat anti-overfit as an optional add-on check — run the backtest, then test for overfitting if you feel like it. QuantGPT builds it into the scoring system and evolution engine: anti-overfit results directly affect factor scores, and the evolution engine reads anti-overfit metrics to decide its next strategy. Factors that haven't proven robustness don't even qualify for iteration.**

## The Problem with "Test After Backtest"

Standard workflow: generate factor → backtest → check Sharpe → use if satisfied. Anti-overfit checks? Optional. Run them if you feel the need.

This workflow has a structural flaw: **anti-overfit checking is decoupled from decision-making.** You see a factor with Sharpe 2.5, you've mentally decided to use it, then the anti-overfit test says "possible overfitting" — but you already have cognitive bias, inclined to explain away the result.

A more serious problem: when an LLM Agent iterates autonomously, if anti-overfit is just an optional step, the Agent will tend to skip it — because skipping leads to faster iteration. The Agent's goal is "find high-scoring factors," not "find robust high-scoring factors," unless you enforce it at the architecture level.

## Four-Layer Anti-Overfit Testing

QuantGPT's anti-overfit system isn't a single test but a combination of four independent tests:

### 1. IC Stability

Computes yearly Spearman IC (rank correlation between factor values and future returns), requiring:
- Positive IC ratio ≥ 55%
- |Mean IC| ≥ 0.02
- No yearly IC sign reversal

A factor with positive IC in 2020 but negative IC in 2021 isn't capturing a stable alpha signal — it's capturing a coincidental association specific to a market environment.

### 2. Sub-Sample Stress Test

Splits data by market regime (bull/bear/sideways) and volatility (high/low) into multiple sub-samples, computing IC for each. Pass condition: IC sign in 60%+ of sub-samples matches the overall sign.

This directly answers "does this factor only work in bull markets?"

### 3. Placebo Test

Generates 20 random permutations of the time series, computing IC for each. The real factor's IC must exceed the 95th percentile of the random permutation ICs. Also checks IC decay after time-shifting — if IC doesn't significantly decrease after a one-day shift, the signal may be spurious.

### 4. Half-Life Estimation

Computes IC across forward periods of 1, 2, 5, 10, 20, and 40 days, fitting an exponential decay curve. Half-life must exceed 5 days to pass.

A half-life that's too short means the factor's predictive power dissipates within 1-2 days — possibly sufficient for daily rebalancing strategies, but inadequate for WorldQuant BRAIN's evaluation framework, which emphasizes medium-term stability.

## How Anti-Overfit Enters the Scoring System

QuantGPT's factor score is a weighted combination of 6 dimensions:

```
Total = IC_Mean(15%) + IC_IR(15%) + Stability(15%) +
        AntiOverfit(15%) + GroupBT(15%) + WQ_Alignment(25%)
```

Anti-overfit carries 15% weight. Passing 3+ of the four tests earns full marks; 2 tests earns 60; 1 test earns 30; 0 tests earns 0.

Key design: **if CAGR or Sharpe is negative, the final grade cannot exceed C (≤ 59.9), regardless of other dimensions.** This is a hard cap, not a soft penalty.

WQ Alignment accounts for 25% — including Sharpe, Fitness, and Turnover compliance checks. This means a factor needs to pass both anti-overfit testing **and** WQ BRAIN simulation to achieve an A grade. Neither alone is sufficient.

## How Anti-Overfit Drives Evolution Direction

This is the more important part. Anti-overfit isn't just "one dimension of the score" — it directly influences the evolution engine's strategy selection.

The evolution engine runs a three-phase adaptive loop:

1. **Trajectory Analysis**: Evaluates quality metrics of historical iterations — score variance (exploration diversity), trend slope (convergence speed), consecutive decline count
2. **Strategy Selection**: Chooses one of 4 strategies based on trajectory characteristics
3. **Execution**: Generates candidate factors according to the selected strategy

Among the 7 rules governing strategy selection, anti-overfit is an implicit signal source:

- **EXPLOIT**: High score + low variance → refine current best. The premise is that the current best's score comes from a complete evaluation including anti-overfit
- **EXPLORE**: Low score + early stage → try new directions. A factor with high Sharpe but failing anti-overfit still gets a low total score, triggering EXPLORE instead of EXPLOIT
- **RECOMBINE**: 2+ consecutive declining rounds → crossover from historical high-scorers. Parents for crossover are sorted by total score, so factors passing anti-overfit naturally rank higher
- **SIMPLIFY**: Nesting depth > 8 → reduce complexity. Overfitting often stems from overly complex expressions

**An evaluation detail**: each candidate factor runs only 2/4 anti-overfit tests (IC stability + half-life) during iteration, not all 4. This is a speed-accuracy tradeoff — the fast screening stage uses low-cost detection to filter obvious overfitting, while the full 4-test suite runs only during final evaluation.

## Why Walk-Forward Alone Isn't Enough

Many systems only do Walk-Forward validation: rolling windows, train/test separation, checking out-of-sample performance. Better than nothing, but it has blind spots.

Walk-Forward validation is fundamentally a **temporal generalization test**. It tells you "will this factor still work in the near future" but doesn't tell you "is this factor robust across different market regimes" — that requires sub-sample stress testing. It also doesn't tell you "is this factor better than random noise" — that requires placebo testing.

QuantGPT uses Walk-Forward as a **second validation layer**, stacked on top of the four anti-overfit tests. Each rolling window's test segment can optionally run the complete anti-overfit suite:

```
Window Score = Test_IC(30%) + Test_IR(25%) + IC_Stability(20%) +
               AntiOverfit(15%) + Sharpe(10%)
```

Anti-overfit accounts for 15% in the window score, IC stability another 20%. Together they're 35%, exceeding the weight of any single metric.

## Results in Practice

This architecture produced 3 factors formally submitted to WorldQuant BRAIN (best Fitness 1.26, Sharpe 1.77), all passing IS tests.

A key data point: the Agent eliminated a large number of factors with high Sharpe but failing anti-overfit during iteration. Without this filter, the Agent would tend to converge on high-Sharpe, high-overfit-risk local optima — because Sharpe is the easiest metric to optimize for.

Anti-overfit isn't "check after you're done backtesting." It's part of the score, a signal source for iteration direction, and the core of the elimination mechanism. Treat it as a plugin, and you get "high-scoring factors." Treat it as architecture, and you get "robust high-scoring factors."

## One-Line Summary

> Anti-overfit testing shouldn't be the last thing you do. It should be embedded in your scoring system and iteration engine, so that overfitting factors don't even qualify to participate in evolution.

---

*Series: [The Expression Parser Is a Compiler, Not eval()](/posts/expression-parser-is-a-compiler/) · [API Guard Pattern](/posts/api-guard-pattern/) · [Design for the Endgame](/posts/design-for-the-endgame/)*
