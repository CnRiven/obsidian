# Harness 101：模型之外的全部

> 来源: 飞书云文档 (KIYJdSjqoodE93xsl2AcOBFonvd)

---

## 前言

> Agent = Model + Harness. If you're not the model, you're the harness.

Viv Trivedy 团队最近做了一件让人看着不可思议的事——他们没换模型，只换了 harness，把一个 Coding Agent 在 Terminal Bench 2.0 上的排名从 Top 30 直接拉到了 Top 5。

---

## 框架

> A coding agent is the model plus everything you build around it.

Harness 包含六类：
1. **指令层**：System Prompt、CLAUDE.md、Skill 文件、Sub-Agent 角色 Prompt
2. **工具层**：Tool 定义与 description、MCP Server 接入与过滤
3. **运行环境**：文件系统、sandbox、浏览器、Shell 与权限模型
4. **编排逻辑**：Sub-Agent spawn、handoff 协议、Model Routing
5. **确定性注入**：Hook、Middleware、PreToolUse/PostToolUse
6. **可观测性**：日志、Trace、Token 与 Cost 计量

---

## 错觉

> It's not a model problem. It's a configuration problem.

当你觉得「模型今天怎么这么笨」，回头去查 CLAUDE.md 与 Hook，大约 80% 的情况会被自己的配置打脸。

---

## 棘轮（Ratchet）

> Every line in a good AGENTS.md should be traceable back to a specific thing that went wrong.

- **加规则**：只在真实失败发生时入场
- **减规则**：下一代模型出来后，挑几条旧规则压力测试

---

## 解剖：五块组件

1. **Filesystem 与 Git**：持久化记忆
2. **Bash 与代码执行**：通用工具
3. **Sandbox 与默认工具链**：安全隔离
4. **Memory 与 Search**：跨会话与跨训练截止日
5. **Hooks**：纪律的工程化

> 成功静默，失败喧哗。

---

## 长程（Long-Horizon）

### Context 战线
- **Compaction**：压缩老 context
- **Tool-call Offloading**：超大输出写磁盘
- **Skills with Progressive Disclosure**：按需加载能力

### 调度战线
- **Ralph Loop**：用 Hook 拦截模型的 exit attempt
- **Plan-then-Act**：模型先把目标拆成 plan 文件
- **Planner/Generator/Evaluator 分离**

---

## 收敛

> Harnesses don't shrink, they move.

---

## 结语

> Every component in a harness encodes an assumption about what the model can't do on its own.

模型由实验室推动，harness 由我们推动。


## 📷 文档图片

![img_KIYJdSjqoodE93xsl2AcOBFonvd_0_1280_548.png](../assets/Harness 101/Harness 101 - 模型之外的全部/img_KIYJdSjqoodE93xsl2AcOBFonvd_0_1280_548.png)
