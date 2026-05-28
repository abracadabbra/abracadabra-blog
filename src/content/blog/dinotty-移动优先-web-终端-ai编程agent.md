---
title: 'Dinotty：移动优先的 Web 终端，让 AI 编程 Agent 挣脱桌面束缚'
description: '深度解析 Dinotty 项目，服务端虚拟终端 + 会话持久化 + 移动端适配，让 Claude Code、Codex 在手机上也能流畅运行，排队、通勤、逛街时随时查看 agent 进度。'
pubDate: '2026-05-28 09:30:00'
tags: ["AI工具", "Claude-Code", "终端", "移动开发", "Rust"]
categories: ["AI工具"]
---

## 前言

用 Claude Code 写代码很爽，但必须守在电脑前。地铁里想看看 agent 跑怎么样了？得掏出笔记本。陪女朋友逛街时想检查一下任务进度？不好意思，没带电脑。

你可能会想：搞个远程桌面不就行了？VNC、RDP、Parsec 都行。但视频流方案在移动网络下卡成 PPT，流量还贼大。

**Dinotty** 换了个思路——它不传视频，只传**文本**。终端本质上就是字符网格，Dinotty 在服务端跑了一个完整的虚拟终端仿真器，只把变化的字符发给客户端，效率比远程桌面高 100~1000 倍。

## 项目概览

- **GitHub**: [xichan96/dinotty](https://github.com/xichan96/dinotty)
- **Stars**: 54 | **Forks**: 8
- **语言**: Vue (前端) + Rust (后端)
- **许可**: MIT
- **桌面端**: Tauri

## 核心技术亮点

### 1. 服务端虚拟终端（VT Screen）

这是 Dinotty 和其他 Web 终端本质的区别。

大多数 Web 终端（ttyd、gotty、Wetty）只是 WebSocket 到 PTY 的透传管道——它们把终端原始字节转发给浏览器，对屏幕上显示什么毫无感知。

Dinotty 在服务端跑了一个完整的 **VTE（Virtual Terminal Emulator）** 状态机（`vt_screen.rs`，600+ 行）。每个字符、每个转义序列、每次光标移动都在服务端被跟踪，形成一个结构化的字符网格。这意味着：

- 服务端始终掌握精确的屏幕状态（带颜色、粗体、斜体等属性）
- 滚动历史以环形缓冲区保留在服务端
- 可随时生成 ANSI 编码的屏幕快照

### 2. 会话持久化与自动重连

传统 Web 终端断网 = 会话丢失。Dinotty 的处理方式：

```
客户端断开 → PTY 进程继续在服务端运行
客户端重连 → 服务端回放滚动历史 → 发送当前屏幕快照 → 完美恢复
```

前端实现了**指数退避自动重连**（1s → 30s 上限）。甚至可以直接刷新浏览器页面恢复完整会话，不需要重启进程。

### 3. 轻量级：文本 vs 视频流

| | Dinotty | 远程桌面 (VNC/RDP/Parsec) |
|---|---|---|
| 传输数据 | 纯文本（JSON，字节流） | 全屏像素流，30-60 fps |
| 带宽消耗 | ~1–10 KB/s | ~1–10 MB/s（多 100–1000 倍） |
| 移动网络 | ✅ 3G/4G 下流畅无延迟 | ❌ 卡顿、高延迟、流量爆炸 |
| 弱信号容忍 | ✅ 自动重连，无画面丢失 | ❌ 画面冻结、输入延迟 |

### 4. 移动端深度适配

手机没有 Ctrl、Esc、功能键，Dinotty 的解法是**完全可自定义的快捷键盘**：

- 开箱即用：Ctrl+C、Ctrl+D、Escape、Tab 等终端常用操作
- 完全自定义：任意按键组合或转义序列（`\x03` = Ctrl+C，`\x1b[A` = 方向上键）
- 修饰键状态跟踪：粘滞 Ctrl/Alt/Shift，配合完整键盘布局使用

响应式布局也做了专门优化：竖屏时终端和预览面板上下排列，横屏时左右并排，和桌面 IDE 一样。

### 5. 内建文件工作区和网页预览

Coding Agent 会生成代码、文档和网页——验证产出通常需要切换到单独的工具。Dinotty 把这些都嵌进来了：

- **文件工作区**：树形文件浏览器，支持上传、重命名、删除
- **代码编辑器**：基于 Monaco Editor，支持语法高亮、自动补全
- **Git 变更指示**：编辑器 gutter 显示新增（绿）/修改（蓝）/删除（红）行标记，支持 inline diff、Stage 和 Revert
- **Markdown 预览**：实时渲染，带 HTML 净化
- **网页预览**：输入 URL 或本地端口号，在嵌入式 iframe 中浏览；内建反向代理路由到本地开发服务器
- **Office 文档**：直接预览 Word、Excel、PowerPoint
- **媒体播放**：内建音频/视频播放器

### 6. 通知系统 + Claude Code 集成

Dinotty 暴露了一个 HTTP API：

```bash
curl -s -X POST http://127.0.0.1:8999/api/notify \
  -H "Content-Type: application/json" \
  -d '{"body": "任务完成", "title": "Claude Code", "notification_type": "info"}'
```

配合 Claude Code 的 hooks，还可以在关键节点自动通知：

```jsonc
// .claude/settings.json
{
  "hooks": {
    "Notification": [{
      "matcher": "",
      "hooks": [{ "type": "command", "command": "curl -s -X POST http://127.0.0.1:8999/api/notify -H 'Content-Type: application/json' -d '{\"body\":\"Claude 需要你的输入\",\"title\":\"Claude Code\",\"notification_type\":\"warning\"}'" }]
    }],
    "Stop": [{
      "matcher": "",
      "hooks": [{ "type": "command", "command": "curl -s -X POST http://127.0.0.1:8999/api/notify -H 'Content-Type: application/json' -d '{\"body\":\"任务已完成\",\"title\":\"Claude Code\",\"notification_type\":\"info\"}'" }]
    }]
  }
}
```

### 7. 插件系统

Dinotty 支持安装插件扩展功能。插件在独立标签页中运行，使用 Vue 3 渲染，可调用终端、通知、持久化存储等内建 API：

| API 类别 | 能力 |
|---------|------|
| 终端 | `terminal.send()` 发送输入、`terminal.createTab()` 创建标签页 |
| 存储 | `storage.get/set` 持久化读写 |
| CLI | `exec.run()` 同步执行二进制、`exec.spawn()` 流式执行 |
| UI | `ui.notify()` 通知、`ui.confirm()` 确认对话框 |

内置插件包括 **CC Switch**（多 Claude Code Provider 一键切换）和 **JSON Formatter**。

## 技术栈

| 层级 | 技术 |
|------|------|
| 后端 | Rust, Axum 0.7, Tokio, portable-pty, vte |
| 前端 | Vue 3, TypeScript, Vite, xterm.js 5 |
| 桌面端 | Tauri |

## 安装与使用

### 前置条件

- Rust 工具链（stable）
- Node.js + pnpm

### 构建与运行

```bash
# 构建前端
cd frontend && pnpm install && pnpm run build && cd ..

# 运行服务器
cargo run
```

浏览器打开 http://127.0.0.1:8999

### 开发调试

```bash
# 带调试日志运行后端
RUST_LOG=debug cargo run

# 前端类型检查
cd frontend && npx vue-tsc --noEmit
```

## 功能对比

| 能力 | Dinotty | ttyd | gotty | Wetty |
|------|---------|------|------|------|
| 服务端虚拟终端 | ✅ 完整 VTE | ❌ | ❌ | ❌ |
| 断网后会话存活 | ✅ | ❌ | ❌ | ❌ |
| 刷新 = 恢复会话 | ✅ | ❌ | ❌ | ❌ |
| 内建文件浏览器和预览 | ✅ | ❌ | ❌ | ❌ |
| Git 变更指示 | ✅ | ❌ | ❌ | ❌ |
| 网页预览（反向代理） | ✅ | ❌ | ❌ | ❌ |
| 可自定义快捷键盘 | ✅ | ❌ | ❌ | ❌ |
| 系统监控 | ✅ | ❌ | ❌ | ❌ |
| 插件系统 | ✅ | ❌ | ❌ | ❌ |
| Token 认证 | ✅ | ✅ | ❌ | ✅ |

## 适用场景

| 场景 | 推荐指数 | 说明 |
|------|---------|------|
| 通勤时查看 Claude Code 进度 | ⭐⭐⭐⭐⭐ | 移动网络下流畅，断网不怕 |
| 排队时让 agent 继续干活 | ⭐⭐⭐⭐⭐ | 掏出手机就能续上 |
| 移动端做代码 Code Review | ⭐⭐⭐⭐ | Monaco Editor + Git diff 够用 |
| 纯桌面场景（公司/家里） | ⌨️ 桌面工具更好 | 直接用 iTerm2 + tmux 更顺手 |
| 弱网络环境（地铁/地下室） | ⭐⭐⭐⭐⭐ | 文本传输 vs 视频流，优势巨大 |
| 需要文件管理和预览一体化 | ⭐⭐⭐⭐ | 不用切工具，但功能不如 IDE |

## 总结

Dinotty 解决的是一个很具体的问题：**AI 编程 Agent 被束缚在桌面**。

它的核心创新是**服务端虚拟终端**——不只是透传字节，而是在服务端维护一个完整的屏幕状态。这使得会话持久化、屏幕快照、精确恢复成为可能，而这些是传统 Web 终端配合 tmux/screen 才能勉强达到的能力。

加上移动端深度适配、内建文件/网页预览、通知系统和插件生态，Dinotty 真正做到了**一个浏览器标签页解决所有问题**。

如果你经常需要在移动端监督 Claude Code / Codex 的工作，或者网络环境不太稳定，Dinotty 值得一试。
