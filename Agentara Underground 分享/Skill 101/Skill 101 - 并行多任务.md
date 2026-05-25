# Skill 101：并行多任务

> 来源: 飞书云文档 (DZIudT0vHoBpgdx6VvgckFJbn5c)

---

## 前言

> Cognition 说"不要搞多 Agent"是对的——他们的语境是代码生成。
> Anthropic 说"多 Agent 提升了 90%"也是对的——他们的语境是信息检索。

---

## 为什么复杂 Skill 要多任务并行

### 第一重优势：性能
- M1-Parallel 论文报告了 2.2 倍端到端加速
- Anthropic 内部评测：多 Agent 架构提升 90.2%

### 第二重优势：上下文隔离
- 注意力聚焦
- 降低"上下文污染"风险
- 延长有效工作时间

### 第三重优势：多样性
- LLM 采样的天然多样性变成资产
- 多条路径同时展开，总有一条能找到最优解

---

## 何时不该并行

1. 子任务之间存在强依赖
2. 子 Agent 之间需要"商量"
3. 上下文共享不充分

---

## 如何在 Skill 中提示并行

在 Frontmatter 里添加 `allowed-tools: Agent Task`，使用 "spawn" 关键词。

---

## 从 Single-Skill 到 Multi-Skill

把复杂子任务独立成子 Skill，通过 `use /{skill-name}` 调用。

---

## 给 Skill 设计者的三条建议

1. 显式声明并行能力
2. 为每个子任务提供足够具体的指令
3. 设计好 fan-out / fan-in 的接口


## 📷 文档图片

![[img_DZIudT0vHoBpgdx6VvgckFJbn5c_0_5500_3072.png]]
