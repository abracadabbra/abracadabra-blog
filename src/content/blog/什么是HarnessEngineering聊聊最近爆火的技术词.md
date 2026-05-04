---
title: '什么是Harness Engineering：聊聊最近爆火的技术词'
description: '2026年春天爆火的Harness Engineering到底是什么？本文深入解析它的起源、三种解释口径、与Prompt Engineering的关系，以及实际落地的八器官系统。'
pubDate: '2025-05-04'
tags: ['AI', 'Harness Engineering', 'Agent', '工程化']
categories: ['AI开发']
---

> 2026 年春天，一个英文单词在开发者社区里忽然火起来。一个月前还很少人提，一个月后已经出现在各平台技术帖和文章里。这个词就是 Harness Engineering。

各种说法之间口径不一，有时候还互相矛盾。顺着几篇不同作者的文章读下来，很容易越读越糊涂。本文不打算给这个词一个"最终定义"。一个术语为什么会在某个时刻出现，是谁提出来的，它想解决什么问题，它和已有概念的边界在哪里——这些问题搞清楚了，"它到底是什么"反而变成一个顺带能回答的小问题。

---

## 一、一个词是怎么出现的

讨论任何一个新术语，从时间线开始总是比较清醒的做法。

**2026年2月5日** - HashiCorp 的联合创始人 Mitchell Hashimoto 在个人博客上发表了一篇长文《My AI Adoption Journey》。他在文章里提出了一个说法："Engineer the harness"。

他的意思很朴素：每次发现 Agent 犯了一个错，就花时间设计一个解决方案，让它永远不再犯同样的错。这个思路听起来有点像反复调校一个实习生。Mitchell 的重点不在于训练模型，而在改造模型周围的环境。

**2026年2月11日** - OpenAI 发表了一篇工程博客《Harness Engineering: Leveraging Codex in an Agent-First World》，作者 Ryan Lopopolo。

这篇文章的分量要比 Mitchell 的博客重得多。它不是个人随笔，是 OpenAI 团队对自己如何在生产环境中使用 Codex 的经验总结。文章里列了一堆实际做法：AGENTS.md 怎么写，plan 目录如何组织，工具如何暴露。这篇文章让 harness engineering 第一次走出小圈子。

**2026年3月24日** - Anthropic 发表了自己的研究《Effective harnesses for long-running agents》。

Anthropic 讲的是另外一类问题：当你希望一个 Agent 连续工作很多个小时，跨越多个 session 完成一个复杂任务，harness 应该怎么设计才能让它不在中途崩掉。

Anthropic 文章里有一句话后来被很多人引用：

> "每一个 harness 组件都编码了一个关于'模型自身做不到什么'的假设，而这些假设是值得反复检验的：它们可能本来就是错的，也可能会随着模型的进步很快过时。"

**2026年3月31日** - 安全研究员 Chaofan Shou 发现 npm registry 上的 @anthropic-ai/claude-code 包存在一个构建配置失误，Claude Code 的完整源码通过 source map 泄露出来。

这件事在技术社区引起了巨大反响，披露推文获得了超过 1700 万次浏览。一夜之间，数不清的开发者涌进去阅读源码，逐行分析这套一直被视为"harness 设计典范"的系统到底是怎么构造的。

---

## 二、补考：其实它并不完全是 2026 年的发明

早在 **2025年3月**，知名开发者 swyx 在一系列文章里提出过相似的说法。他当时叫它 "agent engineering"，并配了一个叫 **IMPACT** 的框架。

| 维度 | 含义 |
|:---|:---|
| Intent | 用户意图和表达 |
| Memory | 跨 session 的记忆 |
| Planning | 任务拆解和规划 |
| Authority | 自主决策和用户介入的边界 |
| Control Flow | 由 LLM 驱动的控制流 |
| Tools | Agent 能调用的工具集 |

把这个框架和 2026 年那几篇 harness engineering 的核心议题放在一起看，会发现几乎完全重合���

严格来说，harness engineering 不是哪个人或哪家公司的发明。它是社区在几乎同一段时间，从不同方向逼近同一个问题之后，自然形成的一个收敛点。

---

## 三、三种解释口径

看市面上各种关于 harness engineering 的解读，可以归为三派。它们背后对"模型应该被怎样对待"这个问题有相当不同的立场。

### 1. 马具派

Harness 这个英文单词本意就是马具或者挽具。Mitchell 说 "engineer the harness" 的时候，字面上就能翻译成"好好打造你的那套马具"。

这一派认为：模型像一匹好马，力气大但不守规矩，harness 是给它穿上的装备，让骑手能够驾驭它。赤兔马，在会骑马的人手里是战神，在不会骑马的人手里是摔伤事故。

这个口径直观好懂，适合做比喻。它的问题在隐喻本身带了一层被动意味。

### 2. 工作空间派

这一派对"马具"那层被动意味有直接的反对意见。

**核心主张**：harness 的目的不是限制模型能做什么，而是创造条件让模型能做到原本做不到的事。

这个视角下，harness engineering 的意义在于定义协作边界和协作协议，让模型能在一个稳定、可交互、可反馈的环境里持续工作。

### 3. 约束执行派

这一派在关于 Claude Code 源码的系统性研究里表达得最充分。

**起点是一个冷峻的判断**：模型是不稳定部件，甚至是整个系统里最不稳定的部件。一旦它能接触 shell、Git、网络和本地文件，问题就从"它答得好不好"变成"它执行留下的后果怎么收拾"。

从这个视角出发，harness engineering 的意义就很直接：**一整套制度化的控制平面，用来处理一个现实问题——模型并不天然值得信任。**

> 代理系统的关键能力是约束执行。

三派的分歧值得额外说一下：它们并不矛盾，只是重心不同。

- 写教程用马具派的语言更容易开场
- 创业公司用工作空间派的语言更能向同事解释自己在做什么
- 在大厂负责生产系统的团队负责人用约束执行派的语言更能说服管理层

如果一定要选一种作为默认理解，约束执行派大概是最安全的那种。

---

## 四、它和 prompt engineering、context engineering 是什么关系

一种流传比较广的说法是把这三个词视为一个递进阶梯。

| 阶段 | 时间 | 关注点 |
|:---|:---|:---|
| Prompt Engineering | 早期 | 单次对话里怎么措辞 |
| Context Engineering | 中期 | 整个 context window 怎么拼 |
| Harness Engineering | 后期 | Agent 的整个运行环境 |

> 模型越强，问题就越往外移。模型很弱的时候，写好一句 prompt 就能大幅改善结果；模型已经很强的时候，决定成败的不是模型本身，是包裹模型的那一整套工程结构。

---

**社区里还流行一个简短的公式**：

```
Agent = Model + Harness
```

按照这个公式，一个裸模型只是一个概率生成器，被 harness 包裹之后才能称作 Agent。

**同一个 Claude 4.6，放在 Claude Code 里和放在一个简陋脚手架里，行为表现可以差得非常远。差距不在权重，在 harness。**

---

## 五、harness 的"器官系统"

一个 harness 具体包含哪些组件？

### 1. System Prompt 与指令分层

这不是传统意义上那种"你是一个有帮助的助手"式的人格设定，是一套分层的运行时规章。

Claude Code 源码里可以看到，它的 system prompt 被拆成多段：身份与总任务在一段，工具与权限说明在另一段，工程约束又在另一段。

### 2. Query Loop（运行时主循环）

一个 Agent 不是"请求-响应"式的问答系统，它是一段持续运行的循环。Claude Code 的核心不是某一次 API 调用，���一��带状态的循环体：messages、tool use context、compact tracking、turn count。

### 3. 工具系统

工具包括本地函数调用、Skills、MCP server、外部 API。在 harness 的视角下，工具不是模型能力的自然延伸，是需要被调度、被授权、被限制并发、被审计的受管执行单元。

### 4. 上下文治理

它包括工作记忆、长期记忆、压缩策略、会话状态的跨轮维护。Anthropic 把这件事叫做 **context compact**。它不是可选优化，是长时运行 Agent 能不能活下去的关键器官。

### 5. 权限与沙箱

是让它直接在宿主机上跑，还是在一个 Docker 沙箱里跑；是让它自动执行所有 shell 命令，还是对高危命令弹出确认窗口。

### 6. 错误恢复

对 Agent 来说，失败是日常天气：模型会超 token，会触发 prompt too long，会遇到工具拒绝。一个成熟的 harness 必须把这些失败路径当作主路径来设计。

### 7. 多代理编排与验证

一个任务如果需要拆解、研究、实现、验证四个阶段，系统该怎么组织？验证环节需要独立角色，因为一个实现者天然倾向于相信自己的改动"差不多行了"。

### 8. 本地规则与 Hook

这包括 CLAUDE.md、AGENTS.md 这类项目级配置文件，也包括 pre-commit hook、session-start hook 这类生命周期钩子。

这八个器官不是独立的模块，它们共同构成一个循环系统。

---

## 六、三层划分：通用、项目、任务

社区里有一个相当实用的划分，把 harness 按照与具体项目的关联程度分成三层。

### 通用 Harness 层

和具体项目弱相关，属于 Agent runtime 或 framework 的通用能力。包括终端交互、tool loop、权限系统、记忆、线程持久化、context compaction、hook 机制。

各大 Agent CLI——Claude Code、Codex、OpenCode——已经在这一层做了大量工作。

### 项目 Harness 层

和具体项目强相关，但不是业务功能本身。包括 AGENTS.md、仓库的知识布局、架构边界定义、lint 规则、质量标准。

OpenAI 那篇 Codex 工程报告里最有参考价值的经验，大部分集中在这一层。比如他们提到的"AGENTS.md 保持在 100 行以内，作为目录而不是百科全书"。

### 任务 Harness 层

和当前这次具体工作强相关。例如 planner-generator-evaluator 三 Agent 架构，跨 session 的文档交接机制，针对特定任务的 QA prompt。

这一层的 harness 往往是任务临时搭起来的，完成之后可能就拆了。

---

## 七、为什么是 2026 年初

**偶然的部分**：那几个标志性事件的时间集中度—— Mitchell 的博客、OpenAI 的工程报告、Anthropic 的研究文章、Claude Code 源码泄露，全都发生在两个月之内。

**必然的部分**：AI coding 领域的主要矛盾变了。

| 时间 | 主要矛盾 |
|:---|:---|
| 2023-2024 | 模型能不能用 |
| 2024下-2025 | 怎么把合适的信息喂给模型 |
| 2025底-2026 | 系统能不能让聪明的模型稳定地做事 |

> 主要矛盾从"模型够不够聪明"变成了"系统能不能让聪明的模型稳定地做事"。harness engineering 之所以在这个时间点爆发，是因为社区终于有了一个共识：这件事的瓶颈已经不在模型一侧了。

---

## 八、最后

经过前面几节的铺垫，现在可以回答得比较克制。

**Harness Engineering** 是一个在 2026 年初由 Mitchell Hashimoto、OpenAI、Anthropic 和社区共同催生的工程术语。它关心的是模型外围的那一整套运行结构：system prompt 与指令分层、query loop、工具系统、上下文治理、权限与沙箱、错误恢复、多代理编排、本地规则与 hook。

它和 prompt engineering、context engineering 的关系更像是**三层楼板**，而不是三代拳王。

如果一定要用���个���式概括：

```
Agent = Model + Harness
```

这篇文章只讲清楚了"是什么"，没有回答一个更重要的问题：**为什么必须要它？**

这个问题的答案藏在那些没有 harness 保护就会发生的具体崩溃里。

- Agent 不守规矩
- 修 A 坏 B
- 上下文腐朽
- 跨会话失忆
- 任务膨胀

每一种崩溃都对应 harness 的一个具体器官。把这些崩溃场景讲清楚，就能理解为什么前面列出的那些看起来琐碎的组件，其实一个都不能少。

---

## 参考资料

- Mitchell Hashimoto, 《My AI Adoption Journey》, 2026年2月5日
- OpenAI (Ryan Lopopolo), 《Harness Engineering: Leveraging Codex in an Agent-First World》, 2026年2月11日
- Anthropic, 《Effective harnesses for long-running agents》, 2026年3月24日
- swyx, agent engineering / IMPACT framework, 2025年3月
- Viv Trivedy, 《The Anatomy of an Agent Harness》
- Claude Code 源码泄露事件（Chaofan Shou, 2026年3月31日）