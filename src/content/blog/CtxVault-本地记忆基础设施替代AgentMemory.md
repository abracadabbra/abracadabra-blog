---
title: 'CtxVault：比 AgentMemory 轻量 10 倍的本地记忆基础设施'
description: '深入解析 CtxVault——用 typed vault 替代统一向量库，语义记忆和程序记忆分离，本地存储、MCP 支持、访问控制一个不少，安装只需一行 pip'
pubDate: '2026-05-30 10:00:00'
tags: ["AI工具", "CtxVault", "AgentMemory", "记忆系统", "本地优先", "MCP"]
categories: ["AI开发"]
---

## 前言

大多数 Agent 框架把"记忆"当附件——一个共享的向量库，什么都往里塞，事实、流程、偏好全混在一起。Agent 没法区分"我知道什么"和"我该怎么行动"。

CtxVault 的设计思路完全不同：**把记忆分层、分类、隔离**，让 Agent 需要什么就取什么，而不是每次把整块记忆糊给 LLM。

最关键的是——它很轻。

```bash
pip install ctxvault
```

就这一行，没有 npm、没有 server、不需要云端账号。

---

## 核心理念：Typed Memory

CtxVault 借鉴了认知科学里的经典分类——**语义记忆**（semantic memory）和**程序记忆**（procedural memory）。

| 记忆类型 | 存什么 | Agent 怎么用 |
|---------|--------|-------------|
| **语义记忆**（Semantic Vault） | 知识、文档、事实 | `query` —— 用自然语言查询 |
| **程序记忆**（Skill Vault） | 工作流程、行为规范 | `skill` —— 读取具体操作步骤 |

大多数框架只有第一种，CtxVault 把第二种也做成了基础设施。

```
┌─────────────────────────────────────────────────────────┐
│                    Agent                                 │
│                                                          │
│   "what should I do?"  ──►  Semantic Vault  ──►  知识库  │
│   "how should I do it?" ──► Skill Vault   ──►  操作手册 │
└─────────────────────────────────────────────────────────┘
```

---

## 与 AgentMemory 的直观对比

| | AgentMemory | CtxVault |
|---|------------|---------|
| **安装** | npm global + 跑 server + MCP 配置 | `pip install ctxvault` |
| **MCP 工具数** | 53 个 | 几个核心操作 |
| **存储** | 自己的格式 + server | 本地目录（纯文件） |
| **记忆类型** | 统一向量库 | **typed vault**（语义 + 程序分离）|
| **访问控制** | agent 名字过滤 | **结构层隔离**（vault 物理分离）|
| **本地化** | 需跑服务 | 纯本地，无需后台进程 |
| **复杂度** | 高（~950 测试）| 低（核心库很薄）|
| **Stars** | — | 57 |

AgentMemory 像一个功能完整的"记忆平台"，CtxVault 像一块可以嵌入任何流程的"记忆基础设施"。

---

## 核心特性逐个说

### 1. 语义 Vault：本地 RAG

Semantic Vault 就是本地的 RAG（检索增强生成）：

```bash
# 初始化一个 vault
ctxvault init my-vault

# 扔文档进去（PDF、MD、TXT、DOCX 都支持）
# 默认位置：~/.ctxvault/vaults/my-vault/

# 索引
ctxvault index my-vault

# 查询——语义搜索，不要求关键词匹配
ctxvault query my-vault "transformer 架构"
```

API 层面：

```python
import requests

# Agent 写内容到记忆
requests.post("http://127.0.0.1:8000/ctxvault/write", json={
    "vault_name": "my-vault",
    "filename": "session_monday.md",
    "content": "讨论了 Q2 预算：需要削减 15% 云成本。竞品价格比我们低 20%。"
})

# 几天后，用完全不同的词语查询
results = requests.post("http://127.0.0.1:8000/ctxvault/query", json={
    "vault_name": "my-vault",
    "query": "财务约束",  # ← 原始文档里没出现过这个词
    "top_k": 3
}).json()["results"]
```

这就是语义搜索的价值——"财务约束"能匹配到"削减 15% 云成本"。

---

### 2. Skill Vault：程序记忆

Skill Vault 存的是**行为规范和工作流**，不是事实：

```markdown
---
name: 每周工程周报
description: 如何为干系人撰写每周工程状态更新
---

你正在写每周工程周报……

## 必要结构
……

## 硬规则
- 不超过 250 词
- 不要以问候语开头
```

Agent 遇到写周报的需求时，查 skill vault 拿到具体步骤，而不是靠 prompt 一次性塞进去。

```bash
# 列出 skill vault 里的所有技能
ctxvault skills comms-skills

# 读取某个技能的具体内容
ctxvault skill comms-skills "每周工程周报"
```

---

### 3. 结构性隔离：Agent 之间的"防火墙"

这是 CtxVault 和其他方案最本质的区别。

大多数记忆系统的隔离靠"配置正确"——给数据打标签、设元数据。但配置会出错、会被绕过。

CtxVault 的隔离是**物理的**：

```bash
# 每个 vault 是独立目录，Agent 没有共享检索路径
ctxvault vaults

# 输出
Found 3 vaults

> agent-a-vault [RESTRICTED]
  path:    ~/.ctxvault/vaults/agent-a-vault
  agents:  agent-a

> shared-vault [PUBLIC]
  path:    ~/.ctxvault/vaults/shared-vault

> agent-c-vault [RESTRICTED]
  path:    ~/.ctxvault/vaults/agent-c-vault
  agents:  agent-c
```

一个 Agent 只能访问它被授权的 vault，边界是架构层保证的，不是配置文件里的规则。

---

### 4. MCP 集成：零代码接入

CtxVault 提供 MCP Server，任何 MCP 兼容的客户端直接用：

```bash
# 安装
uv tool install ctxvault
```

在 Claude Desktop 的 `mcp.json` 里加一行：

```json
{
  "mcpServers": {
    "ctxvault": {
      "command": "ctxvault-mcp"
    }
  }
}
```

Agent 立刻获得：`list_vaults`、`query`、`write`、`list_docs` 四个工具，全本地运行，不走云端。

---

### 5. 全程可观测

每个 vault 就是机器上的一个普通目录， humans 可以直接读写：

```bash
# 看 Agent 往记忆里写了什么
ctxvault list my-vault

# 人类直接往 vault 里扔文档
# 然后通知 Agent 重新索引
ctxvault index my-vault
```

人类和 Agent 共享同一个记忆层，没有黑箱。

---

## 适用场景

CtxVault 最适合这几类场景：

| 场景 | 为什么适合 CtxVault |
|------|-------------------|
| **个人研究助理** | 一行 pip 装完，本地跑 RAG |
| **多 Agent 隔离** | vault 物理隔离，无法跨库偷看 |
| **持续创作记忆** | 语义搜索，跨 session 记住上下文 |
| **技能库管理** | skill vault 存工作流，不是塞 prompt |

不太适合：需要多 Agent 共享同一份记忆（那是另一个场景，CtxVault 支持但不是它的强项）。

---

## 快速上手

```bash
# 安装
pip install ctxvault

# 初始化一个本地 vault
ctxvault init my-memory

# 添加文档并索引
ctxvault index my-memory

# 查询
ctxvault query my-memory "我的技术栈"

# 列出所有 vault
ctxvault vaults
```

API 模式（接 LangChain/LangGraph）：

```bash
uvicorn ctxvault.api.app:app
# 访问 http://127://0.0.1:8000/docs 交互文档
```

---

## 总结

CtxVault 的核心价值：**让记忆变成基础设施，而不是附件**。

- **typed memory** → 语义记忆和程序记忆分开，Agent 各取所需
- **结构性隔离** → vault 是物理目录，不是靠配置表
- **本地优先** → 纯文件存储，不依赖任何云服务
- **轻量** → 一行 pip，没有 server，没有 npm

如果 AgentMemory 是"记忆平台"，CtxVault 是"记忆 SDK"。对于写博客、做研究这个场景，SDK 反而比平台更实用。

> GitHub: [Filippo-Venturini/ctxvault](https://github.com/Filippo-Venturini/ctxvault)
> Stars: 57 | License: MIT | 语言: Python 3.10+
