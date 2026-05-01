---
title: "为什么我做 Agent 基建，而不是做 Agent"
date: 2026-05-01
draft: false
tags: ["AI Agent", "Architecture", "MCP", "Strategy"]
categories: ["Architecture Decisions"]
summary: "当所有人都在卷 Agent 框架的时候，我选择了另一条路：不做 Agent，做 Agent 的基建。不是因为做不了 Agent，而是因为 Agent 层是消耗品，基建层是资产。做资产，不做消耗品。"
---

> **当所有人都在卷 Agent 框架的时候，我选择了另一条路：不做 Agent，做 Agent 的基建。不是因为做不了 Agent，而是因为 Agent 层是消耗品，基建层是资产。**

## 一个反直觉的选择

LangChain、CrewAI、AutoGen、Dify、Coze——每个月都有新的 Agent 框架发布。它们都在解决同一个问题：**怎么让 LLM 完成多步骤任务**。规划器决定做什么，执行器去做，反思器评估结果，然后循环。

QuantGPT 也用了 AI Agent。但我做了一个和主流方向相反的决策：

**我没有写一行 Agent 代码。**

全部 Agent 逻辑——规划、决策、迭代、反思——全部由 Claude 自己完成。我只做了一件事：给它一套好用的工具。8 个 MCP 工具，每个是无状态纯函数。没有 planner，没有 router，没有 reflection loop。

这不是偷懒。这是战略选择。

## 做 Agent 有多难

"做一个 Agent"听起来不难。调 API，写 prompt，加个循环。但那是 demo。

### 复刻领域深度 Agent，难度比肩从零造 Claude Code

Claude Code 看起来"就是调 API 加工具调用"，但它的工程深度远超表面：上下文工程（什么时候压缩历史、什么时候丢弃）、工具编排（优先级、冲突处理、失败重试）、安全边界（哪些操作需要确认）、自我纠错（识别错误路径后怎么回退）。

要在量化研究领域做一个同等深度的 Agent，等于从零造一个"量化版 Claude Code"。这不是一个开源项目能承受的工程量。

### Agent 前沿变化极快

2024 年初的最佳实践（ReAct prompting + 固定工具链），到年底已经被 MCP + 原生工具调用取代。2025 年的共识（多 Agent 协作）正在被质疑——单 Agent + 好工具可能更有效。

你今天花三个月做的 Agent 逻辑，三个月后可能因为模型能力提升而变成多余的代码。Planner？模型自己会规划了。Reflector？模型自己会反思了。Router？模型自己会选工具了。

**你在和模型能力的增长速度赛跑，而你跑不过它。**

### Agent 层是同质化的

所有人用的 LLM 是同一批。Agent 框架本质上是对这些模型的包装。差异化不在 Agent 层，差异化在你喂给 Agent 的**工具有多好**。

一个语义正确的表达式解析器、一个统计严谨的反过拟合检测系统——这些是几个月的领域积累，不是换一个 prompt 就能复制的。

**Agent 是通用的，工具是专有的。投资应该在专有层。**

## 搭便车：Agent-Infra 的核心优势

做 Agent-Infra 最大的好处不是"省事"，而是**你自动享受 Agent 能力上升带来的红利**。

```
LLM Agent（Claude / GPT / DeepSeek）
    │
    └── 8 个 MCP 工具（基建层）
        ├── run_backtest         全市场分组回测
        ├── score_factor         0-100 综合评分
        ├── diagnose_factor      失败模式诊断
        ├── run_anti_overfit     4 项反过拟合检验
        ├── run_rolling_validation Walk-forward 验证
        ├── validate_expression  语法校验（80+ 算子）
        ├── list_operators       算子文档
        └── list_universes       股票池和基准
```

当 Claude 从 3.5 升级到 Opus 4 时，我没改一行代码，但 Agent 的研究质量明显提升了。当 Claude Code 新增了 skill 系统，我写了一个 `/factor-mine` skill，Agent 立刻获得结构化研究能力——还是没改基建代码。

**如果你自己做了 Agent，模型升级时你需要重写 Agent 逻辑来适配新能力。如果你做的是基建，模型升级时你什么都不用做。**

你站在一个上升的电梯上，电梯的动力来自全世界最大的 AI 实验室。

## 可组合性：不绑定任何一个 Agent

如果明天 GPT-5 在工具调用上超过了 Claude 呢？**零改动，直接切换。**

工具是无状态纯函数——接收参数，返回结果，不知道调用者是谁。MCP 协议本身就是模型无关的。

如果你自己做了 Agent，Agent 逻辑绑定了特定模型的 prompt 格式、API 结构、上下文窗口假设。换模型 = 重写 Agent。做基建的人不需要做这个选择。

## 可测试性：Agent 没法测，工具可以

Agent 的决策是随机的——同一个输入，不同运行可能不同输出。你没法对 Agent 行为写单元测试。

但工具是确定性的。QuantGPT 有 74 个测试，覆盖 80+ 算子的数学正确性、截面/时序分组语义、反过拟合统计检验、WQ BRAIN 指标计算、API 边界守卫。

**无论 Agent 怎么调用这些工具，返回的结果都是正确的。** Agent 的决策质量取决于模型，数据的正确性取决于基建——后者是你能控制的。

## 护城河在领域知识

做 Agent 框架的护城河几乎没有。LangChain 今天的用户明天可以迁移到 CrewAI，成本极低。

做 Agent-Infra 的护城河在**领域知识的编码密度**：

表达式解析器里编码了截面算子和时序算子的分组语义差异、80+ 算子的数学实现和边界处理、WQ BRAIN 兼容性的双模式编译、LLM 输入的安全约束。反过拟合系统里编码了 IC 稳定性统计检验、牛熊震荡子样本切分、安慰剂检验、信号半衰期拟合。

**这些不是 prompt engineering 能复制的。** 每一个算子的分组语义、每一个统计检验的阈值、每一个安全限制的数值，背后都有具体的失败案例。

换一个 Agent 框架很容易。换一套经过验证的领域工具链很难。

## 治理天然内嵌

自己做 Agent 面临一个永恒的问题：**怎么控制 Agent 的行为**。

主流方案是 prompt 级约束。问题是 LLM 可以无视 prompt。做基建的话，治理机制在代码层：

```python
# Agent 想直接调用回测函数？RuntimeError。
def _require_api_context():
    if not getattr(_api_context, 'active', False):
        raise RuntimeError("必须通过 API 调用")

# Agent 想生成超深嵌套的表达式？解析器拒绝。
MAX_DEPTH = 100
MAX_WINDOW = 500
MAX_EXPRESSION_LENGTH = 1000
```

Agent 只尊重会报错的规则。**Harness 就是治理。** 工具层定义了 Agent 能做什么、不能做什么，这比任何 prompt 都可靠。

## 什么时候应该自己做 Agent

有场景确实需要自己做 Agent：

- **极端确定性要求**——医疗、法律、金融交易执行，不能容忍 LLM 的随机性
- **超高吞吐量**——每秒几千请求，LLM 推理的延迟和成本不可接受
- **模型能力不足**——目标用户只能用 GPT-3.5 级别的模型，需要外部脚手架来补偿

但如果你的场景是：**任务需要灵活判断、中间步骤不可预见、有足够聪明的 LLM 可用**——做基建比做 Agent 更聪明。

## 一句话总结

> 不要和 LLM 的进化速度竞争。做它的基建，让它每一次变聪明都让你的系统变得更好。Agent 是消耗品——框架会过时，prompt 会失效，编排逻辑会被模型原生能力取代。Agent-Infra 是资产——领域知识不会过时，工具的正确性不会过时，统计检验的严谨性不会过时。做资产，不做消耗品。

---

*系列文章：[Agent-Native 架构](/posts/agent-native-architecture/) · [Harness 即治理](/posts/harness-is-governance/) · [Skill 编排 > Agent 循环链](/posts/skill-orchestration-over-agent-loops/)*
