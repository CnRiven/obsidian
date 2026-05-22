# DeerFlow: 从DeepResearch到SuperAgent Harness

> 来源: 飞书云文档 (QwJRdu1FVokoI1xjwvJcS7Loned)
> GitHub: https://github.com/bytedance/deer-flow
> 官方网站: https://deerflow.tech

---

## DeerFlow 的前世今生

### LangManus

DeerFlow 的前身是 LangManus——一款致敬 Manus 的开源项目。2025 年 3 月初，Manus 横空出世，团队在 5 天内复刻了一款带 Multi-Agent 架构、Deep Research 和 Browser-Use 功能的开源软件。开源短短 3 天，Star 数冲到 5,000，但因商标问题不得不关闭。

### DeerFlow 1.0

2025 年 4 月推出，7 天内突破 10k Star。Deer 是 Deep Exploration and Efficient Research 的缩写。

### DeerFlow 2.0

2026 年 2 月 14 日（情人节）发布，2月28日登上 GitHub Trending 榜首。

**数据一览：**
- Stars: 63k+
- Forks: 8k+
- Contributors: 240+
- Commits: 2000+

---

## 全新功能

### 技能（Skills）

Skill 是 Agent 能力的核心扩展单元。出厂自带 Skills：

| Skill | 说明 |
|-------|------|
| deep-research | 深度研究 |
| data-analysis | 基于 AI Coding 的数据分析 |
| chart-visualization | 数据可视化图表生成 |
| web-design-guidelines | 网页设计与生成 |
| image-generation | AI 图片生成 |
| video-generation | AI 视频生成 |
| podcast-generation | 播客生成 |
| ppt-generation | PPT 幻灯片生成 |
| github-deep-research | GitHub 仓库深度分析 |
| skill-creator | 用 AI 创建新的 Skill |

### 沙箱（Sandbox）

每个任务运行在隔离的沙箱环境中，支持三种运行模式：
- **Local**：直接在宿主机上执行
- **Docker**（推荐）：每个任务运行在独立容器中
- **Kubernetes**：通过 Provisioner 服务在 K8s Pod 中执行

### 子智能体（Sub-agent）

DeerFlow 2.0 采用 Claude Code 中的 Sub-agent-based Multi-Agent 架构：

- **general-purpose**：通用型，继承父 Agent 的所有工具，最多 50 轮对话
- **bash**：命令行专家，只配备沙箱工具，最多 30 轮对话

Sub-agent 最多 3 个并行执行，15 分钟超时。

### 上下文工程（Context Engineering）

- 多层 Middleware 链：11 个 Middleware 各司其职
- 上下文摘要：自动对较早的消息进行摘要压缩
- 子 Agent 上下文隔离
- 文件系统作为外部存储

### 长期记忆（Long-term Memory）

分为三个区块：
- 用户画像
- 时间线
- 事实库（含置信度评分）

### 消息网关（Message Gateway）

支持飞书、Slack 和 Telegram 等 IM 应用。

---

## 全新架构

### DeerFlow 1.0 vs 2.0

| 维度 | 1.0 | 2.0 |
|------|-----|-----|
| 架构模式 | Hand-off 风格 Supervisor-based | 自适应架构 |
| Agent 数量 | 5 个固定角色 | 1 个 Lead Agent + N 个动态 Sub-agent |
| 能力扩展 | 改图、加节点 | 加 Skill 或 Tool |
| 上下文管理 | 全局共享 State | Sub-agent 上下文隔离 + 摘要压缩 |
| 执行环境 | Python REPL | 完整沙箱 |
| 记忆 | 无 | 跨会话长期记忆 |

### Middleware 链

| 中间件 | 功能描述 |
|--------|----------|
| ThreadData | 创建线程目录结构 |
| Uploads | 注入新上传的文件 |
| Sandbox | 分配沙箱 |
| DanglingToolCall | 修补中断的工具调用 |
| ContextSummarization | 上下文摘要 |
| TodoListPlanning | TodoList 计划模式 |
| TitleGeneration | 自动生成会话标题 |
| AsyncMemoryUpdate | 异步记忆更新 |
| ImageInjection | 图像注入 |
| Sub-agentRateLimit | Sub-agent 并发限流 |
| Clarification | 用户澄清拦截 |

---

## 结语

DeerFlow 2.0 是写给这个 AI 时代的第二封情书。

GitHub: https://github.com/bytedance/deer-flow
