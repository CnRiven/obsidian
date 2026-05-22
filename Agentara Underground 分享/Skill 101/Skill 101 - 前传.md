# Skill 101：前传

> 来源: 飞书云文档 (EwK8dKfiEoTpjTxYGtEctS7Rn8f)

---

## 前言

> Prompt 没死，它正在演化成一种更高级的结构：Skill。

---

## 什么是 Skill

> Skill 是一份可复用的、带元数据的 Agent 能力包。

四个要素：
1. **可复用**：写一次，处处触发
2. **带元数据**：Frontmatter（name + description）
3. **能力包**：正文 + references/ + templates/ + scripts/
4. **按需加载**：Progressive Disclosure

---

## Skill 与 Prompt 的区别

| 维度 | Prompt | Skill |
|------|--------|-------|
| 触发方式 | 人工粘贴 | Agent 自主选择 |
| 生命周期 | 一次性 | 持久化 |
| 可复用性 | 靠复制粘贴 | 声明式引用 |
| 结构化程度 | 自由文本 | 强结构 |
| 发现机制 | 无 | Metadata 扫描 |
| 组合性 | 拼接字符串 | 嵌套、调用、继承 |

---

## Frontmatter：Skill 的身份证

```yaml
---
name: weekly-report-writer
description: Generate structured weekly reports from raw work logs.
---
```

---

## Progressive Disclosure：按需展开的能力结构

三层结构：
1. **Frontmatter**：名片（<100 Token）
2. **SKILL.md 正文**：真正的指令与步骤
3. **Bundled Resources**：references/、templates/、examples/、scripts/、sub-skills/

---

## Skill 能做什么，不能做什么

### 能做什么
- 把可复用工作流封装成能力包
- 注入领域知识
- 保证特定格式的产出
- 实现多 Skill 组合

### 不能做什么
- 不能让弱模型变强
- 不能替代真正的 Tool Use
- 不适合解决确定性问题
- 不能保证 100% 触发
- 不是越多越好

---

## 四个主角的象限地图

- **Prompt**：一次性、随机应变
- **Skill**：可被自动发现并复用的指令层
- **Tool**：确定性高
- **Script**：确定性和复用性都是顶格

---

## 结语

> 把你手边那个最常用的"大 Prompt"拆成一个 Skill，看看会发生什么。
