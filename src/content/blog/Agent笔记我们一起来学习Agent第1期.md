---
title: '【Agent笔记】我们一起来学习Agent！（第1期）'
description: '以Agno框架为例，带你入门AI Agent开发。包含Agent核心概念、工具调用、缓存机制等基础知识。'
pubDate: '2026-03-31'
tags: ['AI', 'Agent', 'Agno', 'Python', '学习笔记']
categories: ['AI开发']
---

> 静心学习

**让我们以Agno框架为例，让我们一起来学习Agent吧！这是我的第一篇笔记。**

[Agno文档](https://docs.agno.com/introduction)

最近在做一个项目，要用到轻度的agent，于是我打算学习一些关于agent的知识，了解架构，可以大概看懂代码，以便于更好地用AI开发，**也拓展自己思维和对于coding的理解**，但完全不作为任何专业的培训哈！现在打算每天或者每周写一篇文章，然后作为学习笔记分享给大家。

我会以尽量易懂的方式向大家介绍和分享交流，大家也可以指出我的错误，需要一点Python基础（了解什么是类，什么是方法之类的），其它基本不需要，适合小白，因为我是小白。

---

## 1. Agent以及快速实现

> Agent is a stateful model with loops and tools.

Agent并不神秘，普通的模型只能通过api对话，输入文本输出文本，不能与外部世界获取联系，不能记忆，这就是无状态，并且只能一问一答，这就是没有loop循环。

Agent是如何实现状态的？如何调用工具的？如何决策，自动循环执行任务的？我们会一步一步来了解。

### 1.1 了解Agent类

这是agno的一个快速实现的Agent，我们来看代码：

```python
from agno.agent import Agent
from agno.db.sqlite import SqliteDb
from agno.models.anthropic import Claude
from agno.os import AgentOS
from agno.tools.mcp import MCPTools

agno_assist = Agent(
    name="Agno Assist",
    model=Claude(id="claude-sonnet-4-5"),
    db=SqliteDb(db_file="agno.db"),                     # session storage
    tools=[MCPTools(url="https://docs.agno.com/mcp")],  # Agno docs via MCP
    add_datetime_to_context=True,
    add_history_to_context=True,                         # include past runs
    num_history_runs=3,                                  # last 3 conversations
    markdown=True,
)

# Serve via AgentOS → streaming, auth, session isolation, API endpoints
agent_os = AgentOS(agents=[agno_assist], tracing=True)
app = agent_os.get_app()
```

- `Agent`是一个类，实例化的时候传入了`name`、`model`、`tools`等等参数，这就构建了一个拥有名字和工具的Agent机器人了
- `agent_os = AgentOS(agents=[agno_assist], tracing=True)`实例化了系统类，这个机器人正式部署可以开始用了。OS是系统环境，你可能有十个Agent，但他们都在这个环境里面跑
- `app = agent_os.get_app()`这是OS类的方法，把我们的Agent系统包装成一个API服务（FastAPI），这样你可以启动这个服务了

---

### 1.2 工具：Agent的核心

我每天有一个填Excel表的任务，很简单，我的老板每天给我一个Excel表格，要求是把里面的每行进行求和，然后排序。

我很聪明，那么笨的求和排序任务我写一个Python脚本来解决：

```python
import pandas as pd

file_name = 'daily_task.xlsx'  
df = pd.read_excel(file_name)
df['总和'] = df.sum(axis=1, numeric_only=True)
df_sorted = df.sort_values(by='总和', ascending=False)
df_sorted.to_excel('result.xlsx', index=False)

print("任务完成！结果已保存在 result.xlsx")
```

我的老板又给我派了其它任务，比如把数据全部变成直方图，我依然写代码解决：

```python
import pandas as pd
import matplotlib.pyplot as plt

plt.rcParams['font.sans-serif'] = ['SimHei'] 
plt.rcParams['axes.unicode_minus'] = False
df = pd.read_excel('result.xlsx')

plt.figure(figsize=(10, 6))
plt.hist(df['总和'], bins=10, color='skyblue', edgecolor='black', alpha=0.7)

plt.title('数据总和分布情况直方图', fontsize=15)
plt.xlabel('数值区间', fontsize=12)
plt.ylabel('频率', fontsize=12)
plt.grid(axis='y', linestyle='--', alpha=0.7)

plt.savefig('data_distribution.png')
plt.show()
```

一旦你为你的AI配好了工具，那么就可以让它来执行你可能需要手动做的任务，不同的是，它可以异步完成，甚至根据结果的反馈，进行组合调用，多次迭代，直到任务完成。

我们来简单的配置一个工具：访问HackerNews：

```python
from agno.agent import Agent
from agno.models.anthropic import Claude
from agno.tools.hackernews import HackerNewsTools

agent = Agent(
    model=Claude(id="claude-sonnet-4-5"),
    tools=[HackerNewsTools()],
    instructions="Write a report on the topic. Output only the report.",
    markdown=True,
)
agent.print_response("Trending startups and products.", stream=True)
```

学到这里，只要你学习一下如何写工具，那么基本上你就可以写一个初级Agent了！

---

### 1.3 可调用的工厂（Callable Factories）

如果所有工具都是如此简单的一个脚本，那么调用成本的确是很低的。但是，对于复杂的业务一个工具分为两个步骤：

- **加载**：相当于把工具拿出来放着（会运行初始化`_init_`）
- **执行**：就是我们的`py sth.py`

但是，当一个工具初始化时连接数据库，就会占用内存。成百上千个呢？内存就太大了。

所以Agno引入了动态加载的机制：

```python
from agno.agent import Agent
from agno.models.openai import OpenAIResponses
from agno.run import RunContext
from agno.tools.duckduckgo import DuckDuckGoTools
from agno.tools.yfinance import YFinanceTools


def get_tools(run_context: RunContext):
    role = (run_context.session_state or {}).get("role", "general")
    if role == "finance":
        return [YFinanceTools()]
    return [DuckDuckGoTools()]


agent = Agent(
    model=OpenAIResponses(id="gpt-5-mini"),
    tools=get_tools,
)

agent.print_response("AAPL stock price?", session_state={"role": "finance"}, stream=True)
agent.print_response("Latest AI news?", session_state={"role": "general"}, stream=True)
```

支持传入tools的时候传入一个函数，这个函数的作用是看你的身份，返回你的工具。比如同一个Agent系统，那么管财务的就用财务工具，不需要给你加载开发工具。这种动态加载就是**callable factory**。

---

### 1.4 缓存

缓存，caching的本质是一个内存字典。在Agno程序运行期间，后台会有一个字典：

- **key（键）**：比如"user_123_tools"
- **value（值）**：是一个已经初始化好的Python对象（比如数据库的连接实例）

工具初始化有时候是昂贵的：

```python
class MyDatebaseTool:
    def _init_(self):
        # 下一行会非常慢，要握手验证，占用宽带
        self.connection = connect_todatabase("198.168.1.1")
```

每次对AI说话，Agno都会加载所需要的工具，第一轮的时候我们可能已经加载过`MyDatebaseTool`了，如果没有缓存，这个实例用完就丢弃，第二次对话继续重建，造成了资源浪费。

如果有缓存机制，那么第一轮的对话中，Agno框架会生成一个key："user_123"，并且value绑定一个刚才用到的连接实例。

基于以上原理我们有这些Agent类的参数：

| Setting 设置 | Default | Description |
|:---|:---|:---|
| `cache_callables` | `True` | 启用或禁用所有可调用工厂的缓存 |
| `callable_tools_cache_key` | `None` | 工具工���的��定义缓存键函数 |
| `callable_knowledge_cache_key` | `None` | 知识工厂的自定义缓存键函数 |
| `callable_members_cache_key` | `None` | 成员工厂的自定义缓存键函数（仅限团队） |

---

### 1.5 运行Agent

运行Agent我们使用`Agent.run()`或者`Agent.arun()`：

```python
from agno.agent import Agent, RunOutput
from agno.models.openai import OpenAIChat
from agno.tools.duckduckgo import DuckDuckGoTools

# 初始化 Agent
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    tools=[DuckDuckGoTools()],
    markdown=True
)

# 传入问题，并带上用户和会话 ID
response: RunOutput = agent.run(
    input="搜一下今天 AI 圈有什么大新闻？", 
    user_id="user_001", 
    session_id="session_abc"
)

# 从结果对象中拿数据
print(f"--- AI 的回答 ---")
print(response.content)  # 拿最终的文字结果

print(f"\n--- 运行元数据 ---")
print(f"会话 ID: {response.session_id}")
print(f"运行 ID: {response.run_id}")

print(f"\n--- 消耗指标 ---")
print(f"总 Token 数: {response.metrics.get('tokens')}")
print(f"总耗时 (秒): {response.metrics.get('time')}")
```

其中，`input`参数可以是字符串，列表，消息，字典，pydantic模型或者消息列表。

Agent返回的是一个`RunOutput`对象，它包含：

| 字段 | 描述 |
|:---|:---|
| `run_id` | 运行的 ID |
| `agent_id` | 代理的 ID |
| `session_id` | 会话的 ID |
| `user_id` | 用户的 ID |
| `content` | 响应内容 |
| `content_type` | 内容类型 |
| `reasoning_content` | 推理内容 |
| `messages` | 发送给模型的消息列表 |
| `metrics` | 运行的指标 |
| `model` | 用于运行的模型 |

---

### 1.6 流式输出

```python
stream = agent.run("再搜搜 DeepSeek v4的消息", stream=True)
for chunk in stream:
    # 过滤掉后台工具调用的日志
    if chunk.event == "run_content":
        print(chunk.content, end="", flush=True)
```

---

## 2. 实现 - 基于Qwen-3.6-plus与OpenCode

我们就来实现一下一个简单Agent吧，用这个框架。

测试一下Qwen-3.6-plus在OpenCode中的表现，完成一个高效Excel记账助手。

目标：
1. 一个高效Excel记账助手，每天为在同一个Excel表格中记录账单
2. Agent可以调用的工具有：
   - `question`：询问我不清楚的地方
   - `write_excel`：一个写入Excel的脚本

经过测试，Qwen-3.6-plus写代码中规中矩，遇到API调用的bug会一直循环，需要人工指导修正。总之，拿来编程的话需要靠人的实力，很多小细节把控不好。

但总之这个Agent流程我们跑通了！

---

## 总结

以上是基本的运行Agent过程，接下来的内容我们过几天再学吧！

> 如果你有更多补充或发现信息有误，欢迎留言讨论！