# Harness 101 - 从 for 循环到自治系统的进化之路

> 来源: 飞书云文档 (D0Jzd84c6oxHBgxN82VcfRMVndh)

---

​
🗻
2025-2026 年，AI Agent 领域最火的概念不是某个新模型，而是一个看似朴素的工程模式——Agent Loop。从 Harrison Chase 的 "In the Loop" 系列博文，到 LangChain 的 DeepAgents 开源项目，再到 Claude Code 泄露源码中曝光的 KAIROS 自治守护模式，Agent Loop 正在经历一场从"玩具级 for 循环"到"生产级自治系统"的范式跃迁。本文将从底层原理出发，拆解 Agent Loop 的核心机制、扩展手段，以及前沿工程实践。​
​
🗻
2025-2026 年，AI Agent 领域最火的概念不是某个新模型，而是一个看似朴素的工程模式——Agent Loop。从 Harrison Chase 的 "In the Loop" 系列博文，到 LangChain 的 DeepAgents 开源项目，再到 Claude Code 泄露源码中曝光的 KAIROS 自治守护模式，Agent Loop 正在经历一场从"玩具级 for 循环"到"生产级自治系统"的范式跃迁。本文将从底层原理出发，拆解 Agent Loop 的核心机制、扩展手段，以及前沿工程实践。​
🗻
2025-2026 年，AI Agent 领域最火的概念不是某个新模型，而是一个看似朴素的工程模式——Agent Loop。从 Harrison Chase 的 "In the Loop" 系列博文，到 LangChain 的 DeepAgents 开源项目，再到 Claude Code 泄露源码中曝光的 KAIROS 自治守护模式，Agent Loop 正在经历一场从"玩具级 for 循环"到"生产级自治系统"的范式跃迁。本文将从底层原理出发，拆解 Agent Loop 的核心机制、扩展手段，以及前沿工程实践。​
2025-2026 年，AI Agent 领域最火的概念不是某个新模型，而是一个看似朴素的工程模式——Agent Loop。从 Harrison Chase 的 "In the Loop" 系列博文，到 LangChain 的 DeepAgents 开源项目，再到 Claude Code 泄露源码中曝光的 KAIROS 自治守护模式，Agent Loop 正在经历一场从"玩具级 for 循环"到"生产级自治系统"的范式跃迁。本文将从底层原理出发，拆解 Agent Loop 的核心机制、扩展手段，以及前沿工程实践。​
# Agent Loop：一个被低估的核心抽象​
## 什么是 Agent Loop？​
Agent Loop 的本质可以用一句话概括：LLM 在一个循环中反复调用工具，直到它决定停下来。 这就是所谓的 ReAct（Reasoning + Acting）架构：​
![[img_D0Jzd84c6oxHBgxN82VcfRMVndh_0.png]]

用伪代码表示，Agent Loop 的最简形式大约是这样的：​