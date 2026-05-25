# Harness 101 - 模型之外的全部

> 来源: 飞书云文档 (KIYJdSjqoodE93xsl2AcOBFonvd)

---

# 前言​
Viv Trivedy 团队最近做了一件让人看着不可思议的事——他们没换模型，只换了 harness，把一个 Coding Agent 在Terminal Bench 2.0上的排名从 Top 30 直接拉到了 Top 5。同样的模型权重、同样的题面、同样的预算，差别只在「模型之外的所有东西」。如果你看该榜单，就知道这相当于跨过了好几代模型升级才能换到的位置。​
​
🎁
你知道吗——Terminal Bench 2.0？​
Terminal Bench 是 Stanford 与 Laude Institute 主导的端到端真实工作流基准，覆盖编译、训练、运维、调试等近百个真实任务，比 SWE-bench 更接近「日常开发」。它的 2.0 版本在题目上更难、在 harness 上更可观测，已经成了模型厂商发布稿里的常客。同模型不同 harness 跑分能差出几十个百分位——这是 harness 工程能撬动多大杠杆的最直接证据。​
​
🎁
你知道吗——Terminal Bench 2.0？​
Terminal Bench 是 Stanford 与 Laude Institute 主导的端到端真实工作流基准，覆盖编译、训练、运维、调试等近百个真实任务，比 SWE-bench 更接近「日常开发」。它的 2.0 版本在题目上更难、在 harness 上更可观测，已经成了模型厂商发布稿里的常客。同模型不同 harness 跑分能差出几十个百分位——这是 harness 工程能撬动多大杠杆的最直接证据。​
🎁
你知道吗——Terminal Bench 2.0？​
Terminal Bench 是 Stanford 与 Laude Institute 主导的端到端真实工作流基准，覆盖编译、训练、运维、调试等近百个真实任务，比 SWE-bench 更接近「日常开发」。它的 2.0 版本在题目上更难、在 harness 上更可观测，已经成了模型厂商发布稿里的常客。同模型不同 harness 跑分能差出几十个百分位——这是 harness 工程能撬动多大杠杆的最直接证据。​
你知道吗——Terminal Bench 2.0？​
Terminal Bench 是 Stanford 与 Laude Institute 主导的端到端真实工作流基准，覆盖编译、训练、运维、调试等近百个真实任务，比 SWE-bench 更接近「日常开发」。它的 2.0 版本在题目上更难、在 harness 上更可观测，已经成了模型厂商发布稿里的常客。同模型不同 harness 跑分能差出几十个百分位——这是 harness 工程能撬动多大杠杆的最直接证据。​
笔者最近在写 Harness 系列（见下文链接）：上一篇刚讲完《​
👁️‍🗨️
Harness 101：从 ReAct Loop 讲起》，再上一篇聊了​
🗜️
Harness 101：Context Offloading 机制。写下来一个观察是，一线工程师每天都在调 Skill、写 Hook、设计 Sub-Agent，把工具描述改了又改、把 CLAUDE.md 删了又添，但很少有人退后一步问一个问题：我们到底在工程化什么？这件事如果没有名字，就很难积累。​
4 月底读到了 Addy Osmani 那篇 Agent Harness Engineering，他给了一个超级简洁的等式：​
​
代码块​
Python
自动换行
复制
agent = models + harness​
​
代码块​
Python
自动换行
复制
agent = models + harness​
代码块​
Python
自动换行
复制
agent = models + harness​
本文是带着工程师立场的二次解读。笔者想用它回答三个你也许正在心里反复问自己的问题：​
1.
Harness 究竟是什么？这个词在小红书、司内的 ByteTech 上已经流行到我们不想再听到，然而它不是抽象概念，而是一份你今天就能列出清单的工程对象。​
2.
为什么大多数 Agent 失败不是模型问题？而你又凭什么相信「换 harness」比「等模型」更划算？​
3.
当模型一代比一代强，harness 工程师的价值往哪里走？这是个比想象中更长程的问题。​
接下来的八节，就围绕这三问展开。​
---
![[img_KIYJdSjqoodE93xsl2AcOBFonvd_0.png]]
