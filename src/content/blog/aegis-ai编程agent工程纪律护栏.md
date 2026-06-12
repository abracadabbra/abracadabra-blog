---
title: 'Aegis：为 AI 编程 Agent 装上工程纪律护栏'
description: '深度解析 Aegis Method Pack：如何用 evidence-driven 工作流让 AI 编程 Agent 变得可验证、可预测、不熵增。'
pubDate: '2026-06-12'
tags: ["AI", "AI Agent", "编程工具", "方法论", "Claude Code", "Superpowers"]
categories: ["AI开发"]
---

> 一个 AI 编程 Agent 最常见的问题是什么？它说「完成了」但没跑验证。它说「修好了」但没读代码。它加了新逻辑但没删旧路径。Aegis 就是在解决这个问题。

---

## 项目概览

**Aegis** 是面向 AI 编程 Agent 的工作流纪律包，派生自 [Superpowers](https://github.com/obra/superpowers)，由 [@GanyuanRan](https://github.com/GanyuanRan) 开发和维护。

```yaml
⭐ 498 | Shell | MIT License
创建: 2026-04-30
更新: 2026-06-12
主题: agent-skills, ai-agents, architecture-driven-development, baseline-first, evidence-driven
```

它当前的产品形态是 **`Aegis Method Pack (runtime-ready)`**，意思是它是一套方法包，本身不是运行时引擎，但产出的 artifacts 可以供未来 runtime core 使用。

🔗 GitHub: https://github.com/GanyuanRan/Aegis

---

## 为什么需要 Aegis

AI 编程 Agent 的速度已经很快了。但快带来的问题是：**它报完成的时候，往往并没有真正完成**。

常见场景：

```
AI：「修好了！✅」
人：「跑一下测试」
AI：「……测试还是 fail」
```

```
AI：「完成了，代码已经提交」
人：「有没有跑 lint？」
AI：「没有，我这就跑」
```

```
AI：「加了缓存逻辑，性能应该会提升」
人：「改了多少行？」
AI：「加了 200 行左右」
人：「旧缓存逻辑删了吗？」
AI：「呃……没有」
```

这些问题本质上是**工程纪律的缺失**。Aegis 做的事情，就是在 AI 编程 Agent 周围加一套护栏，让它：

- 动手前先读项目事实（Baseline first）
- 报完成前先跑验证命令（Evidence before claims）
- 修 bug 时同步说明旧逻辑的命运（Repair + Retirement track）
- 代码改动后报告复杂度变化（Complexity Delta）

---

## 核心哲学

### 1. Baseline First — 先读事实再动手

```text
重大改动前 → 先读当前项目 authority
                ↓
         读什么？
         ├── AGENTS.md / docs/current/
         ├── README / ADR / 架构文档
         └── 相关模块的 owner 和边界
```

不凭记忆操作，不凭会话印象推断。先把项目的当前状态搞清楚再说。

### 2. Evidence Before Claims — 没有证据不报完成

常规 AI 行为：
```
写完代码 → 「完成了！✅」
```

Aegis 行为：
```
写完代码 → 跑验证命令 → 报告 evidence → 才说完成
```

每次 completion claim 必须包含：

```yaml
Evidence action:      # 跑了什么命令
Result / exit status: # 输出是什么
Covered scope:        # 覆盖了什么
Uncovered scope:      # 什么没测
Residual risk:        # 还有什么风险
Confidence grade: A | B | C  # 信心等级
```

### 3. Repair + Retirement Track — 修 bug 不能只管新不管旧

修一个问题时，要同时回答：

```
Repair Track:
  - 对象: 哪个文件/模块
  - 动作: 怎么修的
  - 影响: 修完之后呢
  - 验证: 怎么证明修好了

Retirement Track:
  - 对象: 旧的什么逻辑/路径/owner
  - 动作: 保留 / 废弃 / 迁移
  - 边界: 保留的范围是什么
  - 触发: 什么情况下彻底删除
```

### 4. Workflow Quality — 简单任务不展开

Aegis 按任务复杂度自动路由：

| 复杂度 | 路径 | 描述 |
|--------|------|------|
| **低** | Fast Path | 快速确认 → 轻量验证 → 输出结果 |
| **中** | Standard Path | Baseline → Spec → 计划 → 原子任务 → 验证 → 质量收口 |
| **高** | High-Complexity Path | Design Spec → 用户确认 → 计划 → 实施 → 完整报告 |

---

## 22 个 Skill 体系

Aegis 的核心是 22 个可组合的 Skill，覆盖 AI 编程的完整生命周期：

### 入口层

| Skill | 作用 |
|--------|------|
| `using-aegis` | 轻量路由入口，判断加载哪个 skill |
| `goal-framing` | 任务目标定义，生成 TaskIntentDraft |

### 规划层

| Skill | 作用 |
|--------|------|
| `brainstorming` | 产品/架构/行为不清时，先设计再实施 |
| `writing-plans` | 把 spec 拆成可执行的多步骤计划 |
| `first-principles-review` | 重大决策前的第一性原理审查 |

### 实施层

| Skill | 作用 |
|--------|------|
| `subagent-driven-development` | 并行分发多个独立任务给子 Agent |
| `dispatching-parallel-agents` | 多 Agent 并行调查独立问题 |
| `executing-plans` | 执行计划 + checkpoint + drift 检查 |
| `finishing-a-development-branch` | 分支收尾，merge 前质量门控 |

### 调试层

| Skill | 作用 |
|--------|------|
| `systematic-debugging` | 系统化调试，7 层诊断链路 |

### 验证层

| Skill | 作用 |
|--------|------|
| `verification-before-completion` | 完成前必须跑验证命令 |
| `receiving-code-review` | 处理代码审查意见 |

### 质量层

| Skill | 作用 |
|--------|------|
| `requesting-code-review` | 请求代码审查 |
| `establishing-project-context` | 建立项目上下文 |
| `recording-architecture-decisions` | ADR 归档 |

### 复杂任务专项

| Skill | 作用 |
|--------|------|
| `long-task-continuation` | 长任务防丢失，上下文 checkpoint |
| `anti-entropy-governance` | 防止系统熵增，删除旧逻辑 |

### 工具类

| Skill | 作用 |
|--------|------|
| `communicating-concisely` | 精简沟通 |
| `using-git-worktrees` | Git worktree 并行开发 |
| `writing-skills` | 编写新的 Aegis skill |
| `update-aegis` | 更新 Aegis 本身 |

---

## 场景一：修一个 Bug

当用户说「某个功能坏了」，Aegis 的处理流程：

```
systematic-debugging 加载
         ↓
Phase 1: 根因调查
  L1 症状 → L2 逻辑 → L3 系统 → L4 架构
  → L5 跨系统契约 → L6 平台约束 → L7 规格缺口
         ↓
Phase 2: 定位 owner（对比正常代码）
         ↓
Phase 3: 证明假设（一个假设 → 最小测试 → 迭代）
         ↓
Phase 4: 修复（失败测试 → 最小代码 → 验证）
         ↓
Repair Track + Retirement Track 同时报告
```

关键点：**3 次以上修复失败，停止代码修复，回头审查架构**。这是 Aegis 很重要的一个护栏——不让 Agent 在错误的方向上死磕。

---

## 场景二：实现一个新功能

```
brainstorming → 需求不清时先设计
       ↓
writing-plans → 拆成原子任务
       ↓
subagent-driven-development → 并行实施
       ↓
verification-before-completion → 每个任务出口验证
       ↓
Complexity Delta → 报告改动对系统复杂度的影响
```

---

## 场景三：声明任务完成

这是 Aegis 最反 AI 天性的设计。当 AI 想说「完成了」时：

```yaml
# ❌ AI 常规行为
「修好了，测试通过了」

# ✅ Aegis 要求
Verification:
  - Evidence action:     pytest tests/auth/test_refresh.py -v
  - Result / exit:       12 passed, 0 failed, exit 0
  - Covered scope:        auth refresh 主路径 + 并发
  - Uncovered scope:      跨时区边界情况
  - Residual risk:        多域名场景尚未验证
  - Confidence grade:     B（直接验证，有界风险）

Complexity Delta:
  - Files changed:        3（auth/refresh.py, auth/token.py, conftest.py）
  - Lines delta:          +47 / -12
  - Retired paths:        旧 token 轮询逻辑（无调用方，删除）
  - Net entropy:         stable
```

**Confidence Grade 说明**：
- **A** — 直接验证 + 回归测试，无未知项
- **B** — 直接验证，有界风险
- **C** — 部分验证，未完全关闭

---

## 安装与使用

### 方式一：复制安装咒语给 Agent

把这段话发给 AI 编程 Agent：

```
请阅读 https://github.com/GanyuanRan/Aegis，识别我当前使用的 AI 编程宿主，
并按对应宿主说明全局安装 Aegis。如果需要重启或重新加载宿主，请明确告诉我；
然后从已安装的 Aegis method-pack 根目录运行完整安装验证。
```

### 方式二：手动安装（以 Claude Code 为例）

```bash
# 克隆到本地
git clone https://github.com/GanyuanRan/Aegis.git ~/.aegis-method-pack

# 配置 Claude Code 使用全局 skills
# 编辑 ~/.claude/settings.json 或项目 CLAUDE.md
export AEGIS_METHOD_PACK_ROOT=~/.aegis-method-pack
```

### 激活使用

```bash
# 轻量路由（自动判断加载哪个 skill）
aegis:systematic-debugging

# 或者用自然语言
Aegis goal: 修复登录后偶发跳回登录页，不重写 auth 系统
```

---

## 多宿主支持

Aegis 支持多种 AI 编程宿主，并针对每种做了适配：

| 宿主 | 状态 | 说明 |
|------|------|------|
| **Codex** | ✅ 有 fresh evidence | 完整测试通过 |
| **OpenCode** | ✅ 有 fresh evidence | base suite + 集成通过 |
| **Claude Code** | 📋 有文档 | 需测试验证 |
| **GitHub Copilot** | 📋 有文档 | 需测试验证 |
| **DeepSeek-TUI** | 📋 有文档 | 需测试验证 |
| **Trae** | 📋 有文档 | 需测试验证 |
| **Hermes Agent** | 📋 有文档 | 有独立适配文档 |

---

## Aegis vs Superpowers — 一脉相承的进化

Aegis 派生自 [obra/superpowers](https://github.com/obra/superpowers)（今日 Trending ⭐224,507），两者关系：

| 维度 | Superpowers | Aegis |
|------|-------------|-------|
| 核心概念 | Composable skills（技能可组合） | Superpowers + 工程纪律层 |
| 关注点 | Agent 能做什么 | Agent 做完之后可验证吗 |
| 熵增控制 | 无 | Complexity Delta + Anti-Entropy |
| Evidence 驱动 | 无 | verification-before-completion |
| 复杂度路由 | 无 | Fast / Standard / High 三层 |
| 设计理念 | 让 Agent 更强 | 让 Agent 更可靠 |

简单说：**Superpowers 让 Agent 知道怎么做事，Aegis 让 Agent 做完了知道怎么证明**。

---

## 适用场景评分

| 场景 | 评分 | 说明 |
|------|------|------|
| 单次快速任务（修改配置等） | ⭐ | Aegis 反而太重 |
| Bug 诊断与修复 | ⭐⭐⭐⭐⭐ | systematic-debugging 7 层链路非常有效 |
| 多 Agent 并行开发 | ⭐⭐⭐⭐ | subagent-driven-development 解决协调问题 |
| 长周期项目 | ⭐⭐⭐⭐⭐ | long-task-continuation 防丢失 |
| 团队协作规范 | ⭐⭐⭐⭐ | evidence-driven 减少沟通成本 |
| 快速探索原型 | ⭐ | 工作流太重，不适合探索阶段 |

---

## 总结

Aegis 解决的是 AI 编程 Agent 最核心的信任问题：**当 Agent 说「完成了」，我们能不能信？**

它的答案是：
1. 不让它跳过 baseline reading 就动手
2. 不让它跳过验证命令就 claim 完成
3. 不让它修复 bug 时不处理旧逻辑
4. 不让它无限制地增加系统复杂度

本质上，Aegis 是在 AI 编程 Agent 周围建立一套**工程纪律护栏**，类似 TDD 是一种工程纪律，Aegis 是 AI 编程时代的工程纪律。

如果你用 AI 编程 Agent 做生产级项目，Aegis 值得一试。

---

## 相关阅读

- [Superpowers — Agent Skills 框架](https://github.com/obra/superpowers)
- [Claude Code — AI 编程 IDE](https://github.com/cline/cline)
- [项目根目录 CLAUDE.md 的 Authority 顺序](https://github.com/GanyuanRan/Aegis/blob/main/CLAUDE.md)
