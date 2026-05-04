---
title: '从Vibe Coding到Harness Engineering，再到Trellis落地'
description: 'AI编程工作范式的三层进阶：Vibe Coding负责推进开发流程，Harness Engineering负责让流程稳定可控，Trellis负责把这套能力工程化落地。'
pubDate: '2025-05-04'
tags: ['AI', 'Vibe Coding', 'Harness Engineering', 'Trellis', '工作流']
categories: ['AI开发']
---

> 今天想和大家聊三个关键词：Vibe Coding、Harness Engineering、Trellis。它们看起来像是三个不同层次的话题，但其实可以串成一条很清晰的主线。

---

## 一、Vibe Coding 工作范式：从需求到验证的闭环

Vibe Coding 的核心不是"写得快"，而是"尽快进入一个可以讨论、可以验证、可以迭代的状态"。

### 1. 原型图设计

原型图的意义：
- 让需求方更直观地理解流程
- 让研发提前看到交互路径
- 让所有人对同一件事形成共同认知

> 很多时候，原型图比长篇文字更能暴露问题。因为一旦流程画出来，哪里不合理、哪里太复杂、哪里漏了状态，马上就会显现出来。

### 2. 需求澄清

**第一步一定不是写代码，而是澄清需求。**

> 很多问题不是开发慢，而是一开始就没有把目标讲清楚。如果目标不清楚，后面 PRD、原型、前端、后端、测试都会跟着偏。

### 3. PRD 输出

需求澄清之后，就要把口头描述沉淀成 PRD 或 Spec。

> 这一层的重点不是"写得漂亮"，而是"写得清楚"。回答一个关键问题："这个东西应该长什么样，做到什么程度，才算完成？"

### 4. 前端实现

> 这一阶段的重点不只是"页面画出来"，而是要确保：
> - 页面结构和 PRD 一致
> - 交互逻辑和原型一致
> - 状态流转没有遗漏
> - 关键异常场景能够处理

### 5. 后端实现

> 后端这层最重要的不是"有接口"，而是：
> - 接口定义是否清晰
> - 数据结构是否合理
> - 业务规则是否完整
> - 权限、异常、边界是否处理到位

### 6. 测试与评测

> 最后一步不是结束，而是验证。核心目的不是"证明我写了"，而是"证明它真的对了"。

---

## 二、Harness Engineering：让 AI 写代码从"会写"走向"能用"

如果说 Vibe Coding 解决的是"怎么把开发流程推进起来"，那 Harness Engineering 解决的就是"怎么让 AI 在真实项目里持续稳定地做对事"。

### 1. AI 写代码的三种境界

2024-2026 年，AI 工程领域依次出现了三个核心概念。它们不是替代关系，而是层层嵌套：

```
┌─────────────────────────────────────────────────────────────────┐
│                     Harness Engineering                         │
│                                                                  │
│     Agent / 工作流的整体系统设计                                 │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                      Context Engineering                        │
│                                                                  │
│          所有输入给 LLM 的上下文的设计与管理                    │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                     Prompt Engineering                          │
│                                                                  │
│               人类给 LLM 的指令文本优化                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| 维度 | Prompt Engineering | Context Engineering | Harness Engineering |
|:---|:---|:---|:---|
| 核心问题 | "怎么写指令" | "给模型看什么" | "防止什么、控制什么、修复什么" |
| 失败模式 | 回答跑偏 | 单次输出不准确 | 质量随时间退化 |
| 典型实现 | 提示词优化 | RAG、Memory、工具定义 | Linter、CI 集成、任务拆分 |

### 2. Harness 的本质

Harness 这个词本来是"马具"的意思。

> 马本身很有力量，但你不能只靠它乱跑；你需要缰绳、马鞍、嚼子，把力量导向正确方向。

放到 AI 里也一样：
- **Model** 是能力本体
- **Harness** 是让能力可控、可验证、可持续的系统

所以可以非常直接地理解成：
```
Agent = Model + Harness
```

模型负责"想"，Harness 负责"管"。

如果没有 Harness，模型可能很强，但它：
- 不一定遵守规范
- 不一定记得进度
- 不一定知道自己哪里错了
- 不一定能在长任务里一直保持一致性

### 3. 几种 Harness 开源方案

当前个人开发者可选的 harness 工具和方案大致可分为三种：

| 方案 | 描述 | 适合人群 |
|:---|:---|:---|
| 纯配置方案 | 直接编写 CLAUDE.md / AGENTS.md，配合 superpowers 等 skill 集合 | 入门/轻量需求 |
| 框架增强方案 | Trellis 和 ccg-workflow 等框架在配置层之上增加了 spec 系统和 hook 自动注入能力 | 持续迭代项目 |
| 多角色编排方案 | oh-my-claudecode、oh-my-opencode 等套件定义了多种 agent 角色 | 复杂工作流 |

> 值得注意的是，Harness 说到底是在模型智能的基础上构建系统，随着模型基础能力的提升，Harness 的部分组件应该能够简化甚至移除。

**Anthropic 的研究对此的原话是**：

> "every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions are worth stress testing, both because they may be incorrect, and because they can quickly go stale as models improve."

---

## 三、Trellis 框架：把 Harness Engineering 工程化落地

> Trellis 是把 Harness Engineering 真正工程化、自动化、可持续化的一套框架。它不是一个单纯的提示词工具，也不是一个简单的技能集合，而是一整套把规范、任务、上下文、验证、沉淀串起来的系统。

### 1. Trellis 的核心价值

Trellis 的价值不在于"又多了一套工具"，而在于它把以下几件事真正连起来了：
- 规范
- 任务
- 上下文
- 验证
- 历史沉淀

> 所以它不是让 AI 变得更神奇，而是让 AI 的工作方式更像一个真正可运行的工程系统。

### 2. 为什么选择 Claude Code + Trellis

Trellis 之所以能真正发挥作用，很大程度上是因为它和 Claude Code 的 hook 机制天然契合。

#### Claude Code 工具调用生命周期

工具调用生命周期是钩子系统中使用频率最高、功能最强大的一组事件。它们构成了一个"三明治"结构：

```
PreToolUse （工具执行前）
PostToolUse （工具执行后���
PostToolUseFailure （工具执行失败后）
```

#### Trellis Hook 执行顺序

**session-start：会话启动时注入上下文**

| 来源 | 用途 |
|:---|:---|
| .trellis/.developer | 开发者身份 |
| .trellis/workflow.md | 工作流指南 |
| workspace/{name}/index.md | 会话历史 |
| git log | Git提交记录 |
| .trellis/tasks/ | 活跃任务列表 |

**inject-subagent-context：Agent调用时注入精确Spec**

| Agent | 注入内容 | 触发条件 |
|:---|:---|:---|
| implement | implement.jsonl + prd.md + info.md | 实现功能时 |
| check | check.jsonl + prd.md | 代码检查时 |
| check [finish] | finish-work.md + prd.md | 完成检查时 |
| debug | debug.jsonl + codex-review-output.txt | 调试bug时 |
| research | 项目结构概览 + research.jsonl | 信息搜索时 |

**ralph-loop：Check完成后自动质量验证**

### 3. 日常使用三步走

```bash
/trellis:start
-> 输入需求...
-> vibe coding...
/finish-work
#检查完工清单
/record-session
#会话持久化存储
```

### 4. Trellis 的 spec 产物结构

```
.trellis/spec/
├── frontend/
│   ├── index.md
│   ├── component-guidelines.md
│   ├── hook-guidelines.md
│   ├── state-management.md
│   ├── type-safety.md
│   ├── quality-guidelines.md
│   └── directory-structure.md
├── backend/
│   ├── index.md
│   ├── database-guidelines.md
│   ├── error-handling.md
│   ├── logging-guidelines.md
│   ├── quality-guidelines.md
│   └── directory-structure.md
└── guides/
    ├── index.md
    ├── cross-layer-thinking-guide.md
    └── code-reuse-thinking-guide.md
```

---

## 四、把三者串起来：从"会用 AI"到"驾驭 AI"

现在我们把三部分连起来看，主线就会非常清楚：

1. **Vibe Coding** 负责起步
   → 先把需求推进到可讨论、可验证、可迭代的状态

2. **Harness Engineering** 负责稳定
   → 让 AI 在真实项目里持续正确地工作，而不是今天对、明天错

3. **Trellis** 负责落地
   → 把方法变成系统，把经验变成规范，把规则变成自动化机制

### 一句话总结

> 真正优秀的 AI Coding，不是让模型替你写完代码，而是让你有能力设计一套系统，使模型持续稳定地把事情做对。

---

## 五、其他 Trellis 命令

| 功能 | Claude Code / OpenCode |
|:---|:---|
| 启动会话 | /start |
| 后端准备 | /before-backend-dev |
| 前端准备 | /before-frontend-dev |
| 检查后端 | /check-backend |
| 检查前端 | /check-frontend |
| 跨层检查 | /check-cross-layer |
| 完工清单 | /finish-work |
| 并行编排 | /parallel |
| 记录会话 | /record-session |
| 头脑风暴 | /brainstorm |
| Bug分析 | /break-loop |
| 新成员入门 | /onboard |
| 创建命令 | /create-command |
| 集成Skill | /integrate-skill |

---

## 六、结语

AI 编码的重点，已经不只是"会不会写"，而是"能不能被驾驭"。

- 如果只会 Vibe Coding，可能会快，但不一定稳
- 如果只有 Harness 的概念，没有落地系统，可能知道该怎么做，但不一定能持续用
- 如果有 Trellis 这样的框架，才能把方法变成可持续的工程能力