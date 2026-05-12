---
title: 'Oh My ClaudeCode Team 模式深度解析：多 Agent 协作的艺术'
description: '深入剖析 Oh My ClaudeCode 的 Team 编排模式，从五阶段流水线到实战案例，解锁 Claude Code 多 Agent 协作的正确姿势'
pubDate: '2026-05-12 10:30:00'
tags: ["AI工具", "Claude", "多Agent", "Oh-My-ClaudeCode", "编程效率"]
categories: ["AI工具"]
---

## 前言

用 Claude Code 写代码，一个人干到黑？太慢了。

**Oh My ClaudeCode (OMC)** 给出的答案是：**让多个 AI Agent 组队干活**。而 Team 模式，就是这套多 Agent 协作的核心编排方式。

这篇文章带你深入理解 Team 模式的五阶段流水线、19 个专业 Agent 的分工协作，以及如何用一行命令指挥一支 AI 开发团队。

## 什么是 Oh My ClaudeCode？

先快速过一下背景。

**Oh My ClaudeCode** 是一个 Claude Code 的多智能体编排框架，口号是：

> *Don't learn Claude Code. Just use OMC.*

它解决了 Claude Code 原生的几个痛点：

- 复杂任务需要手动分解
- 单 Agent 执行效率低
- 没有内置的验证和修复循环
- 跨会话状态丢失

安装很简单：

```bash
# 方式一：Plugin Marketplace（推荐）
/plugin marketplace add https://github.com/Yeachan-Heo/oh-my-claudecode
/plugin install oh-my-claudecode

# 方式二：npm
npm i -g oh-my-claude-sisyphus@latest
```

装完跑一下 `/omc-setup` 就能用了。

## Team 模式：标准编排方式

从 v4.1.7 开始，**Team** 是 OMC 的标准多 Agent 编排界面。之前叫 `swarm`、`ultrapilot`，现在统一用 `team`。

### 基本用法

```bash
/team 3:executor "fix all TypeScript errors"
```

这行命令的意思是：**启动 3 个 executor Agent，并行修复所有 TypeScript 错误**。

语法是 `/team N:agent-name "任务描述"`，其中：
- `N` — 启动的 Agent 数量
- `agent-name` — 使用的 Agent 类型
- `"任务描述"` — 要完成的任务

### 启用 Team 模式

Team 模式需要 Claude Code 的实验性功能支持，在 `~/.claude/settings.json` 中添加：

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

如果没启用，OMC 会发出警告并回退到非 Team 执行模式。

## 五阶段流水线

Team 模式的核心是一条**五阶段流水线**：

```
team-plan → team-prd → team-exec → team-verify → team-fix (循环)
```

每个阶段都有明确的职责和对应的 Agent。

### 阶段一：team-plan（规划）

**负责 Agent**：`planner` + `architect`

这一阶段的任务是：
1. 分析任务需求
2. 拆解成可执行的子任务
3. 确定任务间的依赖关系
4. 制定执行计划

```
输入："实现一个完整的用户认证系统"

输出：
├── 子任务 1：设计数据库 schema（依赖：无）
├── 子任务 2：实现 JWT 工具类（依赖：无）
├── 子任务 3：实现注册接口（依赖：1, 2）
├── 子任务 4：实现登录接口（依赖：1, 2）
├── 子任务 5：实现中间件（依赖：2）
└── 子任务 6：编写测试（依赖：3, 4, 5）
```

### 阶段二：team-prd（产品需求文档）

**负责 Agent**：`analyst` + `writer`

这一阶段把规划转化为结构化的 PRD：
- 接口定义
- 数据结构
- 错误处理策略
- 验收标准

这一步看起来"多余"，但很重要——它让每个 Agent 都有明确的目标和边界。

### 阶段三：team-exec（执行）

**负责 Agent**：`executor`（可多个并行）

这是实际干活的阶段。多个 `executor` Agent 根据 PRD 并行实现代码。

```
executor-1 → 实现 JWT 工具类
executor-2 → 设计数据库 schema
executor-3 → （等待 1、2 完成）实现注册接口
```

### 阶段四：team-verify（验证）

**负责 Agent**：`verifier` + `test-engineer`

执行完成后，验证阶段检查：
- 代码是否能编译通过
- 测试是否全部通过
- 功能是否符合 PRD 定义
- 是否有遗漏的边界情况

```bash
# 验证检查项
✓ BUILD: 编译通过
✓ TEST: 所有测试通过
✓ LINT: 无 lint 错误
✓ FUNCTIONALITY: 功能符合预期
✓ TODO: 所有子任务完成
```

### 阶段五：team-fix（修复循环）

**负责 Agent**：`debugger` + `executor`

如果验证阶段发现问题，进入修复循环：
1. 定位问题根因
2. 修复代码
3. 重新验证
4. 重复直到通过

这个循环是 Team 模式的关键——**它不会半途而废**。

## 19 个专业 Agent

OMC 提供了 19 个专业 Agent，分为 4 个通道：

### 构建/分析通道

| Agent | 默认模型 | 职责 |
|-------|---------|------|
| `explore` | Haiku | 代码库发现、文件映射 |
| `analyst` | Opus | 需求分析、发现隐藏约束 |
| `planner` | Opus | 任务排序、创建执行计划 |
| `architect` | Opus | 系统设计、接口定义 |
| `debugger` | Sonnet | 根因分析、错误修复 |
| `executor` | Sonnet | 代码实现、重构 |
| `verifier` | Sonnet | 完成验证 |
| `tracer` | Sonnet | 因果追踪 |

### 审查通道

| Agent | 默认模型 | 职责 |
|-------|---------|------|
| `security-reviewer` | Sonnet | 安全漏洞审查 |
| `code-reviewer` | Opus | 全面代码审查 |

### 领域通道

| Agent | 默认模型 | 职责 |
|-------|---------|------|
| `test-engineer` | Sonnet | 测试策略、覆盖率 |
| `designer` | Sonnet | UI/UX 设计 |
| `writer` | Haiku | 文档编写 |
| `qa-tester` | Sonnet | 运行时验证 |
| `scientist` | Sonnet | 数据分析 |
| `git-master` | Sonnet | Git 操作 |
| `document-specialist` | Sonnet | 外部文档查找 |
| `code-simplifier` | Opus | 代码简化 |

### 协调通道

| Agent | 默认模型 | 职责 |
|-------|---------|------|
| `critic` | Opus | 计划审查、差距分析 |

### 智能模型路由

OMC 根据任务复杂度自动选择模型：

| 层级 | 模式 | 适用场景 |
|------|------|---------|
| LOW | Haiku | 快速查找、简单任务 |
| MEDIUM | Sonnet | 代码实现、调试、测试 |
| HIGH | Opus | 架构设计、战略分析 |

这样既能保证质量，又能控制成本——**简单任务不浪费 Opus，复杂任务不用 Haiku 凑合**。

## 实战案例

### 案例一：构建 REST API

```bash
/team 5:executor "build a REST API for task management with auth, CRUD operations, and tests"
```

Team 流水线自动执行：

```
1. [plan] planner 拆解任务
   ├── 设计数据库 schema
   ├── 实现认证中间件
   ├── 实现 CRUD 路由
   ├── 编写单元测试
   └── 编写 API 文档

2. [prd] analyst 输出接口定义
   - POST /auth/register
   - POST /auth/login
   - GET /tasks
   - POST /tasks
   - PUT /tasks/:id
   - DELETE /tasks/:id

3. [exec] 5 个 executor 并行工作
   - executor-1: 数据库 schema + 迁移
   - executor-2: 认证模块
   - executor-3: 任务 CRUD 路由
   - executor-4: 单元测试
   - executor-5: API 文档

4. [verify] verifier 检查
   ✓ 编译通过
   ✓ 测试全绿
   ✓ API 文档完整

5. [fix] 无问题，完成！
```

### 案例二：修复遗留代码

```bash
/team 3:executor "refactor the authentication module to use JWT, add proper error handling, and write tests"
```

这个任务需要顺序执行（有依赖关系），Team 会自动调整：

```
1. executor-1: 分析现有认证模块
2. executor-2: 重构为 JWT（依赖分析结果）
3. executor-3: 添加错误处理 + 测试（依赖重构完成）
```

### 案例三：混合使用不同 Agent

```bash
# 架构师设计，开发者实现，审查员把关
/team 1:architect "design microservice architecture"
/team 3:executor "implement the services"
/team 1:code-reviewer "review all changes"
```

## 与其他模式对比

OMC 提供了多种编排模式，Team 是最推荐的，但不是唯一选择：

| 模式 | 特点 | 适用场景 |
|------|------|---------|
| **Team** | 五阶段流水线 | 复杂任务、需要协调 |
| **Autopilot** | 全自动执行 | 端到端功能开发 |
| **Ultrawork** | 最大并行 | 可并行的独立任务 |
| **Ralph** | 持久模式 | 必须完成的任务 |
| **ccg** | 三模型协作 | 需要多视角分析 |

**选择建议**：
- 任务复杂、需要协调 → **Team**
- 任务明确、想省心 → **Autopilot**
- 任务可并行 → **Ultrawork**
- 任务必须完成 → **Ralph**
- 需要多模型视角 → **ccg**

## 最佳实践

### 1. 任务描述要具体

```bash
# ❌ 太模糊
/team 3:executor "fix the code"

# ✅ 明确具体
/team 3:executor "fix all TypeScript type errors in src/api/ and src/models/"
```

### 2. Agent 数量要合理

- **1-2 个**：简单任务、顺序依赖
- **3-5 个**：复杂任务、可并行
- **5+ 个**：大型项目、明确分工

超过 5 个 Agent 后收益递减，还会增加协调开销。

### 3. 善用 verify 阶段

Team 的 verify 阶段是质量保证的关键。如果任务很重要，可以让 verify 更严格：

```bash
/team 3:executor "implement payment flow" --verify strict
```

### 4. 结合技能系统

Team 可以和其他技能组合：

```bash
# Team + git-master（自动提交）
/team 3:executor "implement feature X" + git-master

# Team + test-engineer（测试优先）
/team 3:executor "implement feature X" + test-engineer
```

### 5. 监控执行状态

使用 HUD 实时监控：

```bash
/oh-my-claudecode:hud setup
```

状态栏会显示：
- 当前阶段
- 活跃 Agent 数量
- Token 消耗
- 预计完成时间

## 踩坑记录

### 坑一：没启用实验性功能

```
Error: Agent teams are disabled
```

**解决**：在 `~/.claude/settings.json` 中添加 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

### 坑二：任务描述太模糊

Agent 可能会"自由发挥"，做出不符合预期的代码。

**解决**：任务描述要具体，最好包含验收标准。

### 坑三：Agent 数量过多

5 个以上 Agent 并行时，协调开销可能超过并行收益。

**解决**：控制在 3-5 个，复杂任务用流水线拆解。

### 坑四：忽略 verify 阶段

有些开发者看到代码生成了就直接用，跳过验证。

**解决**：让 verify 跑完，发现问题及时修复。

## 总结

Oh My ClaudeCode 的 Team 模式，本质上是一套**自动化的多 Agent 协作流水线**：

1. **规划**：拆解任务、制定计划
2. **PRD**：明确需求、定义接口
3. **执行**：并行实现、高效开发
4. **验证**：质量保证、发现问题
5. **修复**：自动修复、循环直到通过

它的价值不是"让 AI 写代码"，而是**让 AI 像一个团队一样协作**——有分工、有协调、有验证、有修复。

如果你在用 Claude Code，强烈建议试试 OMC 的 Team 模式。一行命令，指挥一支 AI 开发团队。

---

**相关资源**：
- [Oh My ClaudeCode GitHub](https://github.com/yeachan-heo/oh-my-claudecode)
- [官方文档](https://yeachan-heo.github.io/oh-my-claudecode-website)
- [Discord 社区](https://discord.gg/PUwSMR9XNk)
