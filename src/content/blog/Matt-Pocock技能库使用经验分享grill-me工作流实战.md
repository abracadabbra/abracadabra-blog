---
title: 'Matt Pocock 技能库使用经验分享：grill-me 工作流实战'
description: '深入解析 Matt Pocock 技能库的核心用法，从 grill-with-docs 到 tdd 的完整工作流，以及作者魔改后的个性化实践'
pubDate: '2026-06-07'
tags: ['AI', 'Matt Pocock', '技能库', 'grill-me', 'tdd', '工作流']
categories: ['AI开发']
---

> 本文整理自 [Linux.do 社区 NukaColaM 的帖子](https://linux.do/t/topic/2296905)，作者分享了使用 mattpocock/skills 的实战经验。

---

## 前言

在 AI 编程工具层出不穷的今天，如何高效地与 LLM 协作完成真实项目开发？Matt Pocock 的技能库（mattpocock/skills）提供了一套经过实践检验的工作流体系。本文将深入解析这套技能库的核心用法，以及作者在实战中的魔改与优化。

---

## 核心思想：达成共识

在整个工作流中，**「达成共识」** 是最核心的关键词。如果你的需求描述太宽泛，LLM 会不断地追问直到完全理解你的意图。这个过程虽然看起来繁琐，但实际上避免了后期大量的返工。

作者原话：

> grill-with-docs 强就强在"达成共识"四个字，如果你的需求太宽泛，是真的会问到你神志不清的。

---

## 最好用的技能

### 1. grill-me / grill-with-docs

**王者技能**，也是整个工作流的起点。

grill-with-docs 会记录文档，这是一个很好的设计。但需要注意：**文档在完成任务之后应该删掉**，因为可能会与后续的需求产生冲突。

作者在 WinTProxy 重构时就遇到了这个问题：

> 切换 ndisapi 就遭殃了，它工作在二层，然后 NAT 会经过几个网卡，然后就重复捕获了好几次。

### 2. to-issues

这个技能的精髓在于**垂直切片**思想：

> 任务划分是从端到端的划分，而不是层间划分。一个任务需要处理完一个需求从后端到前端的全部实现。

有效反馈对于 AGENTS 来说是提效的重点。

### 3. diagnose

很好用的 debug 技能，规范了一整套流程。作者自己魔改后去掉了后面的修改和测试步骤，只报告原因，把修改和测试交给 to-issues 和 tdd。

### 4. tdd

**另一个王者技能**。测试驱动开发在 AGENTS 时代是最合适的开发方式。

> 什么叫做写完就结项？准确来说不是这个技能是王者，是测试驱动这种思想是王者。

---

## 我不用的技能

### triage

搭配 Issue tracker 使用。如果你是本地文档，大概率不需要这个技能：

> 你不可能先写个文档描述 issue，然后再丢目录里面去排序吧！

### zoom-out

一句话总结：直接写提示词都行，没有复杂工作流程。本质上只是方便一点。魔改的话可以让它做数据流图之类的，更清晰。

### Issue tracker

作者认为本地文档更方便。但 Matt Pocock 原设计是维护 GitHub 项目的，所以这个设计并没有问题。

---

## 标准工作流

### 推荐流程

```
grill-with-docs → prototype（可选）→ to-prd → to-issues → tdd
```

或者用于架构改进：

```
improve-codebase-architecture → prototype（可选）→ to-prd → to-issues → tdd
```

debug 场景：

```
diagnose → tdd
```

### prototype 处理

prototype 一般要新开一个 session 完成，前后用 handoff 交接。handoff 的意义是形成 prototype 过程中可取之处和不可取之处的总结性文档。

### to-prd 与 to-issues 的关系

这两个技能很多时候形影不离，但拆分是有道理的。有时候不会去写 prd，一个明确的需求就直接拆分任务了。

---

## 为什么选择这套工作流

superpowers、trellis 等工作流都用过，它们的共同特点是**流程控制比较强，或者说比较重**。对于追求效率的开发者来说，省 token 就是省钱。

作者的观点是：

> 人总是懒惰的，等后续 LLM 的继续发展，这部分工作肯定是越来越少，希望这个过渡期短一点吧。

---

## 实战示例：TinyTrans 托盘菜单优化

### 需求提出

作者做了一个翻译小工具 TinyTrans，想优化右键菜单。他直接用「经典甲方语录」发起需求：

> "I want to optimize the right-click menu; it looks very ugly right now. Clarify it."

TinyTrans 原始托盘菜单长这样：

![TinyTrans 原始托盘菜单](https://cdn3.ldstatic.com/original/4X/a/2/e/a2e63a7055e8a77cc57fd3542844cf61252b2e19.png)

### grill-with-docs 阶段

grill-with-docs 开始「拷问」需求细节，全程追问直到完全理解。这个过程虽然长，但确保了双方对需求的一致理解。

![grill-with-docs 开始追问](https://cdn3.ldstatic.com/original/4X/2/b/b/2bbecda0a57814fa90f6bbd14bf0b2989bdd2327.png)

LLM 问了几个问题来明确需求：

![继续追问细节](https://cdn3.ldstatic.com/original/4X/7/9/2/7921391f3bcf342c760afe3d53b53e7c47b36c8a.png)

![询问具体需求](https://cdn3.ldstatic.com/original/4X/6/e/2/6e2785bebc7973782a526f895ae27b73ee402cc6.png)

### to-prd 阶段

一旦「拷问」完成，意味着你和 LLM 就「达成共识」了。接下来形成 PRD 文档，全程跟着 LLM 的提示走。

![to-prd 形成文档](https://cdn3.ldstatic.com/original/4X/0/c/1/0c1b5d1f9d3e7d9612917f488290e3d1fa7ff958.png)

![继续完善 PRD](https://cdn3.ldstatic.com/original/4X/0/5/1/0512f46ccb4315c321290d5a0f891fe12e81b285.png)

关于是否要自己审查 PRD：

> 我建议你审！但是我不审，因为我懒。本质上这个 spec 或 prd 就是你跟 LLM 对话的总结，我反正是达不到 LLM 的总结能力的。

### to-issues 阶段

拆分任务时唯一需要注意的就是检查是否做了**垂直切片**。当然，这个事情还是具体问题具体分析，横向切片也不是完全不能接受。

![to-issues 拆分任务](https://cdn3.ldstatic.com/original/4X/7/2/0/7201c4d3726c83d817ec73720cd2d116be7380d3.png)

### tdd 阶段

让 LLM 这个「牛马」自己跑就是了。它会用 RED/GREEN 方式自己鞭策自己，过不了就自己打回重做。

![tdd 红色阶段](https://cdn3.ldstatic.com/original/4X/d/c/c/dcc59384075a557bac630100bd80e55997563e6e.png)

![tdd 绿色阶段](https://cdn3.ldstatic.com/original/4X/a/f/5/af5594a0a3d0362c575ee0b9553cd391e0886651.png)

最终的成品：

![TinyTrans 优化后托盘菜单](https://cdn3.ldstatic.com/original/4X/4/2/0/420179ebee8bcf9afb32da88751d13bce68999e19.png)

作者评价：D 老师的品味一言难尽。🤪

---

## 社区观点补充

### 关于模型选择

> 本质上，一旦你模型足够强，你其实都可以完全抛弃这些工作流的。但现阶段，至少在我自己实践中，复杂项目其实 GPT-5.5 也一样是没办法完全脱离约束自由发挥的。

### 关于文档管理

> 完成了之后应该删掉。一旦你新的功能或者改动与旧需求冲突，是会产生污染的。

如果真的要做版本管理，写日期就好。

### 关于工作流选择

> 简单需求直接让它改就好了。superpowers 就是啥都要你走流程所以才费 token。

---

## 写在最后

现在的工作流构建什么的始终都是**过渡产品**。随着 LLM 能力的持续提升，这部分工作肯定会越来越少。等 LLM 能准确判断需求大小时，我们可能只需要说「做一个像样的翻译工具」，剩下的它都能搞定。

希望这个过渡期短一点吧。

---

## 参考链接

- [Matt Pocock Skills 仓库](https://github.com/mattpocock/skills)
- [原文讨论帖](https://linux.do/t/topic/2296905)
