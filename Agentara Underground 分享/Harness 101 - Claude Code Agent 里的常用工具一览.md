# Harness 101：Claude Code Agent 里的常用工具一览

> 来源: 飞书云文档 (J4RZdlUGWofvVyxnPRnccQEznNd)

---

## 背景：从 Messages API 到托管 Agent 平台

```javascript
// 创建子 Agents
const generalAgent = await client.beta.agents.create(...);
const codeRAGAgent = await client.beta.agents.create(...);

// 创建一个受托管的 Claude Agent
const leadAgent = await client.beta.agents.create({
    name: "Coding Assistant", // 给 agent 取一个名字
    model: "claude-opus-4-7", // 指定模型，否则为默认
    system: "You are a helpful coding agent.", // 指定 System prompt
    tools: [{ type: "agent_toolset_20260401" }], // 指定工具或工具集（Toolset）
    mcp_servers, // 指定 MCP 服务
    skills, // 指定本地或远程的 Skills
    callable_agents: [
      generalAgent,
      codeRAGAgent,
    ], // 为 leadAgent 添加可供编排（Orchestration）的子 Agent
    metadata, // 添加自定义元信息，如 template 等
});
```

2026 年 4 月，Anthropic 正式推出 Claude Managed Agents——一套完全托管的 Agent 运行时，开发者无需自己搭 Agent Loop、工具执行层、Sandbox 和上下文管理，只需声明 Agent 的行为和工具权限，剩下的交给平台。

其核心注册方式只有一行：

```json
{
  "type": "agent_toolset_20260401"
}
```

这个 type 会给 Agent 注入一套完整的内置工具集，也就是本文要深度拆解的主角。

---

## 一、工具全览

`agent_toolset_20260401` 共包含 8 个内置工具：

| 工具名 | 描述 | 需要权限 |
|--------|------|----------|
| bash | 在容器内的 shell 会话中执行命令 | ✅ |
| read | 读取本地文件系统中的文件 | ❌ |
| write | 将文件写入本地文件系统 | ✅ |
| edit | 对文件执行精准字符串替换 | ✅ |
| glob | 使用 glob 模式快速匹配文件名 | ❌ |
| grep | 使用正则表达式搜索文件内容 | ❌ |
| web_fetch | 抓取指定 URL 的页面内容 | ✅ |
| web_search | 在互联网上搜索信息 | ✅ |

打 ✅ 的工具需要显式权限确认（或在 permission_mode 中预批准），其余工具为只读类操作，默认允许执行。

### 注册与裁剪

默认启用全部工具。如果想只开放部分工具，可以用 `default_config` + `configs` 组合控制：

```json
{
  "type": "agent_toolset_20260401",
  "default_config": { "enabled": false },
  "configs": [
    { "name": "bash",  "enabled": true },
    { "name": "read",  "enabled": true },
    { "name": "write", "enabled": true }
  ]
}
```

反过来，如果想禁用某几个工具：

```json
{
  "type": "agent_toolset_20260401",
  "configs": [
    { "name": "web_fetch",  "enabled": false },
    { "name": "web_search", "enabled": false }
  ]
}
```

---

## 二、WebSearch — 互联网搜索

### 定位

让 Agent 能够在互联网上检索当前信息，突破模型知识截止日期的限制。适合需要实时数据的任务：最新文档、价格、新闻、API 变更等。

### 核心参数

```typescript
interface WebSearchInput {
  query: string;  // 搜索查询词，1-6 个词效果最好
}
```

### 使用特点

- 返回前 10 条高相关度搜索结果，每条包含标题、摘要、URL
- 搜索词越短越精准（避免使用 `-`、`site:` 等运算符）
- 通常与 `web_fetch` 配合使用：先搜到 URL，再抓取全文
- 不适合完全确定性的历史事实查询

---

## 三、WebFetch — 网页内容抓取

### 定位

抓取指定 URL 的完整页面内容，将 HTML 转为 Markdown，适合深度阅读长文、技术文档、API Reference 等。

### 核心参数

```typescript
interface WebFetchInput {
  url: string;    // 必须是完整 URL（含 https://）
  prompt: string; // 告诉 Claude 从这个页面中提取什么信息
}
```

### 注意事项

- 返回内容会缓存一段时间，不保证是最新版本
- PDF 文档以 base64 编码返回
- 搜索结果里的 URL 才能被 fetch；不能凭空猜测 URL 去 fetch

---

## 四、Glob — 文件模式匹配

### 定位

基于文件名 glob 模式快速定位文件，是代码库探索的首选工具——比 bash find 更语义化，比 grep 更轻量。

### 核心参数

```typescript
interface GlobInput {
  pattern: string;    // glob 模式，如 "**/*.ts"、"src/**/*.test.js"
  path?: string;      // 可选，搜索根目录（默认 cwd）
}
```

### 典型用法

```
**/*.py          → 所有 Python 文件
src/**/*.ts      → src 下所有 TypeScript 文件
**/CLAUDE.md     → 所有层级的 CLAUDE.md
*.{json,yaml}    → 根目录下的 JSON 和 YAML
```

---

## 五、Grep — 内容正则搜索

### 定位

在文件内容中搜索正则模式，底层基于 ripgrep，速度极快。适合找函数定义、追踪引用、定位错误日志。

### 核心参数

```typescript
interface GrepInput {
  pattern: string;           // 正则表达式
  path?: string;             // 搜索目录
  glob?: string;             // 文件类型过滤，如 "*.ts"
  output_mode?: "content" | "files_with_matches" | "count";
  "-A"?: number;             // 匹配行之后显示几行
  "-B"?: number;             // 匹配行之前显示几行
  "-C"?: number;             // 匹配行前后各显示几行
}
```

---

## 六、Read — 深度拆解

Read 是整个工具集里调用频率最高的工具。Agent 的每一次文件修改都必须先经过 Read，才能理解当前内容。

### 完整参数 Schema

```typescript
interface ReadTool {
  file_path: string;   // 必填：绝对路径
  offset?: number;     // 可选：从第几行开始读（1-indexed）
  limit?: number;      // 可选：最多读几行
}
```

对应的 JSON Schema：

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["file_path"],
  "additionalProperties": false,
  "properties": {
    "file_path": {
      "type": "string",
      "description": "The absolute path to the file to read"
    },
    "offset": {
      "type": "number",
      "description": "The line number to start reading from. 1-indexed."
    },
    "limit": {
      "type": "number",
      "description": "The number of lines to read."
    }
  }
}
```

### 行为与限制

| 特性 | 说明 |
|------|------|
| 默认读取量 | 前 2000 行（不指定 limit 时） |
| 行截断 | 单行超过 2000 字符时自动截断 |
| 输出格式 | `cat -n` 风格，每行前缀行号（`  1\t内容`） |
| 多模态支持 | 支持读取 PNG、JPG 等图片，以 Image 呈现 |
| Notebook 支持 | 读取 `.ipynb` 时自动展开所有 cell 及其输出 |
| 不存在的文件 | 不报异常，返回 error 信息；Agent 可据此判断文件是否存在 |

### 分片读取技巧

读大文件时，先读 schema/头部确定结构，再定向读关键区域：

```python
# 先看文件头部 50 行
Read(file_path="/project/src/agent.ts", limit=50)

# 再读第 200-300 行的具体实现
Read(file_path="/project/src/agent.ts", offset=200, limit=100)
```

### 重要限制

路径必须是绝对路径，不接受相对路径（如 `./src/main.ts` 会失败）。在 Managed Agents 的容器环境里，cwd 是会话启动时的工作目录，Agent 应始终使用完整路径。

---

## 七、Write — 深度拆解

Write 用于创建新文件或对已有文件进行完整覆写。Anthropic 的设计哲学是：Write 是重型操作，轻量修改应优先用 edit（详见下一节）。

### 完整参数 Schema

```typescript
interface WriteTool {
  file_path: string;  // 必填：绝对路径
  content: string;    // 必填：文件的完整内容
}
```

对应的 JSON Schema：

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["file_path", "content"],
  "additionalProperties": false,
  "properties": {
    "file_path": {
      "type": "string",
      "description": "The absolute path to the file to write (must be absolute, not relative)"
    },
    "content": {
      "type": "string",
      "description": "The content to write to the file"
    }
  }
}
```

### 返回值

```json
{
  "filePath": "/path/to/file.txt",
  "success": true
}
```

### 核心行为规则

| 规则 | 说明 |
|------|------|
| 覆盖行为 | 如果目标路径已有文件，直接覆盖，无警告 |
| 已有文件必须先 Read | 写已存在的文件前，Agent 被训练为先 Read，确保不丢失当前内容 |
| 新文件直接写 | 创建全新文件不需要 Read |
| 目录自动创建 | 父目录不存在时自动创建（行为同 `mkdir -p`） |

### Write vs Edit：如何选择？

这是 Agent 工具设计中最核心的一个 trade-off：

> 场景：500 行文件，只需改第 47 行的一个变量名
> - Write 方案：传输全部 500 行 → token 浪费 × 幻觉风险 ↑
> - Edit 方案：传输 old_string + new_string（约 2 行）→ 精准、高效、安全

**原则：**
- 创建新文件 → `write`
- 文件内容改动超过 50% → `write`（完整重写比零散 edit 更清晰）
- 精准的局部修改 → `edit`（推荐，减少幻觉）
- 修改已有文件的一行或几个函数 → `edit`

---

## 八、Edit — 深度拆解

Edit 是 Agent Engineering 里减少幻觉的核心武器。它强迫 Agent 明确指定要改哪段文字，而不是生成整个文件——这大幅降低了模型"顺手改了其他地方"的风险。平时我们在 Claude Code 里使用的最多的就是它。这个工具之前叫 `str_replace`。edit 工具还有一个变体叫 `multi_edit(edits: EditTool[])`，其实就是一次执行多个字符串替换的操作。

### 完整参数 Schema

```typescript
interface EditTool {
  file_path: string;     // 必填：绝对路径
  old_string: string;    // 必填：要替换的原文（必须精确匹配，含空格和缩进）
  new_string: string;    // 必填：替换后的新内容
  replace_all?: boolean; // 可选：默认 false；true 时替换所有匹配项
}
```

实际 tool_use block 示例（来自真实 LLM traffic 抓包）：

```json
{
  "id": "call_k0e18w45",
  "name": "Edit",
  "type": "tool_use",
  "input": {
    "file_path": "/project/src/agent.ts",
    "old_string": "const maxSteps = 10;",
    "new_string": "const maxSteps = 25;",
    "replace_all": false
  }
}
```

### 返回值

成功时：
```
Successfully replaced text at exactly one location.
```

失败时（无匹配）：
```
Error: old_string not found in file
```

失败时（多处匹配）：
```
Error: Found 3 matches for old_string. Use replace_all=true or provide a more unique string.
```

### 核心约束

| 规则 | 说明 |
|------|------|
| 覆盖行为 | 如果目标路径已有文件，直接覆盖，无警告 |
| 已有文件必须先 Read | 写已存在的文件前，Agent 被训练为先 Read，确保不丢失当前内容 |
| 新文件直接写 | 创建全新文件不需要 Read |
| 目录自动创建 | 父目录不存在时自动创建（行为同 `mkdir -p`） |

### 实战：replace_all 的场景

重命名一个变量：

```json
{
  "file_path": "/project/src/agent.ts",
  "old_string": "maxSteps",
  "new_string": "maxTurns",
  "replace_all": true
}
```

这等价于 `sed -i 's/maxSteps/maxTurns/g'`，但更安全——Edit 工具会在操作前做文件存在性验证。

### 为什么 Edit 比 Write 能减少幻觉？

当 Agent 用 Write 重写一个大文件时，它需要在 context 里重新生成所有内容，任何一个细节都可能悄悄发生变化。而 Edit 只要求模型输出两段文字（before/after），其余内容完全不动——这是一个 token 更少、风险更低、更可审计的操作。

这也是为什么 Claude Code 和 Agentara 都将 `str_replace/edit` 作为首选编辑策略。

---

## 九、Bash — 深度拆解

Bash 是整个工具集里能力最强、风险最高的工具，也是构建编码 Agent 的核心支柱。

### 核心参数 Schema

```typescript
interface BashTool {
  command: string;      // 必填：要执行的 shell 命令
  description?: string; // 可选：描述本次命令的意图（用于日志/权限判断）
  timeout?: number;     // 可选：超时时间（毫秒）
}
```

### 持久化行为（非常重要）

Bash 工具在一个**持久 shell 会话**中运行，有以下特殊行为：

| 属性 | 行为 |
|------|------|
| 工作目录 | 跨命令持久（`cd /project` 后，下一条命令仍在 `/project`） |
| 环境变量 | 不跨命令持久（`export FOO=bar` 对下一条命令不可见） |
| 后台进程 | 支持，但 Agent 无法感知后台进程的输出变化 |

这是一个常见陷阱：

```bash
# ✅ 正确：cd 和命令在同一条
cd /project && npm run build

# ❌ 错误：分两次调用，第二次不知道 cd 到了哪里
# 第一次：cd /project
# 第二次：npm run build  ← 可能在错误目录执行
```

**最佳实践：** 需要在特定目录执行时，总是用 `&&` 或 `;` 拼成一条命令，或使用绝对路径。

### 权限与安全

Bash 是需要显式权限的工具。在 Managed Agents 里，可以通过 configs 进行精细控制：

```json
{
  "type": "agent_toolset_20260401",
  "configs": [
    {
      "name": "bash",
      "enabled": true
    }
  ]
}
```

在 Agent SDK 里，可以用 `allowed_tools` 预批准特定命令：

```python
options = ClaudeAgentOptions(
    allowed_tools=["Bash(npm run *)", "Bash(git *)"],
    permission_mode="acceptEdits"
)
```

### 典型用法

```bash
# 运行测试
pytest tests/ -x -v

# 安装依赖
pip install -r requirements.txt

# Git 操作
git add . && git commit -m "feat: add agent loop"

# 查看进程
ps aux | grep node

# 检查端口
lsof -i :3000
```

### 注意事项

- 不要执行可能挂起等待输入的命令（如 `npm init` 不带 `-y`）
- 避免破坏性命令：`rm -rf`、`git push --force` 等需要额外确认
- 环境变量持久化可以通过 SessionStart hook 设置，或在 CLAUDE_ENV_FILE 里预配置

---

## 十、工具组合模式

工具单独用都是原语，组合起来才是 Agent 的真正能力。几个经典 pattern：

### 模式 1：探索 → 定位 → 修改 → 验证

```
Glob("**/*.ts")           → 找到所有 TypeScript 文件
Grep("class ReactAgent")  → 定位类定义所在文件
Read(file_path, offset)   → 阅读具体实现
Edit(old_string, new_string) → 精准修改
Bash("npx tsc --noEmit")  → 验证编译通过
```

### 模式 2：搜索 → 抓取 → 写入

```
WebSearch("anthropic managed agents API 2026")
WebFetch(url, "提取 API 参数 schema")
Write("/docs/api-notes.md", content)
```

### 模式 3：读取 → 分析 → 创建报告

```
Glob("**/*.py")
for each file: Read(file_path)  ← 并行执行
Write("/reports/analysis.md", aggregated_content)
```

---

## 十一、与 Messages API Text Editor Tool 的区别

很多人容易混淆这两套工具，做个清晰对比：

| 维度 | agent_toolset_20260401 (Read/Write/Edit) | text_editor_20250728 (str_replace_based_edit_tool) |
|------|------------------------------------------|---------------------------------------------------|
| 适用场景 | Managed Agents、Agent SDK | Messages API（普通 tool_use） |
| 工具执行方 | 平台/SDK 自动执行 | 开发者自己实现执行逻辑 |
| 文件读取 | read 工具（独立） | view command |
| 文件写入 | write 工具 | create command |
| 精准编辑 | edit（old_string/new_string） | str_replace（old_str/new_str） |
| 关键字段差异 | file_path | path |
| 行插入 | 通过 Bash 实现 | insert command（insert_line） |

**结论：** 如果你用 Managed Agents 或 Agent SDK，用 `agent_toolset_20260401` 里的 read/write/edit；如果你在 Messages API 里自己实现 tool loop，用 `text_editor_20250728`。

---

## 十二、小结

`agent_toolset_20260401` 是 Anthropic 把 Claude Code 的工具体系正式产品化的结果。八个工具覆盖了编码 Agent 的完整操作链路：

- **搜索 / 抓取** → WebSearch + WebFetch
- **文件探索** → Glob + Grep
- **文件读写** → Read + Write + Edit
- **命令执行** → Bash

其中 `Read → Edit` 是减少 Agent 幻觉的黄金组合——Read 确保 Agent 看到的是真实内容，Edit 确保 Agent 只改它说要改的地方。这个设计哲学和 Claude Code 内部的 `str_replace` 优先策略一脉相承。

对于正在构建自己的 Agent OS（如 Agentara）的团队，这套工具集是最值得对齐的参考实现——不是因为它最复杂，而是因为它的约束设计得最合理。

---

## 参考资料

- Claude Managed Agents Overview
- Managed Agents Tools Reference
- Claude Code Tools Reference
- Text Editor Tool（Messages API）
- Scaling Managed Agents - Anthropic Engineering Blog
- Building Agents with the Claude Agent SDK
