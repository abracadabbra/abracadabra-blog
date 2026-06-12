---
title: 'Pi Coding Agent 插件生态全景：从极简 harness 到高效编程利器'
description: '整合两篇 linux.do 精品帖子，系统梳理 Pi Coding Agent 的插件生态、极简哲学与实战经验。'
pubDate: '2026-06-05'
tags: ["AI", "AI Agent", "Pi", "编程工具", "插件生态", "上下文工程"]
categories: ["AI开发"]
---

> 两篇 linux.do 精品帖子激发了写这篇文章的灵感。Pi 是一个极度精简的 Agent 框架，但它的高度可塑性让插件生态异常丰富。这篇文章整合两位佬友的实战经验，带你从入门到精通。

---

## 什么是 Pi

Pi（[earendil-works/pi](https://github.com/earendil-works/pi)）是一个终端编程 Agent，跟 Claude Code / Codex / OpenCode 是同类产品，但它走了**极简路线**：

```yaml
⭐ 61,940 | TypeScript | MIT
创建: 2025-08-09
内置工具: 4 个（read / write / edit / bash）
System Prompt: < 1000 token
```

**Pi 刻意跳过的功能**：
- ❌ 无 MCP（需要自己写 Package 桥接）
- ❌ 无内置 to-do / plan mode
- ❌ 无子 Agent（需要单独装 Package）
- ❌ 无后台 bash
- ❌ 无内置 MCP Server

这种「毛坯房」设计让 Pi 的 **System Prompt 开销极小**，上下文完全由用户掌控。

🔗 官网: https://pi.dev

---

## Pi 的 Package 系统

在 Pi 中，一切皆可扩展。Package 可以是：

| 类型 | 说明 |
|------|------|
| **Extension** | 注册新命令、工具、事件钩子、UI 组件 |
| **Skill** | 带 SKILL.md 的技能文档，指导模型用特定工作流 |
| **Prompt Template** | 提示词模板 |
| **Theme** | 终端主题 JSON |

安装方式二选一：

```bash
# NPM 包
pi install npm:pi-subagents

# Git 仓库
pi install git:github.com/user/repo
```

---

## 核心理念：极简是美德

两位佬友都强调了同一个观点：

> 「功能复杂往往导致架构和实现复杂，维护困难。全用 AI 改不会真有人喜欢纯 vibe 写的屎山吧。」

**选插件的原则**：
1. 先装通用基础插件，再按需增加
2. 优先选**独立、轻量、依赖少**的插件
3. 大而全的包办类型插件质量往往不高
4. 下载量高不等于好用，得看实际需求
5. 根据不同项目**选择性开启**插件

---

## 共识推荐：两篇帖子都提到的插件

先列两篇帖子的最大公约数——两位佬都认为值得装的：

### `pi-fff` — 极速模糊搜索替代 find/grep

⭐ 最受欢迎的文件搜索插件，后端是 Rust/SIMD 加速的 `fd` 和 `rg`，注册 `fffind` 和 `ffgrep` 两个工具。

**优点**：
- 速度极快，索引后查询毫秒级
- 显示 git 状态
- 上位替代原生 find/grep

**注意**：这个扩展会覆盖 Pi 内置的 `grep`/`find`/`ls`，需要用 `--exclude-tools find,grep` 参数启动以避免工具名冲突：

```bash
pi --exclude-tools find,grep
```

---

### `pi-tool-display` — 工具输出美化

让 Pi 的 tool call block 变得简洁，edit diff 记录更美观。

**效果**：根据终端宽度自动切换 split/unified diff 视图。

**注意**：会覆盖注册 Pi 内置工具，可能跟其他调整了内置工具的扩展冲突。

---

### `pi-btw` — 并行旁路思考

体验接近 Claude Code 的 `/btw`，在 TUI 上覆盖一个窗口进行并行旁路思考，**不污染主对话**。

支持 TUI markdown 渲染，风格与 Pi 原生契合度高。

**注意**：不同 btw 对话之间**没有共享上下文**，且模型会明确拒绝执行工具（故意限制）。

---

## 上下文管理：让模型始终工作在高召回率区间

这是第一篇帖子的核心关注点——上下文膨胀直接影响模型表现。

### `pi-observational-memory` — 观察者记忆

后台跑 observer 和 reflector，将精简后的对话记录和用户提过的重要要求固化到 session 里。

```text
触发时机：按 token 数自动触发
    ↓
Observer 记录关键上下文
    ↓
Reflector 提炼并固化到 session
    ↓
配合 Pi 内置 compact
    ↓
LLM 始终工作在召回率较高区间
```

### `DCP（Dynamic Context Pruning）` — 动态上下文剪枝

比较常见的总结类工具，定期压缩上下文。适合**长会话防偏移**。

### `context-mode` — 沙箱摘要模式

拦截工具输出到沙箱，模型只能看到摘要，需要时才展开原文。

**第一篇帖子作者的反馈**：效果不好，模型经常判断错该不该展开，导致关键信息丢失。**最终卸载**。

---

## 目标管理：让 Agent 知道何时停止

### `pi-until-done` — 目标完成前不停止

类比 Codex 的 `/goal`，Agent 会持续工作直到目标完成才停止。

```bash
pi install npm:pi-until-done
```

### `pi-codex-goal` — 更接近 Codex 体验的 goal 模式

如果追求更接近 Codex `/goal` 的体验，装这个。

---

## 权限管理：安全底线

> ⚠️ 「Pi 默认不带任何权限管理以及沙盒。rm -rf / 也很丝滑，嗯。」

### `pi-permission-system` — 推荐配置

```json
{
  "permission": {
    "*": "allow",
    "grep": "deny",
    "find": "deny",
    "external_directory": {
      "*": "ask",
      "~/.pi/agent/sessions/*": "allow"
    },
    "path": {
      "*.env": "deny",
      "~/.ssh/*": "deny"
    },
    "bash": {
      "rm *": "ask",
      "git clean *": "ask",
      "git reset --hard *": "ask",
      "Remove-Item *": "ask"
    }
  }
}
```

**策略要点**：
- 拒绝访问 .env / SSH 密钥
- 删文件/重置类 bash 命令询问用户
- 即使设为 deny，LLM 有时也会想办法钻漏洞绕过，所以 bash 部分选 `ask` 而非 `deny`

**注意**：`pi-permission-system` 默认 fallback 是 `ask`，只要设为 `ask` 就会暴露全部默认工具。需要显式 `deny` 才能过滤掉 grep/find。

---

## 子 Agent：并行处理复杂任务

### `pi-subagents` — 推荐

与权限系统有联动，子代理触发权限规则时询问弹窗会转发到主界面。实现比较轻量。

**自带三个角色**：
- `general-purpose` — 通用复杂任务
- `Explore` — 快速仓库探索（只读）
- `Plan` — 架构规划（只读）

支持自定义新角色。

**另一个选项**：`pi-subagents`（下载量最高版），更偏向预定 agent 角色 + 预定工作流，可以手动配置不同 agent 的角色和模型。

---

## 搜索与抓取

### `pi-search` — 搜索增强包

基于站内 grok-search-mcp 二次开发，增加了 context7 和反检测 fetcher。搜索相关装这一个基本够用。

**替代选项**：`pi-web-access` / `pi-smart-fetch`

### `pi-ace-tool` — 代码搜索增强

`ace-tool` 用过的都知道有多好用，Pi 版同样出色。

---

## 后悔药：操作回退

### `pi-workspace-history` — 推荐（不需要 git 初始化）

用 shadow git 实现，不需要项目初始化 git。

```text
/undo — 撤销上一次整个对话以及文件更改
/redo — 重做
/tree — 切换节点时自动触发文件恢复
```

### `pi-rewind` — 需要 git 初始化

通过 git checkpoint 实现，接近 Claude Code 的 `/rewind` 效果。**注意**：几个月没更新了。

### `pi-wtf` — 消息撤回

输入 typo 或不小心回车后，`/fuck` 强制停止当前工作并通过 tree 撤回内容。

**有意思的 combo**：`/fuck` + `pi-workspace-history` 一起装时，`/fuck` 触发 tree 撤销会话，同时也会触发 `pi-workspace-history` 撤销文件更改——**奇妙联动**。

---

## 代码审查

### `@juicesharp/rpiv-advisor` — 请求强模型给第二意见

在关键决策前多一层校验。

### `pi-simplify` — 代码改动审查

审查近期代码改动的清晰度、维护性和一致性。

### `@narumitw/pi-plan-mode` — 只读规划模式

`/plan` 只读，禁止 edit/write/危险 bash，输出 proposed_plan 确认后才恢复写权限。

---

## UI 增强

### `pi-nano-context` — 紧凑上下文占用条

显示 system / user / assistant / tool / free 各占多少 token，比 powerline 更轻量。

**vs `pi-powerline-footer`**：后者太重，会接管编辑器布局和鼠标滚动，改变了 Pi 原生简洁的 TUI 体验。

### `pi-markdown-preview` — Markdown / LaTeX 预览

### `@juicesharp/rpiv-ask-user-question` — 结构化提问 UI

---

## 其他实用插件

| 插件 | 用途 |
|------|------|
| `pi-image-gen` | Image2 图像生成/编辑，支持文生图、图生图 |
| `pi-fast-context` | vibe 出来的 fast-context 插件，上下文加速 |
| `@feniix/pi-sequential-thinking` | 链式思考 MCP 的 Pi 版本 |
| `pi-subagents`（最高下载版） | 预定角色 + 预定工作流 |

---

## 完整 Package 清单

| Package | 安装命令 | 用途 |
|---------|---------|------|
| `pi-subagents` | `pi install npm:pi-subagents` | 子代理 |
| `pi-mcp-adapter` | `pi install npm:pi-mcp-adapter` | MCP 桥接 |
| `pi-markdown-preview` | `pi install npm:pi-markdown-preview` | MD/LaTeX 预览 |
| `@juicesharp/rpiv-ask-user-question` | `pi install npm:@juicesharp/rpiv-ask-user-question` | 结构化提问 |
| `@victor-software-house/pi-curated-themes` | `pi install npm:@victor-software-house/pi-curated-themes` | 主题 |
| `pi-ace-tool` | `pi install git:github.com/justhil/pi-ace-tool` | 代码搜索增强 |
| `pi-rewind` | `pi install npm:pi-rewind` | git 回退 |
| `pi-image-gen` | `pi install git:github.com/justhil/pi-image-gen` | 图像生成 |
| `pi-search` | `pi install git:github.com/justhil/pi-search` | 搜索增强 |
| `pi-btw` | `pi install npm:pi-btw` | 并行旁路思考 |
| `pi-simplify` | `pi install npm:pi-simplify` | 代码审查 |
| `pi-dynamic-context-pruning` | `pi install git:github.com/complexthings/pi-dynamic-context-pruning` | 上下文剪枝 |
| `@ff-labs/pi-fff` | `pi install npm:@ff-labs/pi-fff` | 极速模糊搜索 |
| `pi-until-done` | `pi install npm:pi-until-done` | 目标完成前不停止 |
| `pi-codex-goal` | `pi install npm:pi-codex-goal` | Codex 风格 goal |
| `pi-nano-context` | `pi install npm:pi-nano-context` | 上下文占用条 |
| `pi-tool-display` | `pi install npm:pi-tool-display` | 工具输出美化 |
| `@narumitw/pi-plan-mode` | `pi install npm:@narumitw/pi-plan-mode` | 只读规划模式 |
| `pi-observational-memory` | `pi install npm:pi-observational-memory` | 观察者记忆 |
| `gotgenes/pi-permission-system` | 手动 clone 到扩展目录 | 权限管理 |
| `pi-workspace-history` | `pi install npm:pi-workspace-history` | shadow git 回退 |
| `pi-wtf` | `pi install npm:pi-wtf` | 消息撤回 |

---

## 极简主义 vs Claude Code 的思考

| 维度 | Claude Code | Pi |
|------|------------|-----|
| 内置工具 | 丰富（50+） | 极简（4 个） |
| System Prompt | 较大 | < 1000 token |
| MCP | 内置 | 需要 Package 桥接 |
| Plan mode | 内置 | 需要 Package |
| 子 Agent | 内置 | 需要 Package |
| 上下文膨胀 | 容易 | 完全可控 |
| 可塑性 | 一般 | 极高 |

**什么时候选 Pi**：
- 你追求对上下文的**精确控制**
- 你想让模型只看到你想让它看到的内容
- 你喜欢「极简 harness + 按需扩展」的模式
- 你的工作流已经很成熟，只需要基础工具

**什么时候继续用 Claude Code**：
- 你需要开箱即用的丰富功能
- 你不介意为便利性付出上下文膨胀的代价
- 你喜欢把工具配置好之后就不用管了

---

## 写在最后

> 「Man! What can I say, Pi out!」

Pi 的意义在于它证明了**极简不等于弱**。一个 1000 token 系统提示、4 个内置工具的框架，靠插件生态可以做任何事情。

关键在于：**你真的在控制模型能看见什么，而不是被模型的上下文膨胀推着走。**

如果你也在折腾 Pi，欢迎来聊。

---

## 相关阅读

- [earendil-works/pi — GitHub](https://github.com/earendil-works/pi)
- [pi.dev — 官网 & Package 搜索](https://pi.dev)
- [badlogic/pi-skills — 官方 Skills 包](https://github.com/badlogic/pi-skills)
- [Aegis — Pi 的工程纪律护栏](https://github.com/GanyuanRan/Aegis)
