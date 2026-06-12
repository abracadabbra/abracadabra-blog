---
title: 'Trellis 实战：用「能力合同」让 AI 编程 Agent 学会自我约束'
description: '深入解析 Trellis 治理标准：能力合同、证据图、预算控制、ADR 生命周期，配合 Pi 实现个人项目开发工作流。'
pubDate: '2026-06-12'
tags: ["AI", "AI Agent", "Trellis", "Pi", "代码治理", "方法论", "Claude Code"]
categories: ["AI开发"]
---

> AI 写代码已经不难了，难的是**让代码保持可维护**。

---

## 什么是 Trellis

**Trellis**（[e-onux/trellis](https://github.com/e-onux/trellis)）是一个**与 Agent 无关的代码治理标准**，不是编码工具，不替代 Claude Code 或 Pi，而是叠加在任何编程 Agent 上面，给 AI 生成的代码加一层治理结构。

官网：https://trellis.sidre.site （有向导界面）

```
Trellis 的核心使命：
AI 写代码 → 写得快 → 但代码容易膨胀、架构漂移、忘记为什么这么做

Trellis = 让 AI 生成的软件保持可理解、可测试、可维护
```

---

## 为什么需要 Trellis

当一个 Agent 持续开发一个项目时，常见问题：

- **无界膨胀** — 重复职责、失控依赖、文件越来越大
- **架构漂移** — 局部合理的改动全局不一致
- **丢失「为什么」** — 决策在聊天记录里，不在代码库里
- **测试隐藏** — 只有工程师知道功能是否真正可用
- **忘记来源** — 技术选型的理由比决策活得更久

没有治理的情况下，AI 生成代码的维护成本会随时间急剧上升。Trellis 的思路是：**在代码库里建立一套持久化的治理结构**，让 AI 每次工作时都能看到项目的「记忆」和「边界」。

---

## 核心概念

### 1. 能力合同（Capability Contract）

每个有意义的功能都有一个**机器可检查的合同**：

```yaml
# capabilities/calculate-shipping-cost/contract.yaml
intent: 计算最优运费
responsibility:
  does:
    - 根据重量和距离计算运费
    - 选择最低成本承运商
  does_not:
    - 不处理支付
    - 不做库存检查
inputs:
  - 重量 (kg)
  - 起点 / 终点
outputs:
  - 运费报价
  - 承运商名称
examples:
  normal:
    - input: {重量: 5kg, from: 上海, to: 北京}
    - output: {cost: 12.5, carrier: 顺丰}
  error:
    - input: {重量: 500kg, from: 上海, to: 北京}
    - output: {error: 超出配送范围}
budgets:
  max_files: 5
  max_LOC: 300
  max_dependencies: 2
```

合同里定义了「做什么、不做什么、输入输出示例、代码膨胀上限」。

### 2. 能力预算（Capability Budget）

这是 Trellis 最独特的设计：**每个能力有硬性上限**。

```yaml
budgets:
  max_files: 5       # 不超过 5 个文件
  max_LOC: 300       # 不超过 300 行
  max_dependencies: 2 # 不超过 2 个依赖
```

Agent 遇到新需求时，必须判断：

```
这个需求在当前能力的 does 范围内吗？
  是 → 在预算内吗？→ 扩展能力
        → 超出预算？→ 停止，写「拆分/重构提案」
  否 → 这是新能力吗？→ 提议创建新能力
```

**Trellis 要求 Agent 在该停的时候停下来写提案，而不是继续埋头写代码。**

### 3. 证据图（Evidence Graph）

```
Source（来源） → Claim（主张） → Decision（ADR） → Capability → 实现 → 测试 → 用户验证
```

每一次技术决策都有来源依据，不让 AI「凭直觉」做架构选择。

### 4. ADR 决策生命周期

架构决策记录（ADR）不是写完就完事了，而是有**复审触发条件**：

```yaml
assumptions:          # 做决策时的假设
  - 用户主要在移动端访问
review_after: 6个月   # 6 个月后复审
review_triggers:      # 提前复审的条件
  - 移动端流量超过 50%
  - 引入新的支付渠道
```

### 5. 仓库原生记忆

```
聊天记录里的信息 → 归零
代码库里的信息 → 永久保留
```

Trellis 把决策、合同、证据都放在仓库的 `governance/`、`tech/`、`capabilities/` 目录里，Agent 每次会话都能读取，不依赖上下文窗口。

---

## 目录结构

Trellis 在仓库里建立这套治理结构：

```
├── TRELLIS.md              ← 一句话 bootstrap 入口
├── AGENTS.md               ← Agent 指令（canonical）
├── CLAUDE.md               ← Claude Code 指针 → AGENTS.md
├── governance/             ← 章程、工程原则、Agent 权限
├── product/                ← 产品愿景、约束、领域模型
├── tech/                   ← 架构概览、技术雷达
│   └── decisions/          ← ADR（架构决策记录）
├── sources/                ← 证据库（ bibliography.yaml）
├── extensions/              ← 扩展注册表
├── capabilities/            ← 能力合同（每个功能一个目录）
│   └── calculate-shipping-cost/
│       ├── contract.yaml   ← 能力合同
│       ├── examples/       ← 输入输出示例
│       └── tests/          ← 验收测试
├── quality/                ← 测试策略、安全策略、质量门控
└── lifecycle/             ← 复审日历、技术债务、升级计划
```

---

## 快速起步：零成本（一行 curl）

Trellis 最优雅的地方在于**不需要安装任何东西**：

```bash
# 1. 把 bootstrap 文件放进仓库
curl -o TRELLIS.md https://raw.githubusercontent.com/e-onux/trellis/main/TRELLIS.md

# 2. 告诉你的 Agent
# "Read TRELLIS.md and bootstrap this repository according to it."
```

Agent 会自动：
1. 分析仓库技术栈
2. 选择 profile（backend / frontend / data-pipeline / llm-app）
3. 搭建治理目录骨架
4. 从现有代码提取架构决策（brownfield 项目）
5. 发现能力并创建合同
6. 生成各平台的适配文件

---

## Trellis vs 类似工具对比

| 工具 | 定位 | Trellis 的优势 |
|------|------|----------------|
| **GitHub Spec Kit** | spec → plan → tasks | + ADR 生命周期、预算控制、证据图 |
| **OpenSpec** | 轻量变更管理 | + 能力预算、CI 质量门控 |
| **BMAD-METHOD** | 多 Agent SDLC | + 与 persona 无关的仓库标准 |
| **Kiro** | IDE 内 spec 驱动 | + 开放标准、跨 IDE |
| **Tessl** | 商业 spec 注册表 | + Apache 2.0 开放标准 |
| **无治理** | — | AI 代码膨胀、架构漂移、决策丢失 |

---

## 实战：Pi + Trellis 个人项目工作流

基于 linux.do 佬友分享的真实工作流：

### 步骤 1：GitHub 找参考项目

先在 GitHub 找类似项目，提取 spec。不自己凭空设计。

### 步骤 2：grill-me 梳理需求

用 AI 工具把需求整理清楚。虽然 Trellis 可以做能力发现，但提前梳理需求能让后续更顺滑。

### 步骤 3：Trellis 初始化

```bash
# 进入项目目录
cd my-project

# 拉 Trellis bootstrap
curl -o TRELLIS.md https://raw.githubusercontent.com/e-onux/trellis/main/TRELLIS.md

# 让 Pi 初始化
pi
# "Read TRELLIS.md and bootstrap this repository according to it."
```

Pi 会分析仓库、创建治理骨架、发现能力并生成合同。

### 步骤 4：Claude 规划 + GPT 实现 + Claude Review

```
Claude（规划） → GPT（实现） → Claude（review） → 有问题就继续 → 没问题就提交
```

### 步骤 5：Trellis 验收

```bash
# 验证能力合同
npx @e-onux/trellis validate

# 检查预算
npx @e-onux/trellis budget-check

# 整体审计
npx @e-onux/trellis audit
```

### 步骤 6：整理记忆到 Obsidian

功能完成并验证后，让 Claude 整理：

- 这个项目值得学习的地方
- 项目的长期记忆（架构决策、关键设计）
- 同步到 Obsidian

以后 AI 在这个项目里跑偏了，让它先查 Obsidian 里的记忆，自己纠错。

---

## Trellis 的渐进式采用

不需要一天之内全套上马：

```
Level 1  治理 + ADR                （项目记忆）
Level 2  能力合同                  （可执行的规格）
Level 3  自动化测试               （单元/合同/属性测试）
Level 4  用户测试驾驶舱            （非工程师可验证）
Level 5  能力 + 架构预算           （防膨胀强制执行）
Level 6  生命周期 + 升级自动化    （决策不腐坏）
```

从 Level 1 开始，边用边加。

---

## 适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 个人 side project | ⭐⭐⭐⭐⭐ | 防止代码失控，保持可维护 |
| 多人协作项目 | ⭐⭐⭐⭐ | 统一决策记录，新人容易上手 |
| 快速原型 | ⭐⭐ | 治理层对探索阶段太重 |
| AI 原生项目 | ⭐⭐⭐⭐⭐ | Trellis 本身就是为 AI 生成的代码设计 |
| 已有大型项目 | ⭐⭐⭐ | 渐进式引入，从 ADR 记忆开始 |

---

## 总结

Trellis 解决的是 AI 编程时代的新问题：**代码生成越来越便宜，但代码维护越来越贵**。

它的核心洞察是：**Spec 驱动回答「做什么」，Trellis 保留「为什么这样做、允许多少增长、如何验证」**。

配合 Pi 的极简 harness + Trellis 的治理结构，可能是目前**最可控的个人 AI 编程组合**。

---

## 相关阅读

- [Trellis 官网向导](https://trellis.sidre.site)
- [e-onux/trellis GitHub](https://github.com/e-onux/trellis)
- [Pi Coding Agent](https://github.com/earendil-works/pi)
- [badlogic/pi-skills](https://github.com/badlogic/pi-skills)
