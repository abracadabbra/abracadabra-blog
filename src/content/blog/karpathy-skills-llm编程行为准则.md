---
title: 'Karpathy Skills：让 AI 编程 Agent 少犯错的 4 条原则'
description: '深度解析 Andrej Karpathy 提出的 LLM 编程行为准则，Think Before Coding / Simplicity First / Surgical Changes / Goal-Driven Execution，以及在 Claude Code、Cursor 中的使用方法。'
pubDate: '2026-05-28 10:00:00'
tags: ["Claude-Code", "AI工具", "编程规范", "提示词工程"]
categories: ["AI工具"]
---

## 前言

Andrej Karpathy（OpenAI、 Tesla Autopilot 创始人）2025 年发了一条推特，炮轰 LLM 写代码时的几类典型错误：

> "模型替你做了错误假设，还闷头跑。它们不管理自己的困惑，不寻求澄清，不暴露不一致性，不展示权衡，不在应该反驳的时候反驳。"

> "它们超级喜欢把代码和 API 搞复杂，堆砌抽象层，删掉它们不理解的代码和注释..."

这条推特引发了轩然大波，后来被整理成了一套可安装的 skills，供 Claude Code、Cursor 等工具使用——这就是 **Karpathy Skills**。

---

## 4 条核心原则

| 原则 | 解决什么问题 |
|------|------------|
| **Think Before Coding** | 错误假设、隐藏困惑、不展示权衡 |
| **Simplicity First** | 过度设计、堆砌抽象 |
| **Surgical Changes** | 乱改无关代码、删除不理解的内容 |
| **Goal-Driven Execution** | 测试先行、明确的成功标准 |

### 1. Think Before Coding

**不要假设。不要隐藏困惑。主动展示权衡。**

LLM 经常默默地选择一个解释就开干，这个原则强制显式推理：

- **显式陈述假设** — 不确定就问，不要猜
- **呈现多种解释** — 存在歧义时不要默默选一个
- **该反驳就反驳** — 如果存在更简单的方案，要说出来
- **困惑时停下来** — 说出不清楚的地方，请求澄清

### 2. Simplicity First

**最小代码解决问题。没有投机性代码。**

对抗过度设计的倾向：

- 不添加需求之外的功能
- 不为单次使用的代码创建抽象
- 不添加没人要求的"灵活性"或"可配置性"
- 不为不可能的场景添加错误处理
- 如果 200 行能写成 50 行，重写

自测标准：**"一个高级工程师会说这太复杂了吗？"** 如果是，简化。

### 3. Surgical Changes

**只碰必须碰的。只清理自己造成的烂摊子。**

修改现有代码时：

- 不要"优化"相邻代码、注释或格式
- 不要重构没坏的东西
- 匹配现有风格，哪怕你自己会写得不一样
- 发现无关的死代码——说出来，不要删

你的修改产生了孤立代码时：

- 移除你的修改导致的未使用 import/变量/函数
- 不要删除本来就存在的死代码，除非被要求

自测标准：**每一行修改都能追溯到用户的原始请求。**

### 4. Goal-Driven Execution

**定义成功标准。循环直到验证通过。**

把指令性任务转化为可验证的目标：

| 不要这样 | 改成这样 |
|---------|---------|
| "添加验证" | "先写无效输入的测试，再让它们通过" |
| "修复 bug" | "写一个复现 bug 的测试，再让它通过" |
| "重构 X" | "确保重构前后测试都通过" |

多步骤任务中，先陈述简要计划：

```
1. [步骤] → 验证：[检查点]
2. [步骤] → 验证：[检查点]
3. [步骤] → 验证：[检查点]
```

强成功标准让 LLM 可以自主循环。弱标准（"让它工作"）需要不断澄清。

---

## 原理：CLAUDE.md 是怎么生效的？

Claude Code 启动时会**自动扫描当前目录及父目录**，找到最近的 `CLAUDE.md`（或 `AGENTS.md`）读进 system prompt。本质上是**静态文件注入**——文件存在就生效，没有激活/停用开关。

```
/project/
  CLAUDE.md        ← 读
  src/
    CLAUDE.md      ← 不读（不扫描子目录）
/parent/
  CLAUDE.md        ← 不读
```

---

## 使用方式对比

### 方式一：Claude Code 插件（全局生效）

适合：想让所有项目都遵守这 4 条原则。

```
/plugin marketplace add forrestchang/andrej-karpathy-skills
/plugin install andrej-karpathy-skills@karpathy-skills
```

本质上是把 CLAUDE.md 内容安装到 Claude Code 全局插件目录，**所有项目启动时都会加载**，与项目本地的 CLAUDE.md 是并存关系，不是替换。

### 方式二：手动追加到项目 CLAUDE.md（单项目生效）

适合：只在特定项目里使用，或想通过 Git 管理。

```bash
# 新项目：直接下载
curl -o CLAUDE.md https://raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md

# 已有项目：追加到末尾
echo "" >> CLAUDE.md
curl https://raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md >> CLAUDE.md
```

### 方式三：Cursor 规则

仓库带了 `.cursor/rules/karpathy-guidelines.mdc`，Cursor 打开项目自动加载。

```bash
# 在项目里克隆
git clone https://github.com/forrestchang/andrej-karpathy-skills.git
# 然后用 Cursor 打开项目目录即可
```

---

## 和 Trellis 冲突吗？

**不冲突。**

Trellis 是团队协作层的东西——管理多 CLAUDE.md 配置、分发 prompt 模板。Karpathy Skills 无论以哪种方式引入，对 Claude Code 来说都是 **system prompt 里多了一段文字**。

唯一要注意：Trellis 模板里如果已经装了 Karpathy-skills，再装插件会重复——4 条原则出现两次，只是冗余，不影响功能。

**建议**：既然用 Trellis，直接把 Karpathy 的 4 条原则加到 Trellis 的模板配置里，通过 Trellis 分发，**不需要装插件**。

---

## 效果检验

这 4 条原则起效了的话，会看到：

- ✅ diff 里没有不必要的修改
- ✅ 代码第一次写出来就不复杂
- ✅ 提问在实现之前，而不是出错之后
- ✅ PR 干净，没有顺便"优化"别的地方
- ✅ 困惑时会停下来问，而不是闷头猜

---

## 适用场景

| 场景 | 推荐程度 |
|------|---------|
| 个人项目用 Claude Code 编程 | ⭐⭐⭐⭐⭐ 强烈推荐 |
| 团队统一 AI 编程行为规范 | ⭐⭐⭐⭐⭐ 通过 Trellis 分发 |
| 临时一次性小任务 | ⭐⭐ 不值得折腾，直接干 |
| Cursor 用户 | ⭐⭐⭐⭐ 同样适用 |

---

## 总结

Karpathy Skills 解决的不是"模型能力不足"的问题，而是**模型行为模式**的问题——它太容易闷头跑、太喜欢过度设计、太爱顺手改别的地方。

4 条原则本质上是在 LLM 运行时之前加了一道"检查清单"，让它在动手之前先想清楚：

1. 我的假设是什么？我有没有困惑？
2. 最小方案是什么？有没有过度设计？
3. 我只改必须改的吗？
4. 成功的标准是什么？怎么验证？

不是什么高深技术，就是**好的编程习惯的显式表达**，但对于容易跑偏的 LLM 来说，效果拔群。
