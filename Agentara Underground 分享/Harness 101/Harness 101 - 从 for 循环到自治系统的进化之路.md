# Harness 101：从 for 循环到自治系统的进化之路

> 来源: 飞书云文档 (D0Jzd84c6oxHBgxN82VcfRMVndh)

---

## Agent Loop：一个被低估的核心抽象

> LLM 在一个循环中反复调用工具，直到它决定停下来。

---

## Shallow Agent vs. Deep Agent

**Shallow Agent**：完全依赖上下文窗口，没有显式计划，没有持久记忆。

**Deep Agent** 叠加了四根支柱：
1. 计划工具（Todo List）
2. 文件系统
3. 子代理
4. 记忆系统

---

## DeepAgents：Agent Loop 的开箱即用实现

LangChain 团队的开源项目，设计初衷：逆向工程 Claude Code 的成功要素，然后将其开源化、通用化。

---

## LangChain 1.0 Middleware：Agent Loop 的扩展点

### 六大生命周期钩子

| 钩子 | 功能 |
|------|------|
| before_agent | Agent 调用开始时执行 |
| after_agent | Agent 调用结束时执行 |
| before_model | 每次 LLM 调用之前执行 |
| after_model | 模型返回之后执行 |
| wrap_model_call | 包裹整个模型调用过程 |
| wrap_tool_call | 包裹单个工具调用 |

### 13 个内置 Middleware

- SummarizationMiddleware
- HumanInTheLoopMiddleware
- ContextEditingMiddleware
- TodoListMiddleware
- ModelFallbackMiddleware
- ToolRetryMiddleware
- 等等...

---

## Claude Code Agent Loop 的创新

1. **1,300 行的 Query Engine**
2. **三层自愈式记忆架构**
3. **KAIROS——自治守护模式**
4. **Anti-Distillation 防蒸馏机制**
5. **游戏引擎级别的终端渲染**
6. **ULTRAPLAN——云端规划卸载**

---

## 总结

> Agent Loop 的进化，本质上就是 Context Engineering 的进化。


## 📷 文档图片

![img_D0Jzd84c6oxHBgxN82VcfRMVndh_0_1024_559.png](assets/Harness 101/Harness 101 - 从 for 循环到自治系统的进化之路/img_D0Jzd84c6oxHBgxN82VcfRMVndh_0_1024_559.png)
