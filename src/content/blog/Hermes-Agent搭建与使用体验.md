---
title: 'Hermes Agent 搭建与使用体验'
description: '从零开始搭建 Hermes AI 助手，接入微信和飞书，实现自动新闻搜集'
pubDate: '2026-05-10 19:00:00'
tags: ["AI工具", "Hermes", "自动化"]
categories: ["AI工具"]
---

## 什么是 Hermes Agent

Hermes Agent 是 Nous Research 推出的 CLI AI 助手，可以：
- 在终端中与 AI 对话
- 执行系统命令、读写文件
- 连接多种插件（Telegram、微信、飞书等）
- 设置定时任务自动执行

## 安装过程

### 1. 安装 Hermes

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

安装完成后，配置 API Key 和模型：

```bash
hermes config set provider.custom.api_key <your-api-key>
hermes config set model qwen/qwen3-coder
```

### 2. 环境准备

服务器上原有 Flutter SDK（约 1.6GB），已经不需要了：

```bash
# 删除 Flutter SDK
rm -rf /root/flutter

# 清理环境变量
sed -i '/flutter/d' /root/.bashrc
```

释放了约 **1.6GB** 空间。

## 使用体验

### 1. 基础对话

直接在终端输入 `hermes` 即可开始对话：

```bash
$ hermes
> 你好，能帮我做什么？
```

Hermes 可以：
- 执行 Shell 命令
- 读写文件
- 搜索网页
- 调用各种工具

### 2. 查看系统信息

让 Hermes 检查服务器状态：

```bash
> 服务器能跑多大的AI模型？

# Hermes 会自动执行：
# - free -h 查看内存
# - df -h 查看磁盘
# - lscpu 查看 CPU
```

### 3. 查看聊天记录

```bash
# 查看历史会话
hermes sessions list

# 查看特定会话
hermes sessions view <session-id>
```

## 接入微信和飞书

Hermes 支持接入微信和飞书，实现消息自动回复。

### 安装插件

```bash
hermes plugins install weixin feishu
```

### 配置登录

```bash
hermes weixin login
hermes feishu login
# 会弹出二维码，用微信/飞书 APP 扫码登录
```

### 配置自动回复

```bash
# 设置 AI 自动回复
hermes config set weixin.auto_reply true
hermes config set feishu.auto_reply true

# 设置监听的群聊
hermes config set weixin.listen_groups '["群聊名称"]'
hermes config set feishu.listen_groups '["群聊名称"]'
```

## 定时任务：自动搜集新闻

### 1. 创建新闻搜集脚本

让 Hermes 创建一个新闻搜集的定时任务：

```bash
> 帮我创建一个定时任务，每天早上 8 点搜集 AI 行业新闻，中午 12 点搜集新能源汽车行业新闻
```

Hermes 会自动创建两个 cron job：

```bash
# 查看定时任务
hermes cron list

# 输出示例：
# - AI行业新闻 (每天 08:00)
# - 新能源汽车/固态电池行业新闻 (每天 12:00)
```

### 2. 手动触发测试

```bash
# 立即执行一次新闻搜集
hermes cron run ai-news
hermes cron run ev-news
```

### 3. 查看搜集结果

新闻会自动发送到微信/飞书群聊，格式如下：

```
📰 AI 行业新闻 | 2026-05-10

1. 标题：xxx
   来源：xxx
   摘要：xxx

2. 标题：xxx
   来源：xxx
   摘要：xxx
```

## Git 集成：AI 辅助写代码

Hermes 可以直接操作 Git 仓库，帮你写代码、提交、推送。

### 1. 克隆项目

```bash
> 帮我克隆这个项目 https://github.com/abracadabbra/abracadabra-blog

# Hermes 会自动执行：
# git clone https://github.com/abracadabbra/abracadabra-blog.git
```

### 2. 分析项目结构

```bash
> 看看这个项目用了什么技术栈

# Hermes 会读取 package.json、AGENTS.md 等文件，然后告诉你：
# - 框架：Astro 5 + Vue 3
# - 样式：Tailwind CSS v4 + shadcn-vue
# - 包管理：pnpm
```

### 3. 让 AI 帮你写代码

```bash
> 帮我写一篇博客文章，内容是今天搭建 Hermes 的过程

# Hermes 会：
# 1. 查看现有文章的格式
# 2. 根据今天的对话生成内容
# 3. 写入到 src/content/blog/ 目录
```

### 4. 提交和推送

```bash
> 帮我把刚才写的文章提交并推送到 GitHub

# Hermes 会自动执行：
# git add .
# git commit -m "feat: 添加 Hermes 搭建体验文章"
# git push origin main
```

### 实际案例

今天我就用 Hermes 做了这些事：

1. **克隆博客项目**
   ```bash
   git clone https://github.com/abracadabbra/abracadabra-blog.git
   ```

2. **生成博客文章**
   - 自动分析现有文章格式
   - 根据对话历史生成内容
   - 保存到正确的目录

3. **配置 Git**
   ```bash
   git config --global user.name "shentao"
   git config --global user.email "38794455@qq.com"
   ```

这种方式特别适合：
- 快速生成文档、博客
- 批量修改代码
- 自动化 Git 操作

## 其他实用功能

### 1. 导出聊天记录

```bash
# 导出为 Markdown
hermes export --format markdown --output ./chat-history.md
```

### 2. 查看系统负载

```bash
> 系统负载如何了？

# Hermes 会自动执行 uptime、free -h 等命令并汇总
```

### 3. 磁盘清理

```bash
> 帮我看看磁盘占用，清理一下不需要的东西

# Hermes 会分析磁盘使用情况，建议清理方案
```

## 总结

Hermes Agent 是一个功能强大的 CLI AI 助手，特别适合：

- **服务器运维**：自动执行系统管理任务
- **信息搜集**：定时抓取新闻、监控数据
- **自动化工作流**：通过插件连接各种服务
- **开发辅助**：代码生成、调试、文档编写

相比传统的 AI 聊天工具，Hermes 的优势在于：
1. 可以直接操作服务器
2. 支持定时任务
3. 可以连接外部服务（微信、飞书、Telegram 等）
4. 完全在终端运行，适合开发者

如果你也想试试，官方文档：https://hermes-agent.nousresearch.com/docs
