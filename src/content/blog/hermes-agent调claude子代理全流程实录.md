---
title: '用 Hermes Agent 调 Claude 子代理，从零到部署的全流程实录'
description: '记录使用 Hermes Agent 编排 Claude Code 子代理，创建二次元博客封面页并部署到 Vercel 的完整过程'
pubDate: '2026-05-10 23:30:00'
tags: ["AI工具", "Hermes", "Claude", "Vercel", "前端"]
categories: ["AI工具"]
---

## 前言

今天干了一件有意思的事：用 Hermes Agent 调 Claude Code 子代理，从零开始创建了一个二次元风格的博客封面页，然后推到 GitHub，最后通过 Vercel API 自动部署上线。

整个过程踩了不少坑，但最终效果不错。这篇文章记录一下完整的流程。

## 环境准备

### Hermes Agent

Hermes Agent 已经跑在我的服务器上了，是一个 CLI AI 助手框架，可以：
- 执行终端命令、读写文件
- 派发子代理并行执行任务
- 连接微信、飞书等消息平台

### Claude Code

Claude Code 是 Anthropic 出的编码代理 CLI，能独立读写代码、执行命令。

安装方式：

```bash
npm install -g @anthropic-ai/claude-code
```

装完之后需要认证：

```bash
claude auth login --console   # API Key 方式
```

> ⚠️ 服务器是无头环境，不能用浏览器 OAuth，只能用 API Key 或 `--console` 模式。

确认安装成功：

```bash
claude --version
# 2.1.138 (Claude Code)
```

## 三种调用方式

在 Hermes 中调用 Claude，有三种方式，适合不同场景：

### 方式一：直接调用 Claude CLI

最简单粗暴——Hermes 通过终端命令直接调 Claude Code：

```python
terminal(command="claude -p '帮我写一个 React 组件' --max-turns 10")
```

**特点：**
- Claude 独立执行，读写文件、跑命令都行
- `-p` 是非交互模式，执行完自动退出
- 不经过 Hermes 子代理，没有额外开销
- 适合**单次、明确的编码任务**

### 方式二：ACP 协议子代理（真正的 Claude 子代理）

通过 ACP（Agent Communication Protocol）协议启动 Claude 子代理：

```python
delegate_task(
    goal="帮我写一个 REST API",
    acp_command="copilot",
    toolsets=["terminal", "file"]
)
```

**特点：**
- 真正的 Claude 子代理，用 Claude 的模型能力
- 独立终端会话，支持并行执行多个任务
- 适合**复杂的、需要多步推理的编码任务**

### 方式三：Hermes 原生子代理

Hermes 自己的子代理系统，底层模型取决于你的配置：

```python
delegate_task(
    goal="帮我写一个二次元风格的博客封面页",
    toolsets=["terminal", "file"]
)
```

**特点：**
- 不依赖 Claude Code，用你自己的 API
- 成本最低
- 适合**文件操作、简单编码、信息整理**

### 对比

| | 直接调 CLI | ACP 子代理 | Hermes 原生子代理 |
|---|---|---|---|
| 底层模型 | Claude | Claude | 取决于配置 |
| Token 消耗 | Claude 额度 | Claude 额度 | 你自己的 API |
| 独立会话 | ❌ | ✅ | ✅ |
| 并行执行 | ❌ | ✅ | ✅ |
| 成本 | 中 | 高 | 低 |

## 实际操作：创建二次元封面页

我选择了**方式三**（Hermes 原生子代理），因为只是做一个静态页面，不需要 Claude 的高级推理能力。

### 第一步：派发子代理

```python
delegate_task(
    goal="在 /root/anime-blog-cover/ 创建一个二次元风格的动态博客封面页，包含樱花飘落动画、星光闪烁、霓虹渐变色彩，响应式布局，移动端适配",
    toolsets=["terminal", "file"]
)
```

子代理执行了大约 **400 秒**。

### 第二步：踩坑——子代理只完成了一半

结果出来一看，只生成了 `index.html`，CSS 和 JS 文件完全没有。

这说明一个问题：**子代理的模型能力决定了产出质量**。用非 Claude 的模型（我配置的是 mimo-v2.5-pro），在复杂的前端动画任务上还是力不从心。

### 第三步：人工补全

没办法，只能自己动手。我手动写了：

- `style.css`（10.9KB）— 樱花 Canvas 动画、星光粒子、霓虹渐变、毛玻璃导航栏
- `script.js`（6.3KB）— 樱花飘落逻辑、打字机标题效果、滚动动画

最终效果：
- 🌸 樱花花瓣从屏幕上方飘落
- ✨ 四角星闪烁，粉紫蓝白随机
- 🏮 浮动灯笼缓缓上升
- ☁️ 云朵从右向左漂移
- 📝 标题「梦境笔记」打字机效果
- 📱 响应式布局 + 汉堡菜单

### 第四步：推送到 GitHub

```bash
cd /root/anime-blog-cover
git init
git add .
git commit -m "feat: 二次元风格动态博客封面"
git remote add origin https://github.com/abracadabbra/anime-blog-cover.git
git push -u origin main
```

仓库地址：https://github.com/abracadabra/anime-blog-cover

## Vercel 部署

这一步有点波折。

### 创建 Vercel 项目

通过 Vercel API 创建项目：

```bash
curl -s -X POST https://api.vercel.com/v10/projects \
  -H "Authorization: Bearer <VERCEL_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "anime-blog-cover",
    "framework": null
  }'
```

### 第一次失败：Git Source 错误

关联 GitHub 仓库时报错 `incorrect_git_source_info`——原因是 Vercel 的 GitHub App 还没有安装。

### 解决：安装 Vercel GitHub App

需要先去这个地址授权：
```
https://github.com/apps/vercel/installations/new
```

授权完成后，重新关联仓库：

```bash
curl -s -X PATCH https://api.vercel.com/v10/projects/<project-id> \
  -H "Authorization: Bearer <VERCEL_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "link": {
      "type": "github",
      "repo": "abracadabbra/anime-blog-cover",
      "productionBranch": "main"
    }
  }'
```

### 触发部署

```bash
curl -s -X POST https://api.vercel.com/v13/deployments \
  -H "Authorization: Bearer <VERCEL_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "anime-blog-cover",
    "projectId": "<project-id>",
    "target": "production",
    "gitSource": {
      "type": "github",
      "ref": "main",
      "repoId": "<repo-id>"
    }
  }'
```

部署成功！

**线上地址：https://anime-blog-cover.vercel.app**

以后每次 push 到 `main` 分支，Vercel 都会自动重新部署。

## 踩坑总结

| 坑 | 原因 | 解决 |
|---|---|---|
| 子代理只生成了 HTML | 模型能力不足（非 Claude） | 人工补全 CSS/JS |
| Claude Code 安装后找不到命令 | npm 全局路径不在 PATH | 手动 `export PATH` |
| Vercel 部署报 `incorrect_git_source_info` | GitHub App 未安装 | 去 GitHub 授权页面安装 |
| 服务器磁盘空间不足 | Flutter/Docker 占用太多 | 清理释放 2.9GB |

## 经验与思考

1. **子代理不是万能的**——模型能力直接决定产出质量。简单任务用便宜模型够了，复杂的前端动画还是得上 Claude。

2. **人工介入不可少**——AI 能搭框架，但细节打磨（动画参数、色彩搭配、响应式适配）目前还得人来调。

3. **Vercel + GitHub 的工作流很爽**——推代码自动部署，零配置，适合个人项目。

4. **Hermes 的编排能力很强**——不管是调 Claude CLI、启动子代理、操作 Git、调 Vercel API，都在一个对话里完成，不用切来切去。

## 最终效果

封面页包含：
- 樱花飘落 Canvas 动画
- 星光四角星粒子效果
- 浮动霓虹灯笼
- 打字机标题「梦境笔记」
- 6 篇模拟博文卡片
- 粉紫蓝渐变霓虹主题
- 响应式布局，移动端友好

访问看看：https://anime-blog-cover.vercel.app

---

如果你也想试试类似的流程，核心步骤就是：
1. 装好 Hermes Agent 和 Claude Code
2. 用 `delegate_task` 或 `claude -p` 让 AI 写代码
3. 推到 GitHub
4. 关联 Vercel 自动部署

有问题欢迎交流！
