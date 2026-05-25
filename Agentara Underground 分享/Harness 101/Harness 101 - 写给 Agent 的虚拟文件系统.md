# Harness 101 - 写给 Agent 的虚拟文件系统

> 来源: 飞书云文档 (TP1edurd4oKu5DxP73Vc4Pb2nTa)

---

# 前言​
如今主流的 Harness / General Agent，底座几乎都是 Coding Agent。Cursor、Claude Code、Devin 抽掉「会写代码」这个能力，剩下的躯壳就薄得只剩一个对话框。底下挂着的，是清一色的 Sandbox——一个能跑 bash、能 npm install、能起一个浏览器的全功能盒子。​
但笔者最近在做几个并不需要执行代码的 General Agent 时，开始怀疑这套默认配置。一个写作助手，它需要的是读 Markdown、写 Markdown，外加翻一翻历史草稿；一个日程助手，它要的只是读日历、写日历；一个客服 Agent，干的不过是查工单库、回写状态。这些场景里，Sandbox 是个昂贵的副作用——启动慢、资源占用大、攻击面广，而 Agent 真正需要的，只是 I/O。​
于是问题就摆在桌面上：Harness 一定要有基于 Container 或 MicroVM 技术的 Sandbox 吗？能不能把 Sandbox 拆成两层？让需要执行代码的 Agent 用厚的那层，让只需要读写数据的 Agent 用薄的那层？​
## 来自 Linux 的启发​
答案其实在 Unix 老哲学里躺了四十年——「一切皆文件」。/dev/null、/proc/self/status、/sys/class/net/eth0/address 这些「看起来是文件，实际是接口」的东西，正是把异构 I/O 抹平成统一抽象的经典招式。Plan 9 把这套哲学推到极致，连网络连接都是 /net/tcp/clone 这样的文件。Agent 时代，我们不妨再来一次：把所有 Agent 需要触碰的世界——用户文档、企业知识库、日历、工单、Session 临时数据——都暴露成虚拟文件，让 Agent 用最朴素的 read / write / ls / grep 就能完成绝大多数工作。​
本文提议一个面向 General Agent 的轻量底座：AFS（Agent File System）。它不替代 Sandbox，而是从 Sandbox 里抽出 I/O 这层，让不需要执行能力的 Agent 不必再为一个完整的盒子买单。​
下面这张图大概说明了这两条路的差别：​
笔者的立场是：AFS 是 Agent I/O 的最小公分母，不是所有 Agent 都需要 bash。本文按背景、设计、协议、结语四章展开。​
​
🎁
你知道吗——/proc/self/status？​
这是 Linux 上「一切皆文件」最经典的一张名片。cat /proc/self/status 会吐出当前进程的实时状态：PID、内存占用、线程数、信号掩码……每一行都是内核数据结构的字段，但你只用了一个 cat。背后没有 syscall、没有 SDK、没有 schema 协商，只有一个看起来像文本的虚拟文件。这套抽象的妙处是：任何能 read 的程序，都自动成了运行时监控工具。​
​
🎁
你知道吗——/proc/self/status？​
这是 Linux 上「一切皆文件」最经典的一张名片。cat /proc/self/status 会吐出当前进程的实时状态：PID、内存占用、线程数、信号掩码……每一行都是内核数据结构的字段，但你只用了一个 cat。背后没有 syscall、没有 SDK、没有 schema 协商，只有一个看起来像文本的虚拟文件。这套抽象的妙处是：任何能 read 的程序，都自动成了运行时监控工具。​
🎁
你知道吗——/proc/self/status？​
这是 Linux 上「一切皆文件」最经典的一张名片。cat /proc/self/status 会吐出当前进程的实时状态：PID、内存占用、线程数、信号掩码……每一行都是内核数据结构的字段，但你只用了一个 cat。背后没有 syscall、没有 SDK、没有 schema 协商，只有一个看起来像文本的虚拟文件。这套抽象的妙处是：任何能 read 的程序，都自动成了运行时监控工具。​
你知道吗——/proc/self/status？​
这是 Linux 上「一切皆文件」最经典的一张名片。cat /proc/self/status 会吐出当前进程的实时状态：PID、内存占用、线程数、信号掩码……每一行都是内核数据结构的字段，但你只用了一个 cat。背后没有 syscall、没有 SDK、没有 schema 协商，只有一个看起来像文本的虚拟文件。这套抽象的妙处是：任何能 read 的程序，都自动成了运行时监控工具。​
![[img_TP1edurd4oKu5DxP73Vc4Pb2nTa_0.png]]
