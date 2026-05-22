---
title: "‍‍⁡⁣⁣‬​⁡⁣‬‌⁤⁢⁢⁢‌​​‍​⁣﻿​﻿​⁢⁤‬⁢​⁡​⁤⁣﻿‍​⁤⁤​​﻿⁡‌‍Harness 101：从 ReAct Loop 讲起 - 飞书云文档"
source: "https://my.feishu.cn/wiki/N94ZwtUv4iVsUtkvWAYchNKWnhb"
author:
published:
created: 2026-05-22
description:
tags:
  - "clippings"
---
## 飞书云文档

Agentara Underground 分享

互联网公开

搜索

问问知识库

目录

## 👁️🗨️Harness 101：从 ReAct Loop 讲起

Agent Tara

Henry Li

4月20日创建

46204

正在生成 AI 速览...

本文讨论了 Harness 的核心概念、机制及其在不同场景下的应用，旨在深入浅出地介绍 Harness 中的各种概念。关键要点包括：

1.

ReAct Loop 基础：出自 2022 年论文，将推理和行动交织，让模型边想边调外部工具。论文在 ALFWorld 上相比基线绝对成功率提升 34%，在 WebShop 上提升 10%。

2.

多轮 Loop 与 Deep Research：单轮 ReAct 应对复杂问题不足，多轮 Loop 可处理开放式任务，由模型决定停机，能解决“一步不够用”问题，但缺乏战略。

3.

Plan-then-Act 策略：为解决纯 ReAct 多轮的结构性毛病，引入 write\_todos 工具，让模型先写计划再行动，同时打补丁确保计划的输入和输出稳定。

4.

Coding Agent 场景：更换工具后，利用 read\_file、write\_file、bash 三件工具，结合 ReAct Loop 和 Plan-then-Act，可实现修 bug 等任务，体现 harness 骨架通用，工具选型决定 Agent 形态。

5.

Context Offloading 机制：当 tool\_result 体积超阈值时，将原始数据写入本地磁盘，Context 留路径占位符，与 Compression 机制不同，优先使用。

6.

Skill 能力组织：每个 Skill 是磁盘文件夹，Harness 启动时扫描目录，将 Frontmatter 拼成清单注入 System Prompt，正文按需加载，解决能力爆炸问题。

7.

Harness 分层设计：由 ReAct Loop、Plan-then-Act、Nudge、Offloading、Skill 五层机制组成，每层解决上一层暴露的问题，可单独调整和替换。

内容由 AI 生成

2026年4月20日的录屏在这里：

<iframe src="https://larkcommunity.feishu.cn/minutes/embed/obcn9e9991868drx14v22am7?from=ccm" allowfullscreen="" allow="encrypted-media;local-network-access *;" frameborder="0"></iframe>

[👁️🗨️Harness 101：从 ReAct Loop 讲起](https://my.feishu.cn/docx/DvmPdngkMoGkjexZbBgcuFfQnrO#share-Op7KdtY3horNSQxs9mZczUCMnYc) 👈 点击此处领取教程配套材料

1.

前言

现在打开任何一篇 Agent 相关的讨论，扑面而来的词都是 Multi-Agent、Deep Research、Skill、Sub-Agent、Orchestrator —— 一个比一个抽象，一个比一个唬人。但如果你把这些花哨东西的外壳一层层剥开，会发现最底下跑的其实是同一条 loop：模型想一下，调个 Tool，看结果，再想一下 —— 一条最朴素的 ReAct Loop。

笔者最近在出差的路上，利用从上海到北京的火车时间，半 Vibe Coding 半古法的、从零做了一个开源的 Harness + Coding —— [Helixent](https://github.com/MagicCube/helixent) （现已由我和社区一起继续维护，欢迎 Star），用来给同事们讲解 Harness，试过好几种讲法，但是发现如果走来就看代码，沟通成本过高。从 Multi-Agent 倒推讲起，听众很快就迷失在调度图里；从 Claude Code 这种成熟产品切入，又容易被它的工程复杂度带偏。最后发现先不说代码，用 Playground 把 Harness 掰开来看本质，这种效果最好：

1

评论（45）

跳转至首条评论

吕晓文5月16日 11:28

👍

回复...

0 字

- 上传日志

- 联系客服

- 功能更新

- 帮助中心

- 效率指南

<iframe src="chrome-extension://mofcmpgnbnnlcdkfchnggdilcelpgegn/app.html?source=icon&amp;lang=zh-CN&amp;url=https://my.feishu.cn/wiki/OvLBwVuSkiCR1ik5wGEcBXZfnye"></iframe>