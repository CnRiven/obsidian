# Harness 101：Claude.AI 提示词与记忆结构解析

> 来源: 飞书云文档 (HKRzdO2YvodWa2xjNLWcJI3KnEA)

---

## 序：一次意外的自我披露

> 把 prompt 当零件装配，不要当写作。XML 标签不是装饰，是接口。

---

## 结构全景：Claude 的心智分层

Claude 的 system prompt 是一块**沉积岩**——不是精心设计的大厦，而是多个团队在不同时期把片段塞进了"最近的那个开放标签"。

---

## 十三个功能区

1. 行为总纲：`<claude_behavior>`
2. 记忆系统：`<memory_system>`
3. 记忆编辑工具：`<memory_user_edits_tool_guide>`
4. 计算机环境：`<computer_use>`
5. 可视化路由
6. 搜索规则：`<search_instructions>`
7. 图像搜索：`<using_image_search_tool>`
8. 过往对话检索：`<past_chats_tools>`
9. 用户偏好：`<preferences_info>`
10. 引用格式：`<citation_instructions>`
11. Artifacts 里调 API：`<anthropic_api_in_artifacts>`
12. 其他工具块
13. 环境与渲染

---

## 三条设计暗线

### 暗线一：Progressive Disclosure

> 不要一次性把所有东西塞进 context，按需加载。

三处体现：
- **Skill 系统**：metadata 在 system prompt，body 按需加载
- **记忆系统**：summary 驻留，原始对话靠工具取
- **工具发现**：tool_search 模式

### 暗线二：记忆系统的"防伪印章"

禁用措辞会暴露 RAG 机制：
- ~~"I can see..."~~
- ~~"According to your memories..."~~
- 直接说"对了，Agentara 最近那个集成进展怎么样？"

### 暗线三：Prompt 即防火墙

- 版权硬红线重复三遍
- 儿童安全的"脑补检测"
- 防 prompt injection

---

## 记忆系统的五个 section

| Section | 用途 | 更新频率 |
|---------|------|----------|
| Work context | 当前职业身份 | 低 |
| Personal context | 稳定个人背景 | 极低 |
| Top of mind | 当下最活跃事项 | 高 |
| Brief history | 过往对话活动史 | 中 |
| Other instructions | 显式偏好规则 | 即时 |

---

## XML 标签的四个能力

1. **分区可寻址**
2. **模块可替换**
3. **优先级可表达**
4. **反注入友好**

---

## 四个扩展点

1. **Prompt Fragment**：XML 零件
2. **Tool**：工具定义
3. **Skill**：延迟加载的指令集
4. **Hook**：生命周期拦截器

> 沉积岩不是缺陷。沉积岩是活过来的证据——而可被第三方贡献的沉积岩，才是长得出生态的土壤。
