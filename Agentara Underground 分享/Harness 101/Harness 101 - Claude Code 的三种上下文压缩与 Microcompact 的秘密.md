# Harness 101 - Claude Code 的三种上下文压缩与 Microcompact 的秘密

> 来源: 飞书云文档 (TDHbdiMFUoXEXoxYsDVcNiXYn2f)

---

# 前言​
笔者最近在排查 Claude Code 的 cache hit 率时发现一件事——每一次 API 调用之前，Claude Code 都在悄悄删你的 messages 数组里的某些 tool_result。​
按常识，这件事不应该发生。Anthropic 的 Prompt Cache 是按前缀 hash 比对的，只要 messages 里任何一个字节变了，cache 前缀就断了、整条缓存清零、那个写在账单里 90% 的折扣会瞬间蒸发。但笔者反复核对 usage 字段，cache_read_input_tokens 纹丝不动，改 messages 没有破缓存。​
这件事的反直觉之处在于，它同时违反了两个我们以为铁板一块的假设——客户端不能改历史（改了 cache 就废）、服务端不会让你改历史（API 文档里也没写一个能改的入口）。但 Claude Code 做到了，而且每一次 API call 之前都在做。​
顺着这条线索往下挖，会发现这不是某个孤立的 trick，而是 Claude Code 三种 compact 机制中最便宜也最常被忽略的一种——Microcompact。它跑在每一次 API 调用之前，删旧的 tool_result、删上一轮之前的 thinking block，再借助 Anthropic 一个低调到几乎没人讨论的 API 字段 cache_edits 通知服务端“我改了，但请把缓存保留”。​
笔者打算把这一整套机制拆开看清楚。本文试图回答这几个问题：​
- •
- Claude Code 究竟有几种 compact 模式？它们的代价排序是什么？​
- •
- 为什么 Microcompact 敢在每一次调用前都跑一遍？它怎么保证不破坏对话语义？​
- •
- cache_edits 到底是什么 API，凭什么能做到“既改 messages 又不破 KV 缓存”？​
- •
- 哪些 tool_result 进得了 Microcompact 的候选池，哪些是动一下就会出 bug 的“高压线”？​
---
![[img_TDHbdiMFUoXEXoxYsDVcNiXYn2f_0.png]]
