# Harness 101：从 Chatbot 到 Long-horizon Agent

> 来源: 飞书云文档 (B1mfdU21eonlkuxBA54c58bxnSb)

---

## 序言

Long-horizon Agent 是一种能够在更长时间跨度内自主运行的智能体——从几分钟到数小时不等。

---

## Long-horizon Agent 的核心特征

1. **运行时间更长**：不再是"调用一次 LLM 就结束"
2. **自主决策能力**：LLM 在循环中不断决定下一步行动
3. **产出"初稿"而非最终产品**：人类用户沿着初稿进行后续修改

---

## 典型应用场景

- **Coding**：Claude Code、Cursor 等
- **AI SRE**：深度日志分析和故障排查
- **Research & Report Generation**：Deep Research 类产品
- **高级客户支持**：后台运行生成完整上下文报告

---

## 什么是 Harness？

> LangGraph 是 runtime，LangChain 是 abstraction，Deep Agents 是 harness。

Harness 包含：
- 内置的 Planning Tool
- Compaction 机制
- File System 工具
- Starter Prompts
- Memory 系统
- Sub-agents 机制

---

## AI Agent 发展的三个时代

1. **第一阶段**：简单的 Prompting 和 Chaining
2. **第二阶段**：Cognitive Architecture 时代
3. **第三阶段**：Long-horizon Agent 时代（2025 年中至今）

---

## 结语

> 我们从 scaffolds 转向了 harnesses。


## 📷 文档图片

![[img_B1mfdU21eonlkuxBA54c58bxnSb_0_1280_714.png]]
