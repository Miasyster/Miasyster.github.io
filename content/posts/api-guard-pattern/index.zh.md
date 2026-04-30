---
title: "API Guard Pattern：为什么直接调用函数是被禁止的"
date: 2026-04-28
draft: false
tags: ["Architecture", "Security", "API Design", "Python"]
categories: ["Architecture Decisions"]
summary: "QuantGPT 用 threading.local 实现了一个运行时守卫：所有回测调用必须经过 API 边界，直接调用函数会抛异常。不是因为函数有什么危险，而是因为没有边界的系统无法被审计。"
---

> **QuantGPT 用 threading.local 实现了一个运行时守卫：所有回测调用必须经过 API 边界，直接调用函数会抛异常。不是因为函数有什么危险，而是因为没有边界的系统无法被审计。**

## 问题：LLM Agent 会走捷径

当你把一个 Python 项目暴露给 LLM Agent 时，Agent 倾向于做最高效的事：直接 `import` 你的函数，跳过 API 层。

```python
# Agent 很容易写出这种代码
from quantgpt.backtest import run_factor_backtest
result = run_factor_backtest("rank(close)", "hs300")
```

这段代码**能跑通**。回测引擎不关心调用者是谁。但这制造了一个不可审计的调用路径——没有任务 ID、没有日志、没有速率限制、没有权限检查。在一个 Agent 可以自治运行几十轮迭代的系统里，失去审计就等于失去控制。

文档约定（"请通过 API 调用"）对 LLM Agent 没有约束力。Agent 看到 `import` 路径更短，就会用。

## 解决方案：运行时强制

```python
_api_context = threading.local()

def _require_api_context():
    if not getattr(_api_context, 'active', False):
        raise RuntimeError(
            "run_factor_backtest() must be called within api_context()"
        )

@contextmanager
def api_context():
    _api_context.active = True
    try:
        yield
    finally:
        _api_context.active = False
```

`run_factor_backtest()` 的第一行就是 `_require_api_context()`。没有被 `api_context()` 包裹的调用会立即抛 `RuntimeError`。

线程隔离用 `threading.local()`——不同线程的上下文互不影响，进程池里的 worker 各自独立。

## 合法调用点

系统中只有几个地方包裹了 `api_context()`：

```python
# task_executor.py — 所有回测任务的入口
def _run_backtest_in_process(expression, universe, ...):
    enable_api_context()
    try:
        return run_factor_backtest(expression, universe, ...)
    finally:
        disable_api_context()
```

API 路由、MCP 工具、迭代引擎——全部通过 `task_executor` 提交任务，`task_executor` 负责启用上下文。调用链变成：

```
HTTP/MCP 请求 → task_executor → enable_api_context → run_factor_backtest
```

新增一个调用点？必须显式包裹 `api_context()`。忘了包裹？运行时炸。不存在"偷偷绕过"的可能。

## 测试怎么办

测试也要走这个守卫。通过 pytest autouse fixture 全局启用：

```python
@pytest.fixture(autouse=True)
def _enable_api_context():
    with api_context():
        yield
```

所有测试自动在 `api_context()` 内运行。测试覆盖的是真实的调用路径，不是一个特殊的"测试模式"。

## 为什么不用装饰器/中间件

常见的替代方案：

- **装饰器**：在函数定义处标记，但调用者无感知，不改变调用习惯
- **中间件**：只保护 HTTP 层，不保护 Python 内部调用
- **文档约定**：对人有效，对 Agent 无效

`threading.local` 守卫的特点是**调用者必须主动配合**——你必须先进入 `api_context()` 才能调用。这把"约定"变成了"约束"。

## 更深层的原则

这个模式解决的不是安全问题（回测函数本身无害），而是**边界问题**。

在一个 LLM Agent 可以自由调用任意 Python 函数的环境中，如果没有运行时强制的边界，系统会退化成一个大泥球——所有调用路径都是合法的，审计日志捕获不到直接调用，你无法区分"通过 API 发起的研究任务"和"Agent 随手跑了一个回测"。

边界不是限制自由，而是让自由变得可追踪。

代码约束 > 文档约束。运行时强制 > 静态分析。对 LLM Agent 尤其如此——它们只尊重会报错的规则。

---

*系列文章：[AI as Operator, Kernel as Law](/posts/ai-as-operator/) · [MCP 的问题不在协议层，在语义层](/posts/mcp-semantic-gap/) · [Skill 编排 > Agent 循环链](/posts/skill-orchestration-over-agent-loops/)*
