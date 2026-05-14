---
title: 'CC-Viewer：给 Claude Code 套个 Web 壳，解锁移动端编程和完整日志'
description: '深度解析 CC-Viewer 项目，如何用 Web 界面增强 Claude Code 体验，支持移动端编程、请求报文拦截、多模型热切换'
pubDate: '2026-05-14 11:00:00'
tags: ["AI工具", "Claude", "Claude-Code", "编程效率", "逆向"]
categories: ["AI工具"]
---

## 前言

用 Claude Code 写代码，你有没有过这些疑问：

- Claude 到底读了哪些文件？它"看到"的内容和我以为的一样吗？
- 我的 System Prompt 长什么样？能不能拿来学学？
- 想在手机上改个 bug，非得开电脑？
- 切换到国产模型（DeepSeek、Kimi）能不能无缝切换？

**CC-Viewer** 就是为了解决这些问题而生的——它是 Claude Code 的一个增强外壳，核心能力是**完整拦截和展示 Claude Code 的所有 API 请求**。

## 项目概览

**CC-Viewer** 是一个基于 Claude Code 的 Vibe Coding 工具，作者从自身开发经验中提炼而来。

- **GitHub**: [weiesky/cc-viewer](https://github.com/weiesky/cc-viewer)
- **安装**: `npm install -g cc-viewer`
- **许可**: MIT

### 核心特性

| 特性 | 说明 |
|-----|------|
| 🖥️ Web 编程界面 | 浏览器里写代码，支持 diff 查看 |
| 📱 移动端编程 | 扫码在手机上编程 |
| 📝 完整日志拦截 | 记录 API 请求原文（未阉割） |
| 💬 对话模式 | 把日志解析成聊天界面 |
| 🔄 多模型热切换 | DeepSeek、GLM、Kimi 等 |
| 🔌 插件机制 | 自定义扩展 |

## 安装与使用

### 前提条件

- Node.js 20.0.0+
- 已安装 Claude Code

### 安装

```bash
# npm 安装
npm install -g cc-viewer

# 或 Homebrew（macOS/Linux 推荐）
brew tap weiesky/cc-viewer
brew install cc-viewer
```

### 启动

```bash
# ccv 是 claude 的直接替身，所有参数透传
ccv                    # == claude（交互模式）
ccv -c --d             # == claude --continue --dangerously-skip-permissions
```

启动后会自动打开 Web 页面。

### 日志模式

如果你习惯用原生 Claude Code 或 VS Code 插件，只想记录日志：

```bash
ccv -logger            # 启动日志模式
ccv --uninstall        # 卸载日志模式
```

日志保存在 `~/.claude/cc-viewer/项目名/日期.jsonl`。

## 场景一：看到 Claude 看到的东西

这是 CC-Viewer 最有价值的功能——**KV-Cache 可视化**。

### Claude Code 每次请求发送了什么？

你以为 Claude 只看了你让它改的那个文件？实际上它发送给 API 的内容包括：

```
┌─────────────────────────────────────────┐
│  发送给 Claude API 的内容                │
├─────────────────────────────────────────┤
│  1. System Prompt                       │
│     - AGENTS.md / CLAUDE.md             │
│     - 工具定义、能力说明                  │
│                                         │
│  2. 对话历史                             │
│     - 你说过的话                         │
│     - Claude 之前的回复                  │
│                                         │
│  3. 文件内容                             │
│     - Claude 主动读取的代码文件          │
│     - 目录结构、搜索结果                 │
│                                         │
│  4. 工具调用结果                         │
│     - terminal 输出、文件搜索结果        │
└─────────────────────────────────────────┘
```

### 实际例子

```
你以为：Claude 只看了 main.ts
实际发送：main.ts + utils.ts + AGENTS.md + package.json + 之前的对话历史
```

CC-Viewer 把这些"隐形"的上下文全部展示出来，让你知道 Claude 到底在"想"什么。

### 有什么用？

- **调试** — Claude 回答不对？看看它到底读了什么
- **优化** — 发送太多无效内容会浪费 token，可以精简
- **学习** — 了解 Claude 的上下文管理策略

## 场景二：完整请求报文，用于学习和逆向

CC-Viewer 记录的每条请求包含：

```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 16000,
  "system": [
    {
      "type": "text",
      "text": "You are Claude Code, Anthropic's official CLI...",
      "cache_control": { "type": "ephemeral" }
    }
  ],
  "messages": [
    {
      "role": "user",
      "content": [
        { "type": "text", "text": "帮我修复这个 bug..." },
        { "type": "text", "text": "以下是 /src/main.ts 的内容:\n```typescript\n...\n```" }
      ]
    },
    {
      "role": "assistant",
      "content": [
        { "type": "thinking", "thinking": "让我先看看代码结构..." },
        { "type": "tool_use", "name": "terminal", "input": { "command": "grep -r 'error'" } }
      ]
    }
  ]
}
```

### 记录了什么？

| 记录内容 | 说明 |
|---------|------|
| 完整 System Prompt | 没被阉割的系统提示词原文 |
| 每轮 Messages | 用户消息、助手回复、工具调用 |
| Thinking 过程 | Claude 的思考链 |
| Token 用量 | 输入/输出 token、缓存命中率 |
| Main/Sub Agent 区分 | 自动标记主 Agent 和子 Agent |

### 能学到什么？

**1. 学习 System Prompt 写法**

直接看到顶级 AI 产品的系统提示词：
- 怎么定义工具能力
- 怎么约束行为
- 怎么组织上下文

**2. 学习工具调用设计**

完整的工具 Schema 和调用链：
- terminal 工具的参数定义
- 文件操作的权限控制
- 多工具并行的编排方式

**3. 逆向产品策略**

从日志发现 Claude Code 的内部逻辑：
- 什么情况下触发 Sub Agent
- 代码搜索用了什么策略
- 长对话怎么压缩上下文

**4. 分析 Token 消耗**

```
[请求 #42] Token 明细
├─ 输入: 45,230 tokens
│  ├─ System Prompt: 8,120 (18%)
│  ├─ 文件内容: 22,450 (50%)
│  ├─ 对话历史: 12,340 (27%)
│  └─ 工具结果: 2,320 (5%)
├─ 输出: 1,200 tokens
└─ 缓存命中率: 72%
```

## 场景三：移动端编程

CC-Viewer 支持扫码在手机上编程：

1. 启动 `ccv` 后会显示二维码
2. 手机扫码打开 Web 界面
3. 局域网内实时同步

适合场景：
- 躺床上改个紧急 bug
- 外出时快速调整配置
- 演示给同事看

## 场景四：多模型热切换

CC-Viewer 内置 `cc-switch`，可以热切换三方模型：

| 模型 | 说明 |
|-----|------|
| DeepSeek-v4 | 国产性价比之王 |
| GLM 5.1 | 智谱清言 |
| Kimi K2.6 | 月之暗面 |

切换后无需重启，立即生效。

## 对话模式

把 API 日志解析成聊天界面：

- 用户消息右对齐（蓝色气泡）
- Claude 回复左对齐（深色气泡）
- Thinking 块默认折叠，可展开
- 支持双向定位：点击日志跳转对话，点击对话跳转日志

## 日志管理

- **压缩优化** — MainAgent 日志可降低 66% 体积
- **完整保留** — 不修改 Anthropic 官方定义
- **侧边栏搜索** — 快速定位特定 prompt

## 注意事项

CC-Viewer 与以下工具**冲突**（proxy 竞争）：
- claude-code-switch
- claude-code-router

使用前需要关闭这些工具，CC-Viewer 内置的热切换功能可以替代它们。

## 适用场景

| 场景 | 推荐度 |
|-----|-------|
| 学习 Claude Code 内部机制 | ⭐⭐⭐⭐⭐ |
| 移动端应急编程 | ⭐⭐⭐⭐ |
| 分析 Token 消耗、省钱 | ⭐⭐⭐⭐ |
| 切换国产模型 | ⭐⭐⭐ |
| 日常编程（已有 Claude Code） | ⭐⭐⭐ |

## 总结

CC-Viewer 的核心价值不是替代 Claude Code，而是**让你看到 Claude Code 看到的东西**。

通过完整拦截 API 请求，你可以：
- 学到顶级 AI 产品的 Prompt 设计
- 理解多 Agent 的编排策略
- 优化自己的 Token 使用
- 甚至逆向竞品的实现思路

如果你想深入理解 Claude Code 是怎么工作的，CC-Viewer 是目前最好的工具之一。

---

**相关阅读**：
- [AiGameAnent：把 AI Agent 做成像素风办公室模拟游戏](/blog/AiGameAnent-像素风AI-Agent办公室)
- [Oh My ClaudeCode Team 模式深度解析](/blog/Oh-My-ClaudeCode-Team模式深度解析)
- [Hermes Agent 调 Claude 子代理全流程实录](/blog/hermes-agent调claude子代理全流程实录)
