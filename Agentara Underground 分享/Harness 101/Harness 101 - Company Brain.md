# Harness 101：Company Brain

> 来源: 飞书云文档 (O1Fudqn5QoaqUpxoX8IcfG87nOc)

---

## 前言

> 标题里那句「companies have data but no memory」很抓人。它把今天很多 AI Agent 项目里一个说不清、但人人都隐隐感觉不对劲的问题挑明了：**公司不是没有数据，公司是没有记忆。**

Company Brain 不是一个更聪明的企业搜索，也不是一个套在公司文档上的 chatbot，而是 **Agent 时代的组织记忆系统**。

---

## 公司为什么没有大脑

> Retrieval 回答的是：哪段文字可能相关？
> Memory 回答的是：这件事在组织里意味着什么？

同一句话，落到不同角色、不同时间、不同上下文里，含义完全不同。

---

## 三层记忆

### 1. Factual Memory（事实记忆）

至少要有这几个字段：
- **source**：这条事实来自哪里？
- **owner**：谁对它负责？
- **timestamp**：它是什么时候产生、什么时候被更新的？
- **permission**：谁能看原文？
- **freshness**：它现在还有效吗？
- **confidence**：系统对这条抽取的事实有多确定？
- **relationship**：它连接到哪些客户、项目、承诺、风险、决策？

### 2. Interaction Memory（交互记忆）

> 数据库记录的是 aftermath。真正的决定发生在对话里。

核心词：**ontology** — 系统用来理解世界的概念和关系。

### 3. Action Memory（行动记忆）

> Action Memory 不是「把过去的动作存起来」，而是组织对「什么情况下应该推进、等待、确认、升级、停止」的记忆。

---

## 实现

最小可行架构的关键选择：

1. **先建 Event Log** — 所有输入先作为 append-only event 存下来
2. **Extractor 输出 claim，而不是直接输出 truth**
3. **Retrieval 必须是 embedding + graph traversal**
4. **Policy Layer 不是上线前补的一层胶水**
5. **Action Router 要克制** — 从低风险动作开始

---

## 结语

> 未来真正的护城河，可能不是谁多写了几个 Agent，而是谁能从第一天开始，把自己的「为什么」保存下来。
