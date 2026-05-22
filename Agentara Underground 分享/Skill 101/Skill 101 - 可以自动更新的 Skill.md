# Skill 101：可以自动更新的 Skill

> 来源: 飞书云文档 (KXjqduG93oO0fqxRm1ScjIvonHf)

---

## 前言

> 可以自动更新的 Skill，最好不要把"自动更新逻辑"写进 Skill 自己。

---

## 问题：Skill 会变成缓存文档

三个后果：
1. 本地副本会静默变旧
2. 错误会被稳定复现
3. 多机器同步更麻烦

---

## 反模式：让 Skill 自己检查更新

问题：
- Token 税
- 时机不对（用户已进入任务）
- 职责混乱

---

## 更好的位置：SessionStart Hook

```json
{
  "hooks": {
    "SessionStart": [
      {
        "type": "command",
        "command": "npx skills update -g -y 2>/dev/null"
      }
    ]
  }
}
```

---

## 安装器：把 Skill 当 Package 管

```bash
npx skills add jdevalk/skills --skill astro-seo
npx skills update
```

---

## 维护者怎么做

1. 把主分支当发布渠道
2. 给每个 Skill 写 README
3. 保持 SKILL.md 干净

---

## 结语

> Skill 负责描述能力；CLI 负责安装与更新；Hook 负责选择更新时机；Harness 负责在读取前准备好环境。
