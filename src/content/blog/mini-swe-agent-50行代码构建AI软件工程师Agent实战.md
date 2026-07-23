---
title: 'Mini-SWE-agent：用 50 行代码构建 AI 软件工程师 Agent 实战'
description: 'Princeton & Stanford 团队教你从零开始构建一个只依赖 bash 的极简 AI Agent，SWE-bench 得分 74%，代码比你想的少得多。'
pubDate: '2026-07-23'
tags: ["AI Agent", "SWE-bench", "mini-swe-agent", "编程工具", "LLM 应用"]
categories: ["AI开发"]
---

> 你见过只靠一个 bash 命令就解决问题的 AI Agent 吗？Princeton 和 Stanford 的团队证明了：不需要花哨的工具系统，不需要复杂的消息管理，一个 50 行的 Python 循环就能在 SWE-bench 上拿到 74% 的分数。这篇文章带你从源码层面拆解这个极简哲学。

---

## 项目概览

**mini-swe-agent** 是由 Princeton 大学、Stanford 大学 SWE-bench 团队（John Yang、Carlos Jimenez、Ofir Press 等人）开发的极简 AI 软件工程师 Agent。

```yaml
⭐ 5977 | Python | MIT License
创建: 2024 | 团队: Princeton + Stanford
官方教程: https://minimal-agent.com/
仓库: https://github.com/SWE-agent/mini-swe-agent
```

核心理念一句话：**Agent 不需要任何工具，只需要 bash。**

它背后的论文是 [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering](https://arxiv.org/abs/2405.15793)（NeurIPS 2024），团队在 2025 年进一步做了减法——把 SWE-agent 的精妙架构剥离掉，只留下最核心的 agent loop。

---

## 为什么需要极简 Agent

聊 mini-swe-agent 之前，先说说为什么"少即是多"在 Agent 领域是个重要命题。

现在的 AI Agent 生态已经卷得很厉害了。OpenAI Codex、Claude Code、Devin 这些产品，背后都有复杂的工具系统、消息管理器、状态机。但你真的需要那么多东西吗？

mini 团队给出的答案是：**不需要**。随着 LLM 本身能力的提升，很多 Agent 框架的复杂工具接口已经变得多余了。

mini-swe-agent 只保留三件事：

```
┌─────────────────────────────────────────────────┐
│              mini-swe-agent 哲学                │
├─────────────────────────────────────────────────┤
│ 1. 只依赖 bash，没有任何自定义工具              │
│ 2. 完全线性的对话历史，每一步追加到 messages     │
│ 3. 每次行动用 subprocess.run 独立执行           │
└─────────────────────────────────────────────────┘
```

这种极简设计带来了三个直接好处：

- **可调试性**：整个轨迹就是一份 JSON，等于 LM 看到的上下文 + 历史执行记录
- **可移植性**：换执行环境只需要把 `subprocess.run` 换成 `docker exec`，其他代码一字不用改
- **可扩展性**：没有状态管理，所以可以天然并行跑大量任务

---

## 50 行代码：从零构建 AI Agent

官方教程 [minimal-agent.com](https://minimal-agent.com/) 的核心就是一个 50 行 Python 脚本。完整代码如下：

```python
import re
import subprocess
import os
from litellm import completion

def query_lm(messages: list[dict[str, str]]) -> str:
    """调用 LLM API"""
    response = completion(
        model="openai/gpt-5.1",  # 可以换成任何 litellm 支持的模型
        messages=messages,
    )
    return response.choices[0].message.content

def parse_action(lm_output: str) -> str:
    """从 LLM 输出中提取 bash 命令"""
    matches = re.findall(
        r"```bash-action\s*\n(.*?)\n```",
        lm_output,
        re.DOTALL,
    )
    return matches[0].strip() if matches else ""

def execute_action(command: str) -> str:
    """执行 bash 命令"""
    result = subprocess.run(
        command,
        shell=True,
        text=True,
        env=os.environ,
        encoding="utf-8",
        errors="replace",
        stdout=subprocess.PIPE,
        stderr=subprocess.STDOUT,
        timeout=30,
    )
    return result.stdout

# ====== 主循环 ======
messages = [
    {
        "role": "system",
        "content": "You are a helpful assistant. When you want to run a command, "
                   "wrap it in ```bash-action\n<command>\n```. To finish, run exit.",
    },
    {
        "role": "user",
        "content": "List the files in the current directory",
    },
]

while True:
    lm_output = query_lm(messages)
    print("LM output:", lm_output)
    messages.append({"role": "assistant", "content": lm_output})
    # 记住 LLM 说了什么

    action = parse_action(lm_output)
    print("Action:", action)
    if action == "exit":
        break

    output = execute_action(action)
    print("Output:", output)
    messages.append({"role": "user", "content": output})
    # 把命令输出发回给 LLM
```

跑一下效果：

```bash
python minimal_agent.py
```

输出：

```
LM output: I'll list the files in the current directory.
```bash-action
ls -la
```
Action: ls -la
Output: total 48
drwxr-xr-x  5 user  staff   160 Jul 23 14:00 .
drwxr-xr-x  3 user  staff    96 Jul 23 13:55 ..
-rw-r--r--  1 user  staff   420 Jul 23 13:50 minimal_agent.py
Action: exit
```

---

## 核心组件深度解析

### 1. query_lm — 统一的 LLM 调用层

教程里用的是 litellm，因为它可以兼容 100+ 模型：

```python
from litellm import completion

completion(model="openai/gpt-5.1", messages=messages)
completion(model="anthropic/claude-sonnet-4-5", messages=messages)
completion(model="deepseek/deepseek-chat", messages=messages)
```

但 mini-swe-agent v2 的正式实现更优雅——它用 litellm 的 `tools` 参数直接支持 **Tool Calling**：

```python
# litellm_model.py 源码
response = litellm.completion(
    model=self.config.model_name,
    messages=messages,
    tools=[BASH_TOOL],  # 直接注册一个 bash 工具给 LLM
    **(self.config.model_kwargs | kwargs),
)
```

`BASH_TOOL` 是一个声明式的工具定义，LLM 直接通过 tool call 返回要执行的命令，省掉了正则解析这一步。但这只是糖衣炮弹——核心的 parse + execute 逻辑是一样的。

### 2. parse_action — 正则解析的艺术

这节是教程里最容易被忽略但实际最关键的部分。LLM 返回一段自然语言 + 一个代码块，怎么把命令准确地抠出来？

```python
def parse_action(lm_output: str) -> str:
    """Take LM output, return action"""
    matches = re.findall(
        r"```bash-action\s*\n(.*?)\n```",
        lm_output,
        re.DOTALL,
    )
    return matches[0].strip() if matches else ""
```

这段正则有三个关键点值得拆解：

```
r"```bash-action\s*\n(.*?)\n```"
    │         │    │  │
    │         │    │  └── 非贪婪：停在第一个闭合 ``` 而不是最后一个
    │         │    └──   换行符，确保 action 独占一行
    │         └──────── 允许开头有多余空白
    └────────────────── 原始字符串，避免 \\n 转义地狱
```

`re.DOTALL` 标志让 `.` 匹配换行符，这样多行命令也能被完整捕获：

```
LLM 输出示例：
I'll navigate to the project and check the structure.
```bash-action
cd /path/to/project
find . -name "*.py" | head -20
```
```

生产环境的 parse_action 更复杂，还有 fallback 逻辑：如果正则没匹配到，会把整段 LM 输出当作命令尝试执行（在安全沙箱环境下）。

> 💡 **踩坑提醒**：正则里 `.*?` 的非贪婪匹配是关键。如果写成 `.*`，当 LM 输出里包含多个代码块时，它会匹配到最后一个 ```` `，导致 action 包含不该包含的内容。

### 3. execute_action — subprocess.run 的精妙

```python
subprocess.run(
    command,
    shell=True,          # 允许管道、&& 等 shell 特性
    text=True,           # 返回字符串而非字节
    env=os.environ,      # 继承当前环境变量
    encoding="utf-8",    # 明确编码，避免乱码
    errors="replace",    # 无效字符替换而非抛异常
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,  # stderr 和 stdout 合并捕获
    timeout=30,          # 30 秒超时保护
)
```

这里有一个重要限制：**每次执行都在独立子 shell 中运行**。这意味着：

```bash
# ❌ 不持久
cd /tmp
ls    # 还是在原来的目录

# ❌ 不持久
export API_KEY=xxx
echo $API_KEY  # 空字符串

# ✅ 正确做法
cd /tmp && ls
# 或者
API_KEY=xxx some_command
```

mini 团队认为这其实**不是缺点，反而是优点**——它强制 Agent 使用绝对路径和自包含命令，减少了隐式状态，让对话历史更干净。

---

## 生产级增强：让 Agent 不再翻车

50 行玩具代码在实际使用时会遇到很多问题。mini-swe-agent 通过一套 **异常分层体系** 解决了这个问题。

### 异常分层架构

```
exceptions.py 定义了 6 个异常类：

┌─────────────────────────────────────┐
│  FormatError                        │ ← LM 输出格式不对
│    ├── (捕获后) 告诉 LM 重新输出   │
│    └── 连续 3 次 → 直接退出       │
├─────────────────────────────────────┤
│  InterruptAgentFlow                 │ ← 中断 Agent 流程
│    ├── Submitted                    │ ← 任务提交完成
│    ├── LimitsExceeded               │ ← 超过步数/成本限制
│    │     └── TimeExceeded           │ ← 超过墙钟时间限制
│    └── UserInterruption             │ ← 用户主动中断
└─────────────────────────────────────┘
```

### DefaultAgent 的核心循环

源码里 `DefaultAgent.run()` 的循环设计非常漂亮：

```python
def run(self, task: str = "", **kwargs) -> dict:
    """Run step() until agent is finished."""
    # 初始化：渲染 system + user 模板
    self.messages = []
    self.add_messages(
        self.model.format_message(
            role="system",
            content=self._render_template(self.config.system_template),
        ),
        self.model.format_message(
            role="user",
            content=self._render_template(self.config.instance_template),
        ),
    )

    while True:
        try:
            self.step()                        # query + execute
            self.n_consecutive_format_errors = 0  # 每次成功重置计数
        except FormatError as e:
            # 格式错误：加到消息里，让 LM 自己修正
            self.cost += e.messages[0].get("extra", {}).get("cost", 0.0)
            self.n_consecutive_format_errors += 1
            if 0 < self.config.max_consecutive_format_errors <= self.n_consecutive_format_errors:
                # 连续 3 次格式错误 → 直接退出
                self.add_messages(*e.messages, {
                    "role": "exit",
                    "content": "RepeatedFormatError",
                })
            else:
                self.add_messages(*e.messages)
        except InterruptAgentFlow as e:
            # 正常中断（完成/超限/用户中断）
            self.add_messages(*e.messages)
        except Exception as e:
            # 未知异常 → 记录后抛出
            self.handle_uncaught_exception(e)
            raise
        finally:
            self.save(self.config.output_path)  # 每次循环都存一次

        if self.messages[-1].get("role") == "exit":
            break

    return self.messages[-1].get("extra", {})
```

这个设计的精髓在于：**异常不是 bug，而是 Agent 的感知输入**。当 subprocess 超时、格式不对、成本超标，这些信息都作为 user 消息追加到对话中，LLM 可以自主决策下一步该怎么做。

### format_error_template — 教会 LM 正确输出

default.yaml 里有一个精心设计的格式错误提示模板：

```yaml
format_error_template: |
  Format error:
  <error>{{error}}</error>
  Please always provide EXACTLY ONE action in triple backticks,
  found {{actions|length}} actions.
  If you want to end the task, issue:
    `echo COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT`
  Else, format your response as follows:
  <response_example>
  Here are some thoughts about why you want to perform the action.
  ```mswea_bash_command
  <action>
  ```
  </response_example>
```

当 LM 输出包含多个代码块、或者输出被截断（`finish_reason="length"`）时，这个模板会告诉 LM：
- 你给了 N 个 action，我只能执行 1 个
- 正确的格式是什么
- 如果是要结束任务，请执行特定的 submit 命令

> 💡 **踩坑提醒**：弱模型（比如小参数量或开源模型）经常输出格式不规范。这时候 format_error_template 的质量直接影响 agent 的鲁棒性。模板里最好包含"正确示例"和"错误说明"。

### 环境变量的静默配置

mini 通过环境变量静默禁用所有交互式工具，避免 Agent 卡死：

```yaml
environment:
  env:
    PAGER: cat          # 禁用 less/more 分页
    MANPAGER: cat       # 禁用 man 手册交互
    LESS: -R            # 允许 raw 控制字符
    PIP_PROGRESS_BAR: 'off'  # 禁用 pip 进度条
    TQDM_DISABLE: '1'   # 禁用 tqdm 进度条
```

---

## 使用场景一：本地开发助手

```bash
# 安装
pip install mini-swe-agent

# 运行
mini -t "帮我重构 main.py 中的数据库连接逻辑，改为连接池" -m anthropic/claude-sonnet-4-5
```

```text
mini 工作流程：
1. 读取项目文件（ls, cat, grep）
2. 分析现有数据库代码
3. 修改 main.py，引入连接池
4. 运行脚本验证
5. 输出 echo COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT
```

**关键点**：
- 不需要 Docker，直接在本地跑
- 所有操作都是 shell 命令，日志可追溯
- 适合日常开发中的代码修改、bug 修复

---

## 使用场景二：SWE-bench 批量评测

mini 的核心定位是 **SWE-bench 评测基线**。在 2026 年 7 月，它已经在 DeepSWE 基准上超越了 Claude Code 和 Codex。

```bash
# 批量运行 SWE-bench 任务
mini-swe-agent \
  --env_type docker \
  --model "anthropic/claude-sonnet-4-5" \
  --swe_bench_tasks_file tasks.jsonl
```

mini 的设计使其天然适合大规模并行：

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Instance │  │ Instance │  │ Instance │
│  #1      │  │  #2      │  │  #3      │
│  docker  │  │  docker  │  │  docker  │
│  claude  │  │  gpt-5.1 │  │  deepseek│
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     └─────────────┼─────────────┘
                   │
        完全独立，无共享状态
```

**关键点**：
- 每个实例在一个 Docker 容器里独立运行
- 没有共享内存、没有共享进程，所以可以无限并行
- 成本上限（cost_limit）和步数上限（step_limit）防止单次运行失控

---

## 使用场景三：自定义 Agent 脚手架

如果你想在自己的项目里嵌入一个类似 mini 的 agent，可以参考这个模式：

```python
# 伪代码：在你的系统里实现一个 mini-style agent
class MyAgent:
    def __init__(self, model_name, tool_executor):
        self.model = LiteLLMModel(model_name=model_name)
        self.exec = tool_executor  # 可以是本地、Docker、远程服务器

    def run(self, task):
        messages = [
            {"role": "system", "content": self.SYSTEM_PROMPT},
            {"role": "user", "content": task},
        ]
        while True:
            response = self.model.query(messages)
            action = self.parse_action(response)
            if action == "exit":
                break
            result = self.exec.run(action)  # 你的自定义执行器
            messages.extend([
                {"role": "assistant", "content": response},
                {"role": "user", "content": result},
            ])
        return result
```

**关键点**：
- 核心 loop 只有 20 行
- 换执行器只需要改 `tool_executor`
- 可以接你的任意模型（OpenAI、Anthropic、DeepSeek、本地模型）

---

## 适用场景对比

| 场景 | mini-swe-agent | Claude Code | Codex CLI | SWE-agent v1 |
|------|:---:|:---:|:---:|:---:|
| **本地快速开发** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **SWE-bench 评测** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **自定义工具扩展** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **多模态（截图/图）** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **沙箱化部署** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **调试和可追溯** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐ |
| **学习 Agent 原理** | ⭐⭐⭐ | ⭐ | ⭐ | ⭐⭐ |

mini 的优势场景：**想要一个可理解、可调试、可大规模部署的 Agent 基线**。

不适合的场景：**需要复杂工具链（浏览器操作、数据库查询、文件编辑器）的场景**——mini 只有 bash，复杂的交互需要你自己封装。

---

## 总结

mini-swe-agent 最打动我的不是它的代码有多短，而是它的**设计哲学**：

> 当一个模型足够强大的时候，Agent 框架应该退到幕后，让模型自己决定做什么、怎么做。

50 行代码能做到的事情：

- ✅ 解决 GitHub issue（SWE-bench 74%）
- ✅ 支持 100+ LLM 模型
- ✅ 支持本地/Docker/Singularity 多种执行环境
- ✅ 异常自动恢复（格式错误、超时、超限）
- ✅ 完整的成本追踪和轨迹记录

但它也有边界：

- ❌ 没有文件编辑器工具（需要手动用 `cat > file` 或 `sed`）
- ❌ 没有浏览器操作
- ❌ 没有数据库连接器

如果你刚开始接触 AI Agent，mini-swe-agent 是最值得从源码学起的项目——因为它把 Agent 的**本质**暴露得干干净净：一个 while 循环，一次 query，一个 execute。

---

## 相关阅读

- [CodeGraph + Graphify + AgentMemory：AI Agent 工具栈深度解析](/blog/CodeGraph-Graphify-AgentMemory-AI-Agent工具栈/)
- [GoldAgent：量化 LLM 混合驱动的黄金价格分析系统](/blog/GoldAgent-量化LLM混合驱动的黄金价格分析系统/)
- [Aegis：为 AI 编程 Agent 装上工程纪律护栏](/blog/aegis-ai编程agent工程纪律护栏/)
- [Trellis：AI 编程 Agent 代码治理标准实战](/blog/trellis-ai编程agent代码治理标准实战/)

🔗 官方教程: https://minimal-agent.com/
🔗 GitHub 仓库: https://github.com/SWE-agent/mini-swe-agent
🔗 SWE-agent 论文: https://arxiv.org/abs/2405.15793
