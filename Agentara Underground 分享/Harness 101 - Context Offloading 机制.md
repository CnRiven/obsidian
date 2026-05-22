# Harness 101：Context Offloading 机制

> 来源: 飞书云文档 (BOULd9TfhoPyfdxFtRrcVP1WnCd)

---

## 前言

一个 Agent 挂着 bash、read_file、write_file 几件趁手的工具。某一轮 bash 吐出 1k 的 Token，Harness 默默把它写到本地磁盘上，返回里只留一句"已保存在 abc123.log"。下一轮 Agent 想看怎么办？调一次 read_file，又把 1k Token 原封不动地拉回 Context 里。一出一进、数量相等，还多了一次工具调用。账面上这笔买卖是亏的。

**Harness 这套 Context Offloading 机制到底图什么？**

---

## 错觉

单 turn 视角下，offload 确实不省 Token。

但换一个镜头：一个长程 Agent 任务动辄跑上 30、50 轮。第 3 轮的 bash 输出在第 4 轮被 read_file 拉回来、模型基于它生成了一段下游决策，然后呢？它在第 5、6、7……一直到第 50 轮里继续存在的意义是什么？**几乎没有意义。**

> 把永久占用换成可回收占用。

---

## 生命周期：两个独立机制

### 机制 A — 生成时主动卸载

per tool-call 触发，发生在 tool_result 生成的那一刻。**Preemptive 的，出厂即引用。**

Agent 看到的 tool_result：
```json
{
  "role": "tool",
  "tool_call_id": "call_042",
  "content": "[OFFLOADED] saved to abc123.log (1024 tokens). Use read_file to retrieve."
}
```

### 机制 B — Context Compression 时的历史回收

per compression 触发，发生在 Context 接近上限时。**Retroactive 的——事后改写历史。**

---

## 差异：args 与 results 的非对称性

| 工具 | args 大小 | results 大小 | 是否幂等 | compression 处理策略 |
|------|----------|-------------|---------|---------------------|
| bash | 小 | 大 | 否 | args 不动；result 早在机制 A 被写盘 |
| web_fetch | 小 | 大 | 否 | args 不动；result 早在机制 A 被写盘 |
| read_file | 小 | 大 | 是 | args 不动；result 直接重写为引用 |
| write_file | 大(content) | 极小(ok) | 否 | args 重写为引用；result 原本就小，不动 |
| ls/grep | 小 | 通常小 | 是 | 通常保留原样 |

---

## 阈值

### 主流系统的真实阈值

| 系统 | 触发点 | 200k 模型对应 Token | 说明 |
|------|--------|-------------------|------|
| Claude Code | ~92% | ~184k | 接近上限才动手 |
| Cline/Roo Code | 默认 75% | ~150k | 偏保守 |
| Cursor Agent | ~80% | ~160k | 中等保守 |
| Manus | KV cache 命中优先 | ~180k+ | append-only 设计 |

### 多级阈值设计

| 等级 | 阈值 | 占比 | 动作 |
|------|------|------|------|
| Notice | 140k | 70% | 内部记账，标记"压缩候选" |
| Soft compact | 160k | 80% | 选择性 offload |
| Hard compact | 180k | 90% | 全量 summarize 老对话段 |
| Emergency | 190k | 95% | 强制截断 |

### 三条工程建议

1. 主触发点放在 **85%**（~170k）
2. 区分 **hard limit** 和 **soft limit**
3. 不要均匀压缩，要**集中砍头部**

---

## 结语

> 这套机制几乎就是操作系统的虚拟内存 swap 的翻版。真正让机制净收益为正的，不是单次 swap 的效率，而是一个长期被忽视的事实——绝大部分被 swap 出去的页，永远不会被重新 swap-in。
