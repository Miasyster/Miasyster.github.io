---
title: "Agent-Native 架构：为 Agent 设计系统，不是为人"
date: 2026-05-01
draft: false
tags: ["AI Agent", "Architecture", "MCP", "System Design"]
categories: ["Architecture Decisions"]
summary: "当系统的操作员从人类变成 LLM Agent 时，设计原则需要根本性改变。人类需要 GUI 和文档，Agent 需要语义清晰的工具和会报错的约束。我在 QuantGPT 中实践的 Agent-Native 设计原则：为调用者的认知模式设计，不是为设计者的审美设计。"
---

> **当系统的操作员从人类变成 LLM Agent 时，设计原则需要根本性改变。人类需要 GUI 和文档，Agent 需要语义清晰的工具和会报错的约束。**

## 操作员变了，设计原则也要变

传统软件为人类设计：GUI 引导操作流程，文档解释功能，错误提示帮助人类理解问题。

当操作员变成 LLM Agent 时，这套设计失效了。Agent 不看 GUI，不读文档（至少不以人类的方式），不需要友好的错误提示——它需要**机器可解析的错误信息**来调整下一步行动。

Agent-Native 架构不是"给 Agent 加个 API"。它是从系统的操作员是 Agent 这个前提出发，重新思考每一个设计决策。

## 原则一：工具适配调用者的认知模式

人类擅长组合多个小工具来完成一件事（Unix 哲学：`cat | grep | sort`）。LLM 不擅长这个——它擅长从一个完整结果中提取关键信息。

QuantGPT 的 `run_backtest` 一次性返回：Sharpe、IC、分组收益、换手率、最大回撤、行业暴露、因子载荷。不是拆成 7 个小工具让 Agent 一个一个调。

```python
# 人类友好的设计：7 个小工具
get_sharpe(factor_id)
get_ic(factor_id)
get_turnover(factor_id)
get_drawdown(factor_id)
# ...Agent 需要调 7 次才能看到全貌

# Agent-Native 的设计：1 个完整工具
run_backtest(expression, universe) → {sharpe, ic, turnover, drawdown, ...全部}
# Agent 一次调用就看到完整图景，自己决定关注什么
```

**工具设计要适配调用者的认知模式，不是设计者的审美偏好。** "小而专"是人类的审美；"完整而自包含"是 Agent 的需求。

## 原则二：错误是接口，不是异常

人类看到错误信息会思考原因然后手动修复。Agent 看到错误信息会解析内容然后自动调整。

这意味着错误信息的设计目标从"让人理解"变成"让 Agent 能行动"：

```python
# 人类友好的错误
raise ValueError("表达式语法错误")

# Agent-Native 的错误
raise ValueError(
    "第 23 个字符处 'ts_regrssion' 不是合法算子，"
    "你是否想用 'ts_regression'？"
    "可用的时序算子：ts_mean, ts_std, ts_corr, ts_regression, ..."
)
```

第二种错误包含了三层信息：**哪里错了**（位置）、**可能的修正**（纠错建议）、**可选的替代方案**（算子列表）。Agent 解析后可以直接生成修正后的表达式，不需要额外调用 `list_operators`。

## 原则三：约束用代码表达，不用文档

文档约束（"请通过 API 调用"）对 Agent 没有约束力。Agent 会走最短路径——如果直接 `import` 函数更快，它就会绕过 API。

Agent-Native 系统的约束必须是**运行时强制的**：

```python
_api_context = threading.local()

def _require_api_context():
    if not getattr(_api_context, 'active', False):
        raise RuntimeError("必须通过 API 调用")
```

Agent 只尊重会报错的规则。不会报错的规则等于不存在。

同理，表达式解析器的安全限制：

```python
MAX_DEPTH = 100          # 递归深度
MAX_WINDOW = 500         # 滚动窗口
MAX_EXPRESSION_LENGTH = 1000  # 表达式长度
```

不是"建议值"，是硬限制。超过就抛异常，Agent 被迫修改输入。

## 原则四：无状态 > 有状态

有状态的工具意味着调用顺序重要——必须先调 A 再调 B，否则 B 读不到 A 设置的状态。这对 Agent 是一个隐式约束，而且是文档级的（你需要告诉 Agent "先调 A"），不是代码级的。

无状态工具消除了这个问题。每次调用独立完备，Agent 可以以任意顺序、任意频率调用任何工具。

```python
@mcp.tool()
async def run_backtest(expression: str, universe: str = "hs300", ...):
    # 不读全局状态
    # 不依赖上一次调用的结果
    # 返回完整的回测报告
    ...
```

工具不知道自己是被"第一次"调用还是"第 15 次迭代"调用。它不关心上下文。这意味着 Agent 的任何决策路径都是合法的——不存在"调用顺序错误"这种问题。

## 原则五：语义命名是路由

人类系统需要 router 来分发请求。Agent-Native 系统不需要——工具名本身就是路由。

```
run_backtest           — 我想知道这个因子的表现
score_factor           — 我想知道综合评分
diagnose_factor        — 我想知道为什么表现不好
run_anti_overfit       — 我想知道是不是过拟合了
```

LLM 根据工具名和描述选择工具。名字越接近意图描述，选择准确率越高。不需要在 Agent 代码里写 `if intent == "diagnose": call diagnose_factor`——模型自己会做这个映射。

## 可测试性：被低估的优势

Agent 的决策是随机的——同一个输入，不同运行可能不同输出。你没法对 Agent 的行为写单元测试。

但 Agent-Native 的工具层是确定性的。QuantGPT 有 74 个测试覆盖：

- 80+ 算子的数学正确性
- 截面/时序分组语义
- 反过拟合统计检验
- WQ BRAIN 指标计算

测试保证的是：无论 Agent 怎么调用工具，返回的结果都是正确的。Agent 的决策质量取决于模型能力，数据的正确性取决于基建——后者是你能控制和验证的。

## 一句话总结

> 为 Agent 设计系统和为人类设计系统是两件不同的事。完整返回 > 小而专，运行时强制 > 文档约束，无状态 > 有状态，语义命名 > 显式路由。当你的操作员是 LLM 时，系统设计的每一个决策都需要重新审视。

---

*系列文章：[Skill 编排 > Agent 循环链](/posts/skill-orchestration-over-agent-loops/) · [API Guard Pattern](/posts/api-guard-pattern/) · [Harness 即治理](/posts/harness-is-governance/)*
