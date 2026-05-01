---
title: "Harness 即治理：用代码约束 Agent，不用 Prompt"
date: 2026-05-01
draft: false
tags: ["AI Agent", "Governance", "Harness", "MCP", "Security"]
categories: ["Architecture Decisions"]
summary: "Agent 治理的主流方案是在 prompt 里写规则。但 LLM 可以无视 prompt。真正有效的治理不在 Agent 内部，在 Agent 外部的 Harness 层——工具定义了能做什么，错误定义了不能做什么，代码定义了边界在哪。"
---

> **Agent 治理的主流方案是在 prompt 里写规则。但 LLM 可以无视 prompt。真正有效的治理不在 Agent 内部，在 Agent 外部的 Harness 层——工具定义了能做什么，错误定义了不能做什么，代码定义了边界在哪。**

## Prompt 治理为什么失败

当你在 system prompt 里写"回测必须通过 API 调用"，你做了一个假设：Agent 会遵守文本指令。

这个假设在大多数时候成立。但它在最关键的时候失败——当 Agent 找到更高效的路径时，它会倾向于走捷径。长上下文中早期指令会被稀释，多轮对话中 Agent 可能"遗忘"约束。

更根本的问题：**prompt 级约束没有强制力。** 违反 prompt 不会产生任何后果——Agent 继续运行，只是做了一件你不希望它做的事。你可能直到审阅输出时才发现。

这不是 LLM 的 bug，这是文本约束的本质局限。

## Harness 层：Agent 和系统之间的接口

Harness 不是一个框架，而是一个位置——Agent 和领域系统之间的接口层。

```
Agent（Claude / GPT / DeepSeek）
    │
    ├── Harness 层 ←── 治理在这里
    │   ├── MCP 工具定义（能做什么）
    │   ├── 运行时守卫（不能做什么）
    │   ├── 参数校验（输入边界）
    │   └── 返回值设计（输出完整性）
    │
    └── 领域系统
        ├── 回测引擎
        ├── 表达式解析器
        ├── 反过拟合检测
        └── WQ BRAIN 模拟器
```

Harness 层决定了三件事：

1. **Agent 能做什么**——通过暴露哪些工具
2. **Agent 不能做什么**——通过运行时守卫和参数校验
3. **Agent 看到什么**——通过返回值的设计

这三件事合在一起就是治理。不是一套规则文档，是一层代码。

## 治理机制一：能力边界

Agent 的能力由它能调用的工具定义。你不暴露的工具，Agent 就无法使用。

QuantGPT 暴露了 8 个 MCP 工具。Agent 不能直接操作数据库、不能直接下载行情数据、不能直接修改缓存。它只能通过 `run_backtest` 获取回测结果——回测引擎内部做什么（数据获取、缓存管理、并发控制），Agent 无权过问。

这和操作系统的系统调用是同一个思路。用户态程序不能直接读写磁盘——它必须通过内核的系统调用。Agent 不能直接操作领域系统——它必须通过 Harness 的工具。

**不暴露 = 不存在。** 这比"请不要调用这个函数"可靠一万倍。

## 治理机制二：运行时强制

暴露的工具也需要内部约束。QuantGPT 的三层运行时防御：

**API 边界守卫：**
```python
_api_context = threading.local()

def _require_api_context():
    if not getattr(_api_context, 'active', False):
        raise RuntimeError("必须通过 API 调用")
```

回测函数的第一行就是这个守卫。Agent 如果试图 `import` 然后直接调用（绕过 MCP 工具），立即报错。

**表达式安全限制：**
```python
MAX_DEPTH = 100
MAX_WINDOW = 500
MAX_EXPRESSION_LENGTH = 1000
```

Agent 生成的表达式超过限制？解析器拒绝执行。Agent 被迫生成更简单的表达式。

**双模式编译：**
```python
if mode == "wq" and operator not in WQ_COMPATIBLE_OPS:
    raise RuntimeError(f"'{operator}' 在 WQ BRAIN 模式下不可用")
```

Agent 想提交一个包含本地扩展算子的表达式到 WQ BRAIN？编译阶段就拦截，不用等到平台拒绝。

每一层都是**代码级强制**——不是建议，不是文档，不是 prompt 指令。违反就报错，Agent 必须调整。

## 治理机制三：信息边界

Agent 的决策质量取决于它看到的信息。Harness 通过**返回值设计**控制 Agent 看到什么。

QuantGPT 的 `score_factor` 返回 6 维评分：IC 均值、IC_IR、稳定性、反过拟合、分组回测、WQ 对齐度。Agent 看到的不是一个笼统的"好/坏"，而是**诊断性信息**——它知道哪个维度拖了后腿，从而做出有针对性的改进。

同时，`diagnose_factor` 返回的不只是"评分低"，而是具体的失败模式和改进建议。Agent 不需要自己推理"为什么 Sharpe 低"——工具直接告诉它。

**信息的丰富度和结构决定了 Agent 的决策上限。** 给 Agent 一个数字，它只能做比较。给它一张诊断表，它能做推理。

## 治理机制四：交叉审查

单 Agent 做研究有一个根本问题：生成假设的模型和评估假设的模型是同一个。Confirmation bias 是结构性的。

QuantGPT 用 Harness 层解决这个问题：每个研究结论必须经过第二个 LLM（DeepSeek）独立审查。

```
Agent（Claude）做出判断
    │
    ├── 收集事实数据（回测指标）
    ├── 写出判断 + 推理链
    │
    └── Harness 层：调用 DeepSeek 评审
        ├── 同意 → 输出结论
        └── 不同意 → 呈现双方立场，采用保守方案
```

这不是 Agent 自己选择是否要第二意见——是 Harness 层**强制**的。Agent 无法跳过这一步，因为研究结论的输出接口就要求包含交叉审查结果。

## 为什么 Harness = 治理

总结一下四个机制：

| 治理维度 | 机制 | 实现层 |
|:---------|:-----|:-------|
| 能做什么 | 工具暴露 | MCP 工具定义 |
| 不能做什么 | 运行时守卫 | threading.local + 解析器限制 |
| 看到什么 | 返回值设计 | 工具返回的信息结构 |
| 决策可信度 | 交叉审查 | 双 LLM 强制评审 |

所有治理机制都在 Harness 层——Agent 外部、领域系统外部的接口层。

这个位置至关重要。如果治理在 Agent 内部（prompt），它是脆弱的。如果治理在领域系统内部（业务逻辑），它和业务耦合了。Harness 层是唯一一个**既独立于 Agent 又独立于领域系统**的位置。

**把治理写在 Harness 里，Agent 可以换，领域系统可以改，治理规则不受影响。**

## 一句话总结

> Agent 治理不是写更好的 prompt。是设计更好的 Harness——用代码定义能力边界、运行时约束、信息结构和审查流程。Agent 只尊重会报错的规则，所以把规则写成代码。

---

*系列文章：[为什么做基建不做 Agent](/posts/why-agent-infra-not-agent/) · [Agent-Native 架构](/posts/agent-native-architecture/) · [AI as Operator, Kernel as Law](/posts/ai-as-operator/)*
