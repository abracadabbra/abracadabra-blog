---
title: ExpertTeam-Codex 深度解析：把 Codex CLI 变成「AI 开发团队」
description: 深入剖析 ReJeCtAll/ExpertTeam-Codex 的核心架构——主理人编排模式、质量门禁设计、知识库体系和 Anti-Slop 评审框架
pubDate: 2026-06-13
tags: [AI编程, 多Agent协作, Codex, Agent工具链]
categories: [AI Agent]
heroImage: null
---

## 前言

最近在 GitHub Trending 上发现了一个很有意思的项目——[ReJeCtAll/ExpertTeam-Codex](https://github.com/ReJeCtAll/ExpertTeam-Codex)（⭐24，2026年6月8日创建），它把单个 Codex CLI 助手扩展成了一支完整的「AI 开发团队」：PM → 架构师 → 工程师 → QA，顺序编排，严格门禁。

这个项目的核心价值不在于「多 Agent」，而在于**如何让多 Agent 不乱成一锅粥**。它的解法是：主理人（Lead Agent）中转模式 + 铁律约束 + 量化质量门禁。让我深入读一遍源码，拆解它的核心原理。

---

## 一、项目定位：Skill 扩展包，不是独立产品

ExpertTeam-Codex 本质上是一套 **Codex CLI / Codex App 桌面版 Skills 安装包**，一键安装到 `~/.codex/`：

```bash
# 方式一：远程一键安装
curl -fsSL https://raw.githubusercontent.com/ReJeCtAll/ExpertTeam-Codex/main/install.sh | bash

# 方式二：克隆后审查再装
git clone https://github.com/ReJeCtAll/ExpertTeam-Codex.git
cd ExpertTeam-Codex && ./install.sh
```

安装后会在 `~/.codex/` 下生成三套配置文件：

```
~/.codex/
├── skills/       ← Skill 入口（$expert-software 等）
├── agents/       ← 19个 Agent 角色定义文件
└── commands/     ← 可选 Slash Command 兼容层
```

调用方式非常直观：

```text
$expert-team 软件开发团队帮我做一个 Todo App
$expert-software --fast 做一个带本地存储的 Todo Web App
$expert-design --full 为 AI Agent 平台设计高保真 Landing Page
$expert-security --threat 对用户认证模块做 STRIDE 威胁建模
```

---

## 二、架构总览：6 个入口，3 类模式

```
$expert-team (总路由器)
├── $expert-software  ← 团队型，5人（PM+架构师+工程师+QA+主理人）
├── $expert-design    ← 团队型，6人（主理人+需求发现+设计系统+原型+审查+导出）
├── $expert-product   ← 团队型，6人（产品总监+需求分析+用户研究+竞品+数据+路线图）
├── $expert-ops       ← 单专家型，1人（运维通）
└── $expert-security  ← 单专家型，1人（安全专家）
```

这里有个关键区分：**团队型 vs 单专家型**。

- **团队型**（software/design/product）：有完整 Lead，主理人创建团队、编排任务、中转信息、汇总交付
- **单专家型**（ops/security）：不虚构多角色团队，直接进入专项工作流

这个区分很重要——它避免了「为了多 Agent 而多 Agent」，Ops 和 Security 这种专项任务，一个专家就够了。

---

## 三、核心机制：主理人编排模式

### 3.1 铁律（绝对禁止）

读 `software-team-lead.md` 时最震撼的是它的「铁律」部分——不是建议，是强制：

```markdown
## 团队协作机制（铁律）

1. **建立团队**：任务开始时由主理人亲自创建本次任务的团队。
   团队创建（TeamCreate）必须且只能由主理人执行，严禁委派任何成员创建团队

2. **调度成员**：按 SOP 阶段将每位团队成员拉入协作、下发独立任务；
   团队成员作为独立协作方基于任务说明输出专业产出，不得由主理人代写

3. **消息中转**：成员的产出需回传给你，由你汇总、转交给下一阶段成员；
   所有跨成员的信息流必须经主理人中转，不得互相直连

4. **成员结论为准**：任何专业产出必须由对应成员输出后再采信，
   主理人只做编排与汇编

### 严禁行为
❌ 禁止跳过"建立团队"的正式流程
❌ 禁止自己代写任何团队成员的专业产出
❌ 禁止跳过前序阶段直接进入后续阶段
❌ 禁止让成员互相直连通信
```

这四条铁律直击多 Agent 协作最常见的腐烂方式：**Agent 们跳过了真正的协作，直接模拟对方发言**。

### 3.2 子任务命名约束

调度子 Agent 时，必须在 Agent 工具的 `name` 参数中传入该成员的 **Agent ID**（文件名），同时 `subagent_type` 也传入相同的 ID：

```markdown
<!-- 正确写法 -->
name: "software-architect", subagent_type: "software-architect"
name: "software-engineer", subagent_type: "software-engineer"

<!-- 禁止写法：省略 name / 使用中文名 / 自创名称 -->
```

这个约束保证了主理人和子 Agent 的身份一致性——Codex CLI 的 Agent Team 机制是按 name 做身份路由的。

### 3.3 三档工作流路由

同一个入口，根据任务规模自动选择路径：

```markdown
## 工作流路由（主理人收到请求时首先判断）

| 场景        | 判断标准                        | 使用工作流       |
|-------------|-------------------------------|---------------|
| 小型需求     | ≤10个源文件，单页面应用/工具脚本   | ⚡ 快速模式      |
| Bug 修复    | 用户报告明确 Bug，非新功能         | 🔧 BugFix     |
| 中大型需求   | 多模块/涉及后端+前端/>10个源文件   | 🏗️ 标准 SOP    |
| 仅需分析    | 仅 PRD/架构评审                  | 📋 部分工作流    |
```

**快速模式**是最有趣的设计——很多需求其实不需要走完整流程：

```
用户需求 → TeamCreate → 工程师(直接实现全部代码) → QA工程师(验证)
```

跳过 PRD 和架构设计，工程师一次性写完全部文件，QA 一轮验证就交付。这才是大多数小型需求的正确处理方式。

---

## 四、质量门禁体系：量化 + 可落地

ExpertTeam-Codex 的质量门禁不是泛泛的「要保证质量」，而是一套**可以客观判断是否通过**的量化体系。

### 4.1 软件团队的 IS_PASS 机制

工程师（Alex）完成所有文件后，必须执行全局一致性审查：

```markdown
## 全局一致性审查（after ALL tasks complete）

1. Read through all generated files as a whole
2. Check for:
   - Cross-file import consistency（无缺失 import，无循环依赖）
   - Interface contract compliance（所有调用方使用正确的 method signatures）
   - Data flow correctness（模块间传递的对象类型/字段正确）
   - No duplicate implementations
3. Output verdict:
   - **IS_PASS: YES** → 代码交给 QA
   - **IS_PASS: NO** → 列出问题，修复后重新审查（最多2轮）
```

**IS_PASS: NO → 修复 → IS_PASS 再次审查**，最多两轮。这防止了无限 debug 循环。

### 4.2 QA 的智能路由判定

QA 工程师（Edward）跑完测试后，必须做出**三选一路由决策**：

```markdown
## 智能路由决策

- **Send To: Engineer (Alex)**
  → 源码有 Bug，测试正确，实现有问题
  → 报告：哪个测试失败、期望 vs 实际、具体文件和函数

- **Send To: QA (self)**
  → 测试代码有 Bug，源码正确，测试本身写错了
  → QA 自己修复测试

- **Send To: NoOne**
  → 全部通过，报告成功
```

这个设计的精妙在于：**QA 不负责调试，QA 只负责判断问题出在哪里**，然后把问题路由给正确的角色。

同时，**最多 2 轮测试**（不是 5 轮）：

```markdown
Round 1: 写测试 → 跑测试 → 发现源码 Bug → 发给 Engineer 修复
Round 2: 回归测试 → 仍失败 → 记录为 Known Issues，退出循环

❌ 不进入 Round 3。两轮是硬性上限。
```

### 4.3 设计团队的 5 维评审 + Anti-Slop

设计团队的质量门禁更复杂，分两层：

**第一层：5 维度评分**（每维度 1-5 分，必须每个 ≥ 3 分）

- 功能完整性
- 视觉质量
- 交互体验
- 技术实现
- 品牌一致性

**第二层：Anti-Slop 检测**

Anti-Slop（防 AI 廉价感）是这个项目最有意思的设计。它定义了一套 P0/P1/P2 的「AI 设计反模式」清单：

```markdown
## P0 — 必须修复（阻断发布）

视觉反模式：
❌ 紫色/彩虹渐变背景 — AI 最爱的"高级感"捷径，实际最廉价
❌ 通用 emoji 作为图标 — 用 🚀 表示速度、💡 表示想法
❌ 圆角卡片 + 左侧彩色边框 — AI 套路万金油组件
❌ 手绘风 SVG 人物插图 — generic AI illustration style

内容反模式：
❌ 编造统计数据 — "提升300%效率"、无来源数字
❌ 虚假社会证明 — 杜撰的用户评价、logo墙

## P1 — 累计3条以上建议打回修正

排版：Inter 作为展示字体、段落宽度超过75字符
色彩：同页面4+种色调、纯黑文本色
布局：留白不足、CTA按钮过多、Hero区域内容过载
```

**通过标准**：

```markdown
✅ 5维度每个维度 ≥ 3分
✅ P0 问题数 = 0（P0 是阻断发布的）
✅ P1 问题数 ≤ 3
```

任意一项不满足，原型构建师必须修正，最多 2 轮。

---

## 五、知识库体系：71 套设计系统 + 产品手册

ExpertTeam-Codex 不仅有 Agent 角色，还配套了专门的知识库 Skill：

### 5.1 design-systems：71 套设计系统知识库

```
类别         数量    代表品牌
AI & LLM     10     Claude, Cohere, Mistral AI, X.AI
开发工具      19     Cursor, Vercel, Linear, Supabase
生产力        10     Notion, Figma, Miro, Raycast
金融科技       7     Stripe, Coinbase, Mastercard
电商消费       6     Shopify, Airbnb, Nike
媒体          5     Spotify, PlayStation, Meta
汽车          6     Tesla, BMW, Ferrari
其他/通用     8     Apple, SpaceX, Default, Warm Editorial
```

每套系统包含 **9 段标准结构**：视觉主题、色彩、排版、组件、布局、深度、注意事项、响应式、Agent 指南。

### 5.2 product-playbook：产品管理手册

产品团队的支撑 Skill，提供了完整的产品管理 SOP。

### 5.3 quality-review：质量评审规则

包含 `anti-slop-checklist.md` 和 `critique-5d-rubric.md`，是设计质量门禁的执行依据。

---

## 六、中文角色命名：人物设定感

最有意思的细节是它的**角色命名全部有中文名字 + 英文别名**：

| 角色 | 中文名 | 英文别名 | 职责 |
|------|--------|---------|------|
| 软件团队主理人 | 齐活林（Qi） | Delivery Director | 协调编排，汇总交付 |
| 架构师 | 高见远（Gao） | Bob | 系统设计 + 任务分解 |
| 工程师 | 寇豆码（Kou） | Alex | 批量编码，全局一致性审查 |
| QA | 严过关（Yan） | Edward | 测试 + 智能路由判定 |
| 产品总监 | 方向明（Fang） | Product Helmsman | 产品战略编排 |
| 运维专家 | 运维通（Tia） | Infrastructure Expert | 可靠性/成本/备份 |

这些名字不是随机生成的——高见远（看得远）、严过关（严格质量关）、运维通（运维全能），名字和角色职责有语义关联。这种细节提升了 AI 角色扮演的真实感。

---

## 七、跨团队流水线：从想法到上线

USAGE.md 提供了一套完整的跨团队协作流水线：

```text
$expert-product --prd <产品想法>
        ↓
$expert-security --protect <隐私、安全、合规要求>
        ↓
$expert-design --full <基于 PRD 做高保真原型>
        ↓
$expert-software --standard <基于 PRD 和原型实现>
        ↓
$expert-security --code-review <上线前安全审查>
        ↓
$expert-ops --full <部署、监控、运维方案>
```

安全专家在产品阶段提供安全输入，在上线前做威胁建模，中间穿插代码审计——这才是真正的安全左移。

---

## 八、与同类工具的横向对比

| | **Trellis** | **vibecode-pro-max-kit** | **ExpertTeam-Codex** |
|---|---|---|---|
| **定位** | 代码治理/能力合同/Budget | 开发流程/RIPER-5 流水线 | 多角色团队编排 |
| **规模** | 单框架 | 12 Agent + 31 Skill | 19 Agent + 6 Team |
| **主轴** | 长线代码质量保障 | 开发过程纪律强制 | 需求→交付完整链路 |
| **质量门禁** | 能力合同（Prophet） | RIPER-5 阶段锁 | IS_PASS + QA路由 + 5D评审 |
| **语言** | 英文 | 英文 | **中文全员** |
| **入口** | CLI | CLI | CLI / Skill 调用 |

**三者是正交的**：Trellis 管「代码长期质量」，vibecode 管「开发过程纪律」，ExpertTeam-Codex 管「多角色协作流程」，可以叠加使用。

---

## 九、核心设计思想总结

读完全部源码后，我认为 ExpertTeam-Codex 最值得学习的不是它的多 Agent 架构，而是**它对多 Agent 协作腐烂问题的系统性解法**：

1. **主理人垄断消息中转** — 禁止成员直连，所有信息流经过 Lead，防止「假协作」
2. **铁律约束而非建议** — 四条铁律写死在 prompt 里，不是「建议这么做」而是「禁止那么做」
3. **量化质量门禁** — IS_PASS / 5D评分 / Anti-Slop，每道门禁都有客观判断标准
4. **两轮上限** — 防止无限循环，保证交付节奏
5. **团队型 vs 单专家型区分** — 不为单一职责虚构团队，避免过度工程

这套设计对任何想搭建「多 Agent 协作系统」的人都有参考价值——不管是搭开发团队、还是搭自动化流水线。

---

*项目地址：[ReJeCtAll/ExpertTeam-Codex](https://github.com/ReJeCtAll/ExpertTeam-Codex)*  
*相关博客：[Trellis 实战](/blog/trellis-ai编程agent代码治理标准实战/)、[Oh-My-ClaudeCode 全解析](/blog/Oh-My-ClaudeCode-19个Agent角色全解析/)*
