---
title: "MCP 的问题不在协议层，在语义层"
date: 2026-04-02
draft: false
tags: ["MCP", "AI Agent", "Vibe Coding", "Intent Validation"]
categories: ["Architecture Decisions"]
summary: "MCP 协议本身没什么大问题。真正的问题是：自然语言到结构化指令之间存在一条语义鸿沟，而当前的 MCP 生态完全没有处理它。我用 Intent Validator 模式在自己的系统里解决了这个问题。"
---

## 问题是什么

我有一套 AI 驱动的量化研究系统。AI Agent 通过 MCP 工具提交各种研究任务：ML 训练、回测、因子研究等。每种任务有几十个参数，有些参数之间存在复杂的业务约束。

举个例子，ML 回测任务有一条关键规则：

> **禁止设置 start_date 和 end_date。** 必须使用全区间 OOF（Out-of-Fold）回测——样本内用交叉验证预测，样本外用最终模型。限制日期范围会导致数据点太少，结果不可靠。

这条规则写在 MCP server 的 instructions 里，用自然语言告诉 Agent：

```
ml_backtest: ALWAYS use full period (do NOT set start_date/end_date).
Empty dates = OOF backtest: in-sample uses cross-validated predictions,
out-of-sample uses final model.
NEVER restrict to test period only — too few data points.
```

问题来了：**Agent 可以完全无视这段话。**

LLM 读了这段 instruction，大多数时候会遵守。但偶尔——尤其在用户给了明确日期范围的情况下——它会直接把日期填进去。没有任何东西阻止它。请求合法地通过 MCP，合法地到达 REST API，Pydantic 模型校验通过（因为 start_date 就是个 string 字段），任务提交成功，结果不可靠但看起来没有报错。

**这不是 MCP 协议的 bug。JSON-RPC 传输没有问题，tool schema 定义没有问题。** 问题在于：自然语言规则没有代码级的强制力。

## 三层校验，中间是空的

我的系统有三层校验，但中间恰好缺了最关键的一层：

```
第一层：MCP instructions（自然语言）  → Agent 可以忽略
第二层：???                          → 空的
第三层：Pydantic 模型校验（类型检查） → start_date 是合法 string，通过
```

第一层是"建议"，第三层是"类型检查"。中间缺一层"业务规则的代码级强制"。

这个空洞在所有 MCP 应用里都存在。MCP 的 tool schema 能描述参数类型（string、int、array），但不能描述参数之间的业务约束（"如果 symbols 为空，则 universe 必须非空"、"ml_backtest 不允许设置日期"）。

## 解决方案：Intent Validator

我在 MCP tool 层和 HTTP 调用层之间加了一个 Intent Validator：

```
Agent 调用 MCP submit()
  → JSON 解析
    → Intent Validator ← 新增：代码级业务规则强制
      → HTTP 调用 Orchestration API
        → Pydantic 类型校验
```

核心设计：

### 1. 注册式规则架构

每个任务类型可以注册多条校验规则，新增规则不需要改 MCP server 代码：

```python
@_register("ml_backtest")
def _no_date_restriction(config, objective):
    violations = []
    if config.get("start_date"):
        violations.append({
            "rule": "ml_backtest_no_start_date",
            "error": "start_date is set but ML backtest requires full period.",
            "fix": "Remove start_date for full-period OOF backtest.",
        })
    return violations
```

### 2. 硬拦截 + 软警告

每条规则有 severity 级别。硬错误直接拒绝提交并返回修复指令，警告附带在成功响应里：

```python
# 硬错误：直接拒绝
{"rule": "ml_backtest_no_start_date", "error": "...", "fix": "..."}

# 软警告：允许提交但提醒
{"rule": "ml_backtest_split_dates_recommended", "severity": "warning",
 "error": "split_dates not provided", "fix": "Pass split_dates from training result"}
```

### 3. 可操作的错误信息

每条 violation 包含三个字段：`rule`（机器可读标识）、`error`（问题描述）、`fix`（修复指令）。Agent 收到拒绝后能直接按 `fix` 字段修正参数重新提交，不需要人类介入。

### 4. 两条路径统一覆盖

同一套规则同时覆盖 MCP 和 HTTP 两条入口路径。MCP 路径在 `submit()` 里调用 validator，HTTP 路径通过 Pydantic 的 `model_validator` 在请求反序列化时自动触发：

```python
class IntentValidatedModel(BaseModel):
    _intent_task_type: str = ""

    @model_validator(mode="after")
    def _run_intent_validation(self):
        result = validate_intent(self._intent_task_type, self.model_dump(), ...)
        if hard_errors:
            raise ValueError(f"Intent validation failed: {msg}")
        return self

class MLBacktestRequest(IntentValidatedModel):
    _intent_task_type: str = "ml_backtest"
    model_id: str
    # ...
```

这样无论 Agent 通过 MCP 提交，还是脚本通过 curl 直接调 REST API，都经过同一套业务规则校验。

## 已注册的规则

| 任务类型 | 规则 | 级别 |
|---------|------|------|
| ml_training | 空 symbols 必须有 universe | 硬拦截 |
| ml_training | model_type 白名单校验 | 硬拦截 |
| ml_backtest | 禁止设 start_date/end_date | 硬拦截 |
| ml_backtest | 必须有 model_id | 硬拦截 |
| ml_backtest | 必须有 universe 或 symbols | 硬拦截 |
| ml_backtest | 建议传 split_dates | 警告 |
| ml_predict | 必须有 model_id + universe | 硬拦截 |
| rolling_factor_research | 日期范围 >= 窗口总长 | 硬拦截 |
| data_update | update_mode 枚举校验 | 硬拦截 |
| 通用 | 日期格式 YYYY-MM-DD | 硬拦截 |
| 通用 | start_date < end_date | 硬拦截 |

## 这和 Harness 的区别

Claude Code 有 permission mode 和 hooks，Cursor 有 rules 文件。这些是 Harness 模式——从外部约束 Agent 的行为边界。

Intent Validator 不是 Harness。区别：

- **Harness** 管的是"Agent 能不能调这个工具"——粒度是工具级别
- **Intent Validator** 管的是"Agent 调这个工具时，参数组合是否符合业务语义"——粒度是参数级别

Harness 能做到"禁止 Agent 调 submit"，但做不到"允许 Agent 调 submit，但如果 task_type 是 ml_backtest 且 start_date 非空则拒绝"。这种参数间的条件约束需要在应用层实现。

## 对 Vibe Coding 的启示

Vibe coding 的核心矛盾是：**自然语言的模糊性 vs 系统操作的精确性。** 用户说"帮我跑个回测"，这句话可以映射到几十种不同的参数组合，其中只有少数几种是合理的。

当前的解决思路集中在两端：

- **提升 LLM 理解能力**：让模型更好地理解 instructions → 有上限，永远不会 100%
- **收紧 tool schema**：用更严格的参数定义 → JSON Schema 表达力有限，描述不了跨参数约束

缺失的中间层就是 Intent Validation：**在 LLM 输出之后、系统执行之前，用代码强制检查业务语义。** 不依赖 LLM 的理解能力，不受 JSON Schema 的表达力限制。

这个模式适用于任何 Agent 操作有业务约束的场景——不只是量化研究。医疗 Agent 不能开违禁药物组合、法律 Agent 不能引用已废止的法条、运维 Agent 不能在高峰期执行全量部署——这些约束都需要代码级强制，不能靠 prompt 软约束。

---

*系列第二篇。上一篇：[为什么我没有用多智能体架构做量化研究系统](/posts/why-not-multi-agent/)。*
