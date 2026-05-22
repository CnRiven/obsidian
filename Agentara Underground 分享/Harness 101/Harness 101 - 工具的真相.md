# Harness 101：工具的真相

> 来源: 飞书云文档 (W2xtddNBIokpjmxMyhUcNExMnFh)

---

## 前言

> Tool 不是模型突然长出了手，也不是一段普通的自然语言注释。它更像是模型与 Harness 之间的一份契约。

---

## 先把 Tools 这件事说小一点

Tool 是一张**可调用能力说明书**。模型看不到函数源代码，能看到的只有：name、description、parameters/JSON Schema，以及 strict 这样的约束开关。

> 模型只看说明书，不看实现代码，更不帮你执行代码。

---

## Tool Definition 是一份写给模型的契约

四块内容：
1. **Name**：工具在模型心里的索引
2. **Description**：使用时机的说明
3. **Schema**：约束参数形状
4. **Result**：返回结果的设计

> strict: true 解决的是结构合规问题，不解决语义判断问题。

---

## 没有 Tools API 的 2023

用纯 Prompt 实现 Tool Calls：把工具清单写进 System Prompt，规定模型必须输出固定格式。

---

## 天气预报例子：三种调用

### 一次调用
模型决定该查东京，输出 tool_call，Harness 执行并回填结果。

### 并行调用
同一轮产出多个 tool_call，Harness 并发执行。

### 多轮调用
答案依赖上一步观察，模型边观察边改变行动。

---

## 工程上真正要调的是契约质量

调优原则：
- 工具名要像动作，不要像抽屉
- description 要写使用时机
- schema 要尽量收窄
- 返回值要为下一轮模型消费而设计
- 避免多个工具职责重叠

---

## 结语

> Tool 是模型与 Harness 之间的一份契约，不是模型自己的超能力。
