# Skill 101 - 当 HTML 替代 Markdown 作为输出

> 来源: 飞书云文档 (HZHVd6F5uoZfZQxJokPc75minEg)

---

​
📦
第三章有我做的 Skill 实现​
​
📦
第三章有我做的 Skill 实现​
📦
第三章有我做的 Skill 实现​
第三章有我做的 Skill 实现​
# 前言​
Thariq Shihipar 在《Using Claude Code: The Unreasonable Effectiveness of HTML》里那句「HTML is the new Markdown」很抓人，但它也容易把话题带偏。​
它像是在宣布一次格式换代：以前我们让 Agent 生成 Markdown，现在是不是该让它生成 HTML 了？Simon Willison 也在跟进文章里提醒，HTML 可以塞进 SVG、交互小组件和页内导航，解释不再只是往下滚的一条长文。​
笔者觉得真正有价值的部分，不是「Markdown 过时了」，而是它把一个新问题摆到了桌面上：Agent 的产物正在变复杂。当输出只是会议纪要、代码解释、读书摘要，Markdown 足够好。但当输出变成探索规划、代码审查、设计稿、交互原型、动态图表、报告，甚至一个自定义编辑器时，人需要的就不只是阅读，而是查看、筛选、点击、折叠、回写反馈。笔者最近在帮 HR 部门做一个报告的时候，深有体会：与其输出 Markdown 的静态报告，不如输出带简单 BI 功能的交互式报告来得效果好！​
所以本文不讨论格式宗教。我们讨论的是：当 Agent 从「写一段话」走向「交付一个可操作对象」时，HTML 为什么会自然变成它的交互层。​
---
![[img_HZHVd6F5uoZfZQxJokPc75minEg_0.png]]
