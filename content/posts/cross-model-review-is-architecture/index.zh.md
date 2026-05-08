---
title: "跨模型评审是架构，不是经验"
date: 2026-05-08
draft: false
tags: ["AI Agent", "LLM", "DeepSeek", "Architecture", "Quant Research", "MCP"]
categories: ["Architecture Decisions"]
summary: "单 LLM 的自省有结构性盲点——它倾向于确认自己的输出。QuantGPT 在 factor-mine SKILL 的 Phase 0.5 强制 Claude 调用 DeepSeek 评审，是硬性规则，不是建议。这不是冗余，是结构性偏见的解药。"
---

> **单 LLM 的自省有结构性盲点——它倾向于确认自己的输出。QuantGPT 在 factor-mine SKILL 的 Phase 0.5 强制 Claude 调用 DeepSeek 评审，是硬性规则，不是建议。这不是冗余，是结构性偏见的解药。**

## 一、单 LLM 自省的结构性盲点

让 Claude 反思自己的输出是 LLM 应用的标准做法。Reflection、Self-Critique、Chain-of-Thought——名字不同，本质相同：**让模型用自己的能力评价自己**。

这个范式在小任务上有效。但在领域深度任务上有一个根本性局限：**一个模型的反思仍然落在它自己的训练分布里**。

举一个具体场景。Claude 设计了一个因子表达式：

```
-1 * rank(ts_av_diff(close, 10)) + rank(debt / enterprise_value)
```

接下来你让 Claude 评审这个表达式。它会输出"这个因子结合了价格反转和基本面信号，理论合理，符合 WQ BRAIN 风格"。

听起来很对。但这是同一个模型在评价同一个模型的产出——**它使用的认知工具集和原始生成时是同一套**。如果原始设计有偏见（例如对某种因子结构的过拟合），反思也会带这个偏见。

更糟的是 RLHF 对话训练让模型倾向于"完成任务"。让它"反思自己设计的因子"，它会倾向于支持自己——因为否定自己等于反复改设计、延长上下文、推迟交付。

**这是结构性问题，不是 prompt 工程能解决的**。

## 二、跨模型评审 ≠ "用 AI 检查 AI"

第一反应是简单粗暴的：让另一个 LLM 检查。但**不是任意一个**。

跨模型评审的有效性来自三件事的差异：

1. **训练数据分布差异** — 不同模型在不同语料上训练，知识盲点和强项不同
2. **RLHF 路径差异** — 不同人类反馈数据塑造不同的"什么算对"的判断
3. **推理风格差异** — 不同的 chain-of-thought 倾向

如果用 GPT-4o 评审 Claude 的输出：两者都是英文为主、对齐到通用 helpfulness、推理风格相近。差异不够大，互补价值有限。

跨模型评审的工程化要求**选择一个分布上有真正差异、且在目标领域有更强能力的模型**。对量化因子研究，这个模型是 **DeepSeek**。

## 三、为什么必须是 DeepSeek

我用 DeepSeek 做评审已经几个月。下面是为什么**它是当前量化场景的最优选择**——不是因为便宜，是因为分布对齐。

### 它来自一家量化公司

DeepSeek 的母公司是 **幻方量化（High-Flyer Quantitative Investment）**——国内千亿规模量化私募之一。

这不只是商业血缘标签。它意味着：

- 团队对量化研究的工作流有第一手理解
- 训练语料天然包含高比例的金融、统计、衍生品文本
- 在金融数学、因子分析、回测语义上的训练密度远超通用模型

让 Claude 评审的因子表达式拿给一个**根上就在做量化的团队训练出的模型**评审——这是分布对齐，不是噱头。

### 中文金融的最佳选择

Claude 的训练数据以英文为主。处理"中证 500 行业中性化"这种 phrase，它能理解，但不像处理 "S&P 500 sector neutralized" 那样自然。

DeepSeek 在中文金融语料上的密度远高于通用模型。当因子设计涉及 A 股市场结构（涨跌停、ST、停牌、双边市场、Wind 行业分类标准）时，DeepSeek 的反馈往往更具体、更准确。

这不是性能差异（两者 reasoning 都很强），是**领域接地（domain grounding）的差异**。

### R1 的推理可见性

DeepSeek-R1 暴露 `reasoning_content` 字段——你能看到模型的完整 chain-of-thought：

```python
result = ask_deepseek(prompt, model="deepseek-reasoner")
print(result["content"])     # 最终答复
print(result["reasoning"])   # 完整推理过程
```

对评审场景，**reasoning trace 比答案更重要**。"为什么这个因子可能过拟合"的推理路径，比一句"建议简化"有用 10 倍——前者能让 Claude 真正学到东西并修改设计，后者只是噪声。

OpenAI o1 把 reasoning trace 隐藏在产品后面，DeepSeek 直接暴露。对开发者来说这是质量差异，对 Agent 自治系统来说这是**可不可用的差异**。

### 推理深度对标 o1，价格低一个数量级

DeepSeek-R1 在 AIME / MATH-500 / GPQA 等数学推理基准上对标 OpenAI o1。

定价对比（截至 2026 年初）：

| 模型 | Input（百万 tokens） | Output（百万 tokens） |
|------|---------------------|----------------------|
| DeepSeek-R1 | ~¥4 | ~¥16 |
| OpenAI o1 | ~¥110 ($15) | ~¥440 ($60) |

差距约 **27 倍**。

每次 phase 0.5 评审消耗约 5K-10K tokens，单次成本约 ¥0.02-0.05。**便宜到可以让"评审"变成默认行为，而不是奢侈品**。价格便宜是结果不是原因，但它让"强制评审"在工程上成为可行的设计选择。

## 四、QuantGPT 的实现

QuantGPT 在 factor-mine SKILL 的 `Phase 0.5: DeepSeek 设计咨询` 强制 Claude 调用 DeepSeek。

### Phase 0.5 触发条件

```
- 开启新的研究方向时
- 现有信号家族 SC 饱和需要全新结构时
- 从知识库找不到现成可用的表达式模板时
```

满足任一即触发，**SKILL 流程中无法跳过**。

### MCP 工具实现

DeepSeek MCP server 是 196 行的 stdio JSON-RPC 服务（`scripts/mcp_deepseek.py`），核心定义：

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

通过 `.mcp.json` 注册到 Claude Code，Claude Agent 看到这个工具就能调用。

### 评审 prompt 的工程化

不是简单的"你看看这个因子怎么样"。Phase 0.5 喂给 DS 的是**结构化包**：

```
事实层:
  - 当前研究方向（来自 research_notes/archive/）
  - 已验证规则（来自 knowledge/rules/）
  - 已证伪路径（来自 knowledge/failures/）
  - Claude 的初步设计 + 设计理由

请求层:
  - 评价该设计是否有未识别风险
  - 推荐 1-2 个替代结构
  - 标出与已知 failures 的潜在冲突
```

DS 返回 reasoning_content + content，Claude 读完后**必须明确表态采纳/拒绝/调整**，并写入研究笔记。这一步不是讨论，是工程流程的一部分。

## 五、为什么必须"强制"，不是"建议"

把跨模型评审做成可选项是无效的。有两个原因：

### Agent 倾向于跳过非必需步骤

LLM 对话训练让 Agent 倾向于完成任务路径最短化。"建议你问问 DeepSeek"在 prompt 里写一千遍，Agent 在赶进度时仍会跳过——因为多调一次 API、等 30 秒、读完再写笔记，都是延迟。

### Prompt 级约束没有强制力

我在 [Harness 即治理](/posts/harness-is-governance/) 里详细论证过：违反 prompt 不产生任何后果。Agent 跳过"建议"，流程仍能继续，没有反馈信号告诉它做错了。

### 解决方法：硬规则 + 流程依赖

QuantGPT 把 phase 0.5 写成 SKILL 的硬性规则。不是 prompt 里的"建议"，是流程依赖——后续 phase 1 的 prompt 模板**显式引用 DS 评审结果**作为上下文。跳过 0.5，phase 1 的 context 缺失，流程失败。

这就是 Harness 即治理的具体应用：**用代码而不是 prompt 约束 Agent**。

## 六、Trade-off 与边界

跨模型评审不是免费午餐。

### 成本

- 每次 phase 0.5：~¥0.02-0.05 + 30 秒延迟
- 一次完整研究循环（4-8 phase 0.5）：~¥0.20-0.40
- 相比纯单 LLM 流程，token 成本约高 30%-50%

### 双模型同时错的可能性

如果某个偏见在 Claude 和 DeepSeek 的训练分布中**都存在**（例如某些经典因子的过拟合解读），双模型一致地"通过"，评审就失效了。

这种"分布重叠"的盲区目前没有完美方案。可行的缓解：

- 加入第三模型（GPT-4o / Qwen / Llama 3）做仲裁
- 当评审通过率长期 > 90% 时主动注入对抗 prompt（"假设这个因子是过拟合的，找证据"）
- 对历史已证伪的因子结构维护 hard-coded blacklist（不依赖任何 LLM 判断）

### 何时不适用

- 任务领域只有一个模型有真正知识（例如某种 niche 医学数据库）—— 跨模型互审没有信号
- 任务延迟敏感（毫秒级）—— 跨模型多一次 API 是 deal breaker
- 团队已经有人工评审流程 —— 加 LLM 评审是冗余

## 结尾

跨模型评审是**架构层级**的设计选择，不是 prompt 里的"请你再仔细想想"。

单 LLM 自省的局限来自训练分布——而训练分布只能用**另一个分布**去切割。当评审目标是量化因子，这个"另一个分布"几乎只有一个选择：DeepSeek。它来自一家做量化的公司，训练在大量中文金融语料上，推理深度对标 o1，价格低一个数量级到可以做默认行为。

QuantGPT 强制 Phase 0.5 不是因为我相信 DeepSeek 永远对——它会错。是因为**两个有真实分布差异的模型同时错的概率，比一个模型自己错的概率显著低**。架构的工作不是消除错误，是把错误率压到可接受的水平。

**Cross-Model Review Is Architecture, Not Heuristic.**
