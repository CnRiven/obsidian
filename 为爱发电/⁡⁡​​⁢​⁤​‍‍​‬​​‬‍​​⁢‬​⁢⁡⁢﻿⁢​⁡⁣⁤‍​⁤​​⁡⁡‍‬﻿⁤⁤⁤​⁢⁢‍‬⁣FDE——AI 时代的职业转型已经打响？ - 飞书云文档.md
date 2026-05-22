---
title: "⁡⁡​​⁢​⁤​‍‍​‬​​‬‍​​⁢‬​⁢⁡⁢﻿⁢​⁡⁣⁤‍​⁤​​⁡⁡‍‬﻿⁤⁤⁤​⁢⁢‍‬⁣FDE——AI 时代的职业转型已经打响？ - 飞书云文档"
source: "https://my.feishu.cn/wiki/LiubwwOU4iqbS4kxn4EcikHGnTf"
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

- [🚀FDE——AI 时代的职业转型已经打响？](#ZXKQdsFPhoBObHxt87ocpYYdnGb)
- [1\. 前言](#doxcnsre7VvUcj9Cm4k3uGb5owb)
- [2\. 从 OpenAI 的 Deployment 团队说起](#doxcniszGt38YDJcXDwBppS2rid)
- [3\. FDE 不是麦肯锡：模型边界 vs 流程边界](#doxcngZy426TIkiESxaEnWd09Pb)
- [4\. FDE 不是软件外包：共同探索 vs 需求实现](#doxcnHBlBfEHvJatDM7bOLmKxlf)
- [5\. 国外 FDE 的两条根：Palantir 与新一代模型公司](#doxcnz639jcQYdXD20hgkEEAutd)
- [6\. 国内 FDE：从解决方案架构师到 AI 落地工程师](#doxcnce6cMppuzhhuQr6e8P1P8b)
- [6.1 两个本土前身](#doxcnCqVzHq3z2GuVvODaTjWlhb)
- [6.2 三个水土差异](#doxcnoZG2YDfhFgRx55ncynlQWh)
- [6.3 一个独特定位：内部 FDE](#doxcnaF4652yxJafRDv67WTRWYd)
- [7\. 谁适合做 FDE，谁不适合](#doxcn2jo7OgUoQGDvPmXjrpUBef)
- [7.1 适合 FDE 的五个特质](#doxcnZFCENr8VQuehufJa4BAxch)
- [7.2 不适合 FDE 的四类人](#doxcnBfW6iq8hjwWk3LnMOPvl0d)
- [8\. 结语：从超级个体到超级岗位](#doxcnsbJFKePbMDMln5FruwXZHf)

## 🚀FDE——AI 时代的职业转型已经打响？

Henry Li

正在生成 AI 速览...

本文讨论了 AI 时代职业转型背景下 FDE（Forward Deployed Engineer）岗位的情况，包括其起源、与传统咨询和软件外包的区别、国内外发展情况、适合与不适合的人群等。关键要点包括：

1.

FDE 岗位兴起：FDE 原本是 Palantir 圈子里的工种黑话，2026 年 5 月 OpenAI 成立 Deployment Company 并投资 40 亿美元，Anthropic 也在多地招聘 FDE，使其成为热门岗位。

2.

与传统咨询区别：传统咨询卖流程边界，交付 PPT 等；FDE 卖模型边界，交付跑通的代码等，且资产需每次跑闭环重新构建。

3.

与软件外包区别：外包接 SOW，做局部交付，按项目计费；FDE 接 mission，做端到端交付，KPI 是模型 token 长期消耗曲线，反馈回流到模型公司。

4.

国外 FDE 根源：有 Palantir 和新一代模型公司两条根脉，共同点是客户买的是解决问题的工程师和工具组合，差异在于前者偏 OPS 集成，后者偏模型能力落地。

5.

国内 FDE 情况：前身是云厂商解决方案架构师和 AI 创业公司新岗位，存在私有化部署、模型能力、付费意愿等水土差异，还有内部 FDE 模式。

6.

适合与不适合人群：适合不抗拒销售沟通、享受模糊地带等特质的人；不适合纯技术控、需 OKR 驱动等人群。

7.

未来展望：FDE 是新岗位形态，后续可能出现 Forward Deployed PM 等岗位，岗位结构将随工作流重新切分。

内容由 AI 生成

1.

前言

笔者最近一个月遇到了四位准备转型的朋友——前端、解决方案架构师、产品经理、传统算法工程师，背景、年龄、城市都不一样，却问了同一个英文缩写： [FDE](https://www.linkedin.com/jobs/view/%E8%B1%86%E5%8C%85ai%E5%A4%A7%E6%A8%A1%E5%9E%8Bfde%EF%BC%88forward-deployed-engineer%EF%BC%89-%E7%81%AB%E5%B1%B1%E6%96%B9%E8%88%9Fmaas-at-bytedance-4330374800/) 是不是值得我跳过去？

[FDE，全称](https://www.linkedin.com/jobs/view/%E8%B1%86%E5%8C%85ai%E5%A4%A7%E6%A8%A1%E5%9E%8Bfde%EF%BC%88forward-deployed-engineer%EF%BC%89-%E7%81%AB%E5%B1%B1%E6%96%B9%E8%88%9Fmaas-at-bytedance-4330374800/) [Forward Deployed Engineer](https://www.linkedin.com/jobs/view/%E8%B1%86%E5%8C%85ai%E5%A4%A7%E6%A8%A1%E5%9E%8Bfde%EF%BC%88forward-deployed-engineer%EF%BC%89-%E7%81%AB%E5%B1%B1%E6%96%B9%E8%88%9Fmaas-at-bytedance-4330374800/) 。它在两年前还是 Palantir 圈子里的一个工种黑话，今天已经悄悄变成猎头的开场白、招聘启事的高频岗位、以及社交媒体上“AI 时代最值钱岗位”的候选答案之一。OpenAI 在 2026 年 5 月直接以这个名字成立了 [Deployment Company](https://www.primeai.solutions/blog/openai-deployment-company-forward-deployed-engineers-uk) ，初始投资 40 亿美元，明确说要派工程师钻进客户的现场办公，进入客户的工作流里；Anthropic 的 Applied AI 团队也在四个时区同步招聘 FDE。这件事从圈内黑话变成显性词汇，只用了一年多一点。

笔者上一篇文章 [《致超级个体》](https://my.feishu.cn/wiki/AfvNwnZiEirKPSkmUz7cfDlYnM1) 讨论的是“人的发动机”——好奇心、自学、自驱、动手能力，如何在完整的 Closed-loop 里被激发出来。但人不是悬浮的，人要被一个具体的岗位坐标系接住。如果说超级个体是 AI 时代生产关系的“原料”，那 FDE 就是市场在这一年里长出的、最显性的一种“岗位形态”。

附件不支持打印

![飞书文档 - 图片](blob:https://my.feishu.cn/0ba7251c-b806-4aa3-b1c0-2aecfa5eae94)

评论（0）

跳转至首条评论

0 字

- 上传日志

- 联系客服

- 功能更新

- 帮助中心

- 效率指南