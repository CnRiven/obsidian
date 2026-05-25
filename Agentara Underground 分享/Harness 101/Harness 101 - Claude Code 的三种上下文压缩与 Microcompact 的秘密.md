# Harness 101：Claude Code 的三种上下文压缩与 Microcompact 的秘密

> 来源: 飞书云文档 (TDHbdiMFUoXEXoxYsDVcNiXYn2f)

---

## 前言

每一次 API 调用之前，Claude Code 都在悄悄删你的 messages 数组里的某些 tool_result。

---

## 三层级联

三种 compact 模式是一条 cost-ordered 的级联：

| 模式 | 触发时机 | 策略 | 代价 |
|------|----------|------|------|
| Microcompact | 每一次 API call 之前 | 删除旧的 tool_result + thinking block | 不通过 LLM，近乎为零 |
| autocompact | token 数 + tool_calls 数双阈值触发 | 把对话历史浓缩进本地 background notes 文件 | 通过 LLM 压缩 |
| fullcompact | 上面两层都没救住时 | fork 一个 sub-agent 跑完整 LLM 摘要 | 显著 |

---

## Microcompact

- **Zero-LLM**：纯规则的本地操作
- **只动 tool_result**，绝不动 user message 与 assistant text
- **八工具白名单**：Bash、Read、Grep、Glob、WebFetch、WebSearch、FileEdit、FileWrite

---

## cache_edits：服务器端的隐形橡皮擦

> 在已经建好的服务器端 cache 条目里，就地"挖空"指定的 tool_use/tool_result 占位，但不重新计算前缀 hash。

四条事实：
1. 本地 messages 列表不动
2. 前缀 hash 视角下视而不见
3. 只能动幂等可重建或低价值的 tool_result
4. cache break detector 会被显式通知

---

## Layer 1 vs Layer 2

- **Layer 1**：每次 API 调用前的热路径（走 cache_edits 协议）
- **Layer 2**：60 分钟冷启动后的本地路径（直接改写本地 messages）

---

## 结语

> Harness 不再只是 API 的消费者，它正在变成 API 演化的合谋者。


## 📷 文档图片

![img_TDHbdiMFUoXEXoxYsDVcNiXYn2f_0_3832_1642.png](assets/Harness 101/Harness 101 - Claude Code 的三种上下文压缩与 Microcompact 的秘密/img_TDHbdiMFUoXEXoxYsDVcNiXYn2f_0_3832_1642.png)
