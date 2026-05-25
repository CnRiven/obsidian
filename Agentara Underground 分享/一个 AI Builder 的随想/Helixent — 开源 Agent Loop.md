# Helixent — 开源 Agent Loop

> 来源: 飞书云文档 (BKRSd4vudo7K2WxcbVqcdGnzn9c)
> GitHub: https://github.com/Agentara/helixent

---

Helixent 是一个基于 Bun 技术栈构建 ReAct-style Agent Loop 的轻量级库。

> 点击上面的卡片直达 GitHub 链接，欢迎大家 Star 和 Fork。

## 什么是 Agent Loop？

Agent Loop 的本质可以用一句话概括：**LLM 在一个循环中反复调用工具，直到它决定停下来。** 这就是所谓的 ReAct（Reasoning + Acting）架构。

## 架构

Helixent 分为三层，另有一个 community 区域用于第三方集成。

```
src/               # 源代码
├── foundation/    # Layer 1 – 核心原语
├── agent/         # Layer 2 – Agent Loop
├── coding/        # Layer 3 – Coding Agent（领域特化）
└── community/     # 第三方集成（如 OpenAI）
skills/            # 内置的 Skills
```

### Layer 1: Foundation

所有上层模块依赖的核心原语：

- **Model** — 对 LLM Provider 的统一抽象。定义一次模型，切换 Provider 无需改动 Agent 代码。目前仅支持 OpenAI 兼容的模型网关，将来会支持 Anthropic 的网关。
- **Message** — 贯穿整个系统的单一对话记录类型，是会话上下文的唯一数据源。
- **Tool** — 工具定义与执行管道（Agent 可以调用的"动作"）。

### Layer 2: Agent Loop

一个可复用的 ReAct-style Agent Loop：

- 在对话记录上维护状态。
- 以循环方式编排 "think → act → observe" 步骤。
- 并行调用工具，并将结果反馈到下一轮推理步骤中。
- 支持 Middleware 扩展行为（见下文）。
- 内置了 Skills Middleware，支持标准的 Agent Skills。

该层仅依赖 Foundation，保持通用性——不绑定任何特定领域。

### Layer 3: Coding Agent

基于通用 Agent Loop 构建的领域特化 Agent，预配置了面向编码的工具（read_file、write_file、str_replace、bash 等）以及 Skills Middleware。

### Community

可选的、解耦的适配器，为特定 Provider 实现 Foundation 接口：

- `community/openai` — 基于 openai SDK 的 OpenAIModelProvider，兼容任何 OpenAI 兼容的 API 端点。

---

## 快速上手：从零搭建一个 Coding Agent

以下是一个使用 OpenAI 兼容 Provider 创建 Coding Agent 的完整示例：

```typescript
import { createCodingAgent } from "helixent/coding";
import { OpenAIModelProvider } from "helixent/community/openai";
import { Model } from "helixent/foundation";

// 1. 配置 Model Provider（任何 OpenAI 兼容的端点均可）
const provider = new OpenAIModelProvider({
  baseURL: "https://api.openai.com/v1",
  apiKey: process.env.OPENAI_API_KEY,
});

// 2. 创建 Model 实例并设置参数
const model = new Model("gpt-4o", provider, {
  max_tokens: 16 * 1024,
  thinking: { type: "enabled" },
});

// 3. 创建 Agent — 工具和 Skills 会自动装配
const agent = await createCodingAgent({ model });

// 4. 以流式方式获取 Agent 的响应
const stream = await agent.stream({
  role: "user",
  content: [{ type: "text", text: "在当前目录创建一个 hello world Web 服务器。" }],
});

for await (const message of stream) {
  for (const content of message.content) {
    if (content.type === "thinking" && content.thinking) {
      console.info("💡", content.thinking);
    } else if (content.type === "text" && content.text) {
      console.info(content.text);
    } else if (content.type === "tool_use") {
      console.info("🔧", content.name, content.input.description ?? "");
    }
  }
}
```

就这么简单——四步搭建一个可用的 Coding Agent。

---

## Middleware

Helixent 提供了一套 Middleware 系统（类似于 Hooks），让你可以在 Agent Loop 的每个阶段观察和修改 Agent 的行为。Middleware Hook 按数组顺序依次调用。

### 可用的 Hook

| Hook | 触发时机 |
|------|----------|
| beforeAgentRun | 用户消息追加后、第一步开始前，执行一次 |
| afterAgentRun | Agent 即将停止时（无工具调用），执行一次 |
| beforeAgentStep | 每步开始时，模型调用前 |
| afterAgentStep | 每步结束时，所有工具调用完成后 |
| beforeModel | Model Context 发送给 Provider 前 |
| beforeToolUse | 工具调用前 |
| afterToolUse | 工具调用完成后 |

每个 Hook 接收当前上下文，可以返回一个部分更新合并回上下文，或返回 void 表示不做修改。

### 示例：Skills Middleware

内置的 Skills Middleware 是 Middleware 实际应用的一个好例子。它做两件事：

1. **beforeAgentRun** — 扫描 Skill 目录下的 SKILL.md 文件，解析 Frontmatter，并将发现的 Skills 挂载到 Agent Context 上。
2. **beforeModel** — 向模型 Prompt 中注入 `<skill_system>` 块，让 LLM 知道哪些 Skills 可用，并按需加载。

Skills 是存放在 Skills 目录下各文件夹中的 Markdown Playbook。每个 Skill 包含一个带有 YAML Frontmatter（name、description）的 SKILL.md，模型在匹配到用户任务时通过 read_file 按需加载完整内容。

---

## 为什么选择 Bun？

Bun 在去年年底被 Anthropic 收购，估值高达 10 亿美元，可以被视为 Anthropic 家的 Node.js。Agent Loop 本质上是异步的——模型思考、工具执行、结果回流，往往是并行进行的。JavaScript/TypeScript 将 `async/await` 原生内置于语言和运行时中，使并发编排变得简洁直观，无需像 Python 那样面对回调嵌套或 asyncio 的样板代码。

在 JS 运行时中，我们选择 Bun 的理由：

- **性能** — Bun 的 HTTP 客户端、文件 I/O 和启动速度显著快于 Node.js，原生支持 async，当 Agent Loop 在一次运行中执行数十次工具调用时，这一点至关重要。
- **独立可执行文件** — `bun build --compile` 可生成无外部依赖的单一二进制文件，使分发 CLI Agent 变得极为简单，终端用户无需安装运行时即可使用。
- **开箱即用** — 内置测试运行器、打包器和 TypeScript 支持，无需额外配置工具链。

作为 Anthropic 自家产品，Claude Code 也选用了 Bun 作为底层技术栈。

---

## 为什么不用 LangGraph

- Coding Agent 的核心已简化为纯粹的 Agent Loop，流程编排的复杂性被模型推理能力吸收
- LangGraph 的图式抽象更适合 Workflow，对 Agent 场景反而是多余的抽象层

---

## Roadmap

- **TODO List** — 内置任务追踪，让 Agent 能够规划、拆解并跟踪多步骤工作的进度。
- **Sub-agent** — 在运行中生成子 Agent 独立处理子任务，每个子 Agent 拥有自己的上下文和工具集。
- **Agent Team** — 多 Agent 协作，Agent 之间可以协调、委派和共享结果，合力解决复杂问题。
- **CLI** — 命令行界面层，支持从终端直接运行 Helixent Agent，提供交互式 I/O。
- **Print Mode** — 类似 Claude Code 的渲染模式，以丰富、友好的终端 UI 流式展示 Agent 的思考过程、工具调用和输出。
- **Sessioning** — 基于本地文件的会话存储，用于持久化 Agent 的上下文和历史记录。

最终目标是替代 Agentara 中的 Anthropic Claude Code，这样 Agentara 就可以成为 100% 自主研发的 Claw-like App 了。


## 📷 文档图片

![[img_BKRSd4vudo7K2WxcbVqcdGnzn9c_0_1024_559.png]]
