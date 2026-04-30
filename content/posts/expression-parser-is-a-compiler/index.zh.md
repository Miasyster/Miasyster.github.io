---
title: "表达式解析器是编译器，不是 eval()"
date: 2026-04-29
draft: false
tags: ["Compiler", "Expression Parser", "Architecture", "Security"]
categories: ["Architecture Decisions"]
summary: "QuantGPT 的核心是一个 870+ 行的手写递归下降解析器，支持 80+ 算子、截面/时序自动分组、双模式编译。不是因为不知道 eval() 更简单，而是因为 eval() 做不到的事情恰好是最重要的。"
---

> **QuantGPT 的核心是一个 870+ 行的手写递归下降解析器，支持 80+ 算子、截面/时序自动分组、双模式编译。不是因为不知道 eval() 更简单，而是因为 eval() 做不到的事情恰好是最重要的。**

## 为什么不能用 eval()

因子表达式看起来像数学公式：

```
-1 * rank(ts_av_diff(close, 10)) + rank(debt / enterprise_value)
```

最直觉的实现：把 `rank`、`ts_av_diff` 定义为 Python 函数，然后 `eval()` 整个字符串。三行代码搞定。

但这个系统的调用者是 LLM Agent。Agent 生成的表达式不可预测。`eval()` 意味着任意代码执行——不只是"安全隐患"，而是**系统性地无法约束 Agent 的行为空间**。你无法区分"合法的因子表达式"和"Agent 幻觉出的任意 Python"。

即便忽略安全，`eval()` 也解决不了因子计算的核心难题：截面算子和时序算子的分组语义完全不同。

## 截面 vs 时序：同一个括号，不同的语义

```
rank(close)              — 按 trade_date 分组，在同一天所有股票中排名
ts_mean(close, 20)       — 按 stock_code 分组，在同一只股票的历史中滑动平均
```

`rank()` 是截面算子：同一天的所有股票是一个截面，rank 在截面内计算。`ts_mean()` 是时序算子：同一只股票的历史价格是一条时序，均值沿时间轴滚动。

两者的函数签名看起来一样（接收一列数据），但底层的 `groupby` 完全不同。`eval()` 无法从语法中推断出这个语义差异。手写解析器可以：

```python
# 解析器在识别到 rank() 时，自动注入截面分组
s.groupby(df['trade_date']).rank(pct=True)

# 识别到 ts_mean() 时，自动注入时序分组
_apply_ts_op_per_stock(df, lambda g: g.rolling(window).mean())
```

**分组逻辑对表达式作者完全透明。** 写 `rank(close)` 的人不需要知道底层是按日期分组的——解析器根据算子类型自动处理。这才是"编译"：把高层意图翻译成正确的低层操作。

## 双模式编译

同一个表达式可以编译到两个目标：

- **`mode="local"`**：开放全部 80+ 算子，包括 `tanh`、`sigmoid`、`rsi`、`macd` 等本地扩展算子。用于快速实验和自由探索。
- **`mode="wq"`**：只允许 WorldQuant BRAIN 兼容的算子子集。用于提交前校验——如果表达式包含 WQ 不支持的算子，编译阶段就报错，不用等到提交时被平台拒绝。

28 个 WQ-only 远程算子（`vector_neut`、`ts_regression`、`bucket` 等）在本地模式下注册为 stub——调用时抛出 `RuntimeError`，提示"此算子仅在 WQ BRAIN 平台上可用"。

这让 Agent 可以自由探索（local 模式），然后在提交前切换到 wq 模式做合规检查。两个阶段用的是同一个解析器，只是编译目标不同。

## 安全约束

LLM Agent 会生成各种边界情况。解析器硬编码了三道防线：

```python
MAX_WINDOW = 500        # 滚动窗口上限
MAX_DEPTH = 100         # 递归深度限制
MAX_EXPRESSION_LENGTH = 1000  # 表达式字符数上限
```

- **窗口上限**：防止 `ts_mean(close, 99999)` 吃掉所有内存
- **递归深度**：防止 `rank(rank(rank(rank(...))))` 无限嵌套
- **表达式长度**：防止 Agent 生成超长组合表达式

这些不是建议值，是运行时强制。超过限制直接抛异常，Agent 必须修改表达式。

## 80+ 算子的分类

| 类别 | 数量 | 示例 |
|:-----|:-----|:-----|
| 一元算子 | 8 | `log`, `abs`, `sign`, `scale`, `sqrt` |
| 截面算子 | 4 | `rank`, `zscore`, `group_rank`, `group_zscore` |
| 时序算子 | 18+ | `ts_mean`, `ts_std`, `ts_corr`, `decay_linear`, `ts_av_diff` |
| 技术指标 | 8 | `rsi`, `macd`, `ema`, `sma`, `atr`, `obv`, `boll_*` |
| 条件/特殊 | 5 | `where`, `trade_when`, `clip`, `indneutralize`, `ternary` |
| WQ 远程 | 28 | `vector_neut`, `ts_regression`, `bucket`, `quantile` |
| 算术运算 | 11 | `+`, `-`, `*`, `/`, `^`, `>`, `<`, `==`, `!=`, `and`, `or` |

每个算子的分组语义（截面/时序/标量）在解析器中显式注册。不存在"通用算子"——每个算子必须声明它的计算域。

## 为什么不用现成的解析框架

Python 生态有 `lark`、`ply`、`pyparsing`。为什么手写？

1. **语义动作与解析深度耦合。** 解析器不只产出 AST——它在解析过程中直接生成 Pandas 计算图。`lark` 的 Transformer 做得到，但代码量不会更少，还多了一层抽象。
2. **双模式切换需要解析时判断。** `mode="wq"` 在遇到非兼容算子时要立即报错，不是解析完再检查。嵌入在递归下降逻辑里最自然。
3. **错误信息要给 Agent 看。** "第 23 个字符处 'ts_regrssion' 不是合法算子，你是否想用 'ts_regression'？" 这种带纠错建议的错误信息，通用解析框架很难做到。

870 行代码听起来多，但它包含了解析、编译、安全检查、错误处理、80+ 算子的完整实现。这是系统中最稳定的模块——在 3 个因子正式提交 WorldQuant BRAIN（IS 全部通过）的过程中，解析器本身零 bug。

## 一句话总结

> 解析器不是"输入处理"——它是类型系统、安全边界和编译目标的统一实现。当你的调用者是 LLM 时，解析器就是你的第一道也是最重要的防线。

---

*系列文章：[API Guard Pattern](/posts/api-guard-pattern/) · [让 AI 写的代码跑起来，但别让它跑出去](/posts/sandbox-defense-in-depth/) · [Skill 编排 > Agent 循环链](/posts/skill-orchestration-over-agent-loops/)*
