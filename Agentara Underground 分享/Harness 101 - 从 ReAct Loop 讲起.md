# Harness 101：从 ReAct Loop 讲起

> 来源: 飞书云文档 (Ivh2dHR5Gop0ttx0lD6cePSJnld)
> 录屏: 2026年4月20日

---

## 前言

现在打开任何一篇 Agent 相关的讨论，扑面而来的词都是 Multi-Agent、Deep Research、Skill、Sub-Agent、Orchestrator —— 一个比一个抽象，一个比一个唬人。但如果你把这些花哨东西的外壳一层层剥开，会发现最底下跑的其实是同一条 loop：**模型想一下，调个 Tool，看结果，再想一下** —— 一条最朴素的 ReAct Loop。

从「天气预报」这个最土的例子正着推。让模型调一次 get_weather，再让它连调两次，再让它自己写 plan，再把它扔进 Coding Agent 的场景，最后让它面对装不下的上下文和塞不下的能力。每一步都只比上一步多一点点，但等走完五步，你再回头看 Deep Research、Claude Code、Skill，会发现它们都只是这一条线上的某个站点而已。

---

## ReAct 的来源

ReAct 出自 Yao et al. 2022 年的论文 *ReAct: Synergizing Reasoning and Acting in Language Models*。它首次提出把 Reasoning（推理）和 Acting（行动）交织起来，让模型一边想一边调外部工具而不是一口气想完再做。

论文在 ALFWorld 上相比基线绝对成功率提升 34%，在 WebShop 上提升 10%。

---

## 演化阶梯

每一级往上加的东西都极少：

1. **单轮**：模型调一次工具
2. **多轮**：让 loop 多转几圈
3. **Plan**：在 loop 外头多塞一条 TODO
4. **Coding Agent**：把 Tool 从 web_search 换成 read_file、bash
5. **Offloading 和 Skill**：解决「东西多到塞不进 Context Window」之后的两个不同方向的妥协

理解 harness 的关键不是记住每一级的名字，而是看清每一级是为了解决上一级剩下的什么问题。
