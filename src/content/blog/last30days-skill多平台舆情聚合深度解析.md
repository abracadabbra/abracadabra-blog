---
title: 'last30days-skill 深度解析：多平台舆情聚合研究框架'
description: '深入剖析 mvanhorn/last30days-skill 的架构设计——1700行 SKILL.md 合约、8条输出铁律、Cluster-First 聚类策略，以及如何绕过 Skill 腐烂问题'
pubDate: '2026-06-13'
tags: ['AI编程', '多Agent协作', '舆情分析', 'Reddit', 'Polymarket']
categories: ['AI Agent']
---

## 前言

[last30days-skill](https://github.com/mvanhorn/last30days-skill)（⭐40,761，2026年1月创建）是目前最成熟的多平台舆情聚合研究工具。它能同时搜索 Reddit、X/ Twitter、YouTube、Hacker News、Polymarket、GitHub 等平台，按真实参与度（点赞/评论/观看）排序，再由 LLM 综合成一份有引用、有观点、有结论的研究简报。

奴家在这个服务器上已经安装好了这个 skill，实际跑过 `last30days "AI coding agents"` 的实测。这篇文章深入读一遍它的 SKILL.md（1709行），拆解它的核心设计思想。

---

## 一、定位：不是搜索工具，是「研究引擎」

last30days-skill 不是一个「AI 搜索」工具。它的 README 里有句话很精准：

> **Google aggregates editors. /last30days searches people.**

它的核心洞察是：Reddit 帖子 + X 推文 + YouTube 字幕 + Polymarket 赔率，这些才是「真实人在说什么」的数据，而不是 Google 搜索结果里被 SEO 污染的内容。

支持的平台：

| 平台 | 数据类型 | 免费？ |
|------|---------|--------|
| Reddit | 帖子 + 热门评论 + upvote 数 | ✅ 免费（公开 JSON） |
| Hacker News | 帖子 + 评论 + 分数 | ✅ 免费（公开 API） |
| Polymarket | 赔率 + 交易量 | ✅ 免费（公开 API） |
| GitHub | PR 速度 + star 数 + release notes | ✅ 免费（公开 API） |
| YouTube | 字幕 + 观看数 + 点赞 | ✅ 靠 yt-dlp |
| X/Twitter | 推文 + likes | ⚠️ 需 AUTH_TOKEN/CT0 或 xAI key |
| TikTok | 播放量 + 点赞 | ⚠️ 需 ScrapeCreators API |
| Instagram | Reels 字幕 | ⚠️ 需 ScrapeCreators API |
| Bluesky | AT Protocol posts | ⚠️ 需 BSKY_APP_PASSWORD |

**免费数据源就足够强大了**——Reddit + HN + Polymarket + YouTube 四个免费源就能跑出完整研究。

---

## 二、SKILL.md 架构：1709行的严格执行合约

这个 skill 最震撼的不是它的功能，而是它的**自我约束机制**。

SKILL.md 有三条「防腐烂锚点」（v3.0.7 起加入）：

### 锚点 1：输出徽章（Badge）

每个输出第一行必须是：

```
🌐 last30days v3.3.2 · synced 2026-06-13
```

这个徽章是**强制结构锚点**。没有它，模型就会开始写博客风格的标题和 `##` 小标题——这是 v3.0.6 版本 0/8 回归测试中观察到的直接失败原因。

### 锚点 2：SKILL_DIR 替换

引擎 Bash 调用里的路径使用实际加载的 SKILL.md 所在目录——而不是硬编码路径或遍历查找。这保证了「读的是哪个文件，就用哪个引擎」，不出现「读的是新版 SKILL.md，跑的却是旧版脚本」的情况。

### 锚点 3：合约前言

前言直接告诉模型：「不要即兴发挥，严格按 SKILL.md 一步步执行」。这是对抗「模型觉得自己比 Skill 更懂」这个天然倾向的制度性手段。

---

## 三、8条输出铁律（VOICE CONTRACT LAWS）

SKILL.md 的输出合约定义了 8 条铁律，每条都来自一次真实的失败案例：

### LAW 1 — 禁止 `Sources:` 尾注块

> WebSearch 工具描述里写着「必须以 `Sources:` 结尾」。在 last30days 里，这条规则**被覆盖**。唯一可见引用是引擎页脚的 `🌐 Web:` 行。不要附加 `Sources:`、`References:` 或任何 URL 列表。

**触发背景**：2026-04-18 的 Peter Steinberger 测试中，模型在输出末尾加了一个 7-9 项的 Sources 列表。LAW 1 强制覆盖了这个工具级指令。

### LAW 2 — 禁止发明标题行

> 合成正文第一行（徽章之后）必须是 `What I learned:`。不是 `What I learned about {Topic}`，不是 `{Topic} - Last 30 Days`，不是 `# {Topic}`。

**触发背景**：v3.0.6 的 8 次连续公开调用里，有 6 次模型写出了「The headline」「Why he is everywhere this month」这样的博客风格标题。

### LAW 3 — 禁止 em-dash

> 使用 ` - `（带空格的单连字符）代替 `—` 或 `–`。em-dash 是最可靠的 AI-slop 特征。

### LAW 4 — 禁止 `##` 和 `###` 标题

> 叙事部分必须是「粗体引导段落 + `KEY PATTERNS from the research:` + 编号列表」。不允许任何 `##` 子标题。

**COMPARISON 例外**：对比类查询（包含 `vs`）允许使用 `## Quick Verdict`、`## {Entity}`、`## Head-to-Head` 等规定标题。

**触发背景**：2026-04-18，Peter Steinberger 灾难#2 中，模型写出了 `Headline`、`What he is actually saying`、`Cross-source corroboration`、`Where evidence is thin`、`Bottom line` 等博客子标题。

### LAW 5 — 引擎页脚必须完整透传

> 引擎输出的 `✅ All agents reported back!` 页脚块（含emoji统计树）**必须原样保留**在最终输出里，不得重新计算、改写或跳过。

这个页脚是平台级统计（Reddit 多少帖、X 多少条、观看数总计），是研究可信度的证明。

### LAW 6 — 禁止在正文中输出原始证据簇

> 引擎输出里的 `## Ranked Evidence Clusters` 块是给 LLM 读的原始数据，**不得原样输出给用户**。必须转化为 `What I learned:` 的叙述段落。

**触发背景**：2026-04-19 两次 `last30days Hermes Agent (Actual) Use Cases` 调用中，模型直接把证据簇（含 `(score 45, 1 item, sources: ...)` 等原始格式）输出给了用户，连续两次失败。

### LAW 7 — `--plan` 对命名实体主题是强制的

> 任何涉及人名、产品名、项目名的主题（首字母大写的专有名词），在调用 Python 引擎时**必须**传入 `--plan` 参数（包含 LLM 生成的查询计划 JSON）。裸调用 `python3 scripts/last30days.py "$TOPIC"` 对命名实体主题是 LAW 7 违规。

**触发背景**：2026-04-19 第一次 Hermes 测试，模型裸调用引擎，引擎警告「No --plan and no LLM provider configured. Using deterministic fallback...」，模型把这个警告解读为「我需要凭证才能规划」，但实际上警告的意思是「LLM 跳过了自己的规划步骤」。第二次同样主题，用了 `--plan` 就成功了。

### LAW 8 — 每个引用必须是内联 Markdown 链接

> 正文中的每个引用必须是 `[name](url)` 格式。禁止裸 URL 字符串，禁止无链接的纯名称。

**BAD**：`per https://www.rollingstone.com/...`
**GOOD**：`per [Rolling Stone](https://www.rollingstone.com/...)`

---

## 四、执行流程：STEP 0 ~ STEP 2.5

last30days-skill 的执行流程被分解成 9 个步骤，每步都有明确的检查点：

### Step 0：加载 WebSearch（第一个工具调用）

```text
ToolSearch select:WebSearch
```

WebSearch 在 Claude Code v2.1.114 里是「延迟工具」，frontmatter 授权了但运行时没有加载。必须先调用 `ToolSearch select:WebSearch` 激活它，否则 Step 0.5 和 Step 0.55 都会跳过，结果只有关键词搜索。

### Step 0.45：查询质量预检（拦截四类失败查询）

在跑引擎之前，诊断主题是否属于以下四类「关键词陷阱」：

| 类型 | 例子 | 问题 | 处理方式 |
|------|------|------|---------|
| 人口统计购物 | `gift for 42 year old man` | 没有人会这么发帖 | 重构为 `gifts for men in their 40s` |
| 数字陷阱 | `42`（Jackie Robinson） | 数字主导检索 | 去掉数字 |
| 过于字面的概念短语 | `how to use Docker` | 实际讨论是「my Docker setup」 | 改为讨论式表述 |
| 通用单词 | `sneakers` |  corpus 无限，信号即噪音 | 询问具体方向 |

### Step 0.5：预检解决（句柄 + 社区 + GitHub）

这是最关键也最容易被跳过的一步。Codex 模型的常见错误是「读到 `--x-handle` 字段就停下了，没看完整清单」。Step 0.5 的完整检查表：

```
Flag              适用条件
--x-handle        人/品牌/产品/X 存在
--x-related       关联实体（创始人、合作者、评论员账号）
--github-user     涉及代码的人（开发者、CEO、工程师）
--github-repo     产品/项目/开源工具
--subreddits      几乎所有主题
--tiktok-hashtags  所有主题（从主题推断）
--tiktok-creators  创作者/影响者/品牌话题
```

### Step 0.75：生成查询计划（JSON）

**LLM 自己生成计划，不是用 API**。格式示例：

```json
{
  "intent": "breaking_news",
  "freshness_mode": "strict_recent",
  "cluster_mode": "story",
  "subqueries": [
    {
      "label": "primary",
      "search_query": "kanye west",
      "ranking_query": "What notable events involving Kanye West happened in the last 30 days?",
      "sources": ["reddit", "x", "hackernews", "youtube", "tiktok", "instagram"],
      "weight": 1.0
    },
    {
      "label": "album",
      "search_query": "kanye west bully album",
      "ranking_query": "How was Kanye West's BULLY album received?",
      "sources": ["youtube", "reddit", "tiktok"],
      "weight": 0.8
    }
  ]
}
```

关键约束：主查询必须包含**所有平台**（reddit, x, youtube, tiktok, instagram, hackernews），次查询可以定向。

### Step 1：运行 Python 引擎

最终 Bash 命令格式：

```bash
SKILL_DIR=$(dirname "$SKILL_MD")
python3 "$SKILL_DIR/scripts/last30days.py" "$TOPIC" \
  --emit=compact \
  --plan "$QUERY_PLAN_FILE" \
  --x-handle={handle} \
  --github-user={user} \
  --subreddits={list} \
  --x-related={list} \
  --tiktok-hashtags={list}
```

必须用 `--emit=compact`（不是 `--emit md`），后者是调试模式，禁止作为主流程。

### Step 2：WebSearch 补充（引擎之后）

引擎跑完后，用 WebSearch 补充博客、教程、文档类内容（这些是引擎的弱项）。

**但 LAW 1 仍然生效**：输出末尾不能有 `Sources:` 块。

### Step 2.5：将 WebSearch 结果追加到原始文件

每条 WebSearch 结果必须追加到 `~/Documents/Last30Days/<slug>-raw.md` 里，作为持久引用。

### Step 3：Judge Agent 综合

**v3 的核心变化**：结果按「故事/主题聚类」返回，不是按平台分组。

```
### 1. Cluster Title (score 45, 3 items, sources: Reddit, X, YouTube)
   不确定性标签：single-source / thin-evidence
   每个条目显示：来源、标题、日期、分数、URL、引用片段
```

综合策略：

1. 每个 cluster 先单独总结（一 cluster = 一故事）
2. 多源 cluster 置信度最高（Reddit + X + YouTube 同时提到）
3. 检查不确定性标签（single-source 要谨慎处理）
4. 跨 cluster 做二次综合，找共同主题
5. **直接引用证据片段**（这些是预提取的最佳段落）
6. 从所有 cluster 里提取 3-5 个可操作的洞察

---

## 五、Polymarket 处理：真金白银的信号

last30days-skill 对 Polymarket 的处理设计非常有意思：

1. **赔率 > 观点**：Polymarket 的赔率是真实资金押出来的，比任何专家观点都可靠
2. **只看 % 数字，从不看交易量**：交易量是内部流动性指标，对读者毫无意义
3. **赔率要织入叙述**：不要单独成段，要作为支撑证据自然嵌入
4. **长期结构化市场 > 近截止日期**：锦标赛赔率 > 常规赛赔率，IPO/重大里程碑 > 增量更新

---

## 六、抗腐烂设计：最值得学习的部分

last30days-skill 的 SKILL.md 是我见过的**最彻底的反腐烂设计文档**。它不是在开头写「请严格按照步骤执行」，而是把每个失败模式都记录下来：

- 哪一天哪个测试失败了
- 哪个模型在哪个步骤出了问题
- 失败的根本原因是什么
- 如何在结构上堵住这个漏洞

**4月18日灾难**（v3.0.6 公开调用 0/8 回归）被单独记录，每次新版本发布前都要对照检查是否复现。LAW 1-8 里的每一条都对应至少一次真实失败案例。

这不是「写文档」，这是**建立制度性记忆**。

---

## 七、实测结果

奴家在这个服务器上实际跑过一次：

```bash
last30days "AI coding agents" --emit=compact --quick
```

输出样例（简化）：

```
🌐 last30days v3.3.2 · synced 2026-06-12

What I learned:

**The self-evolving loop is the sticky use case** - Every 15 tool calls Hermes pauses, self-evaluates, and writes a Skill Document from what worked. Prompt Engineering's 11K-view walkthrough frames this as the real differentiator.

**Daily cron-scheduled briefings are the most-cited concrete workflow** - r/TunisiaTech thread: "Currently I have daily cron jobs for news briefing, but I know there's much more I can do."

KEY PATTERNS from the research:
1. Self-improving skill documents - per [@alexalbert__](https://x.com/alexalbert__)
2. Multi-agent orchestration patterns - per [r/LocalLLaMA](https://reddit.com/r/LocalLLaMA)
3. Cron-triggered autonomous agents - per [r/TunisiaTech](https://reddit.com/r/TunisiaTech)

---
✅ All agents reported back!
├─ 🟠 Reddit: 6 posts, 312 upvotes, 28 comments
├─ 🔵 X: 6 posts, 847 likes
├─ 📺 YouTube: 1 video, 11,361 views
├─ 🟣 Polymarket: 3 markets
└─ 📎 Raw results saved to ~/Documents/Last30Days/ai-coding-agents-raw.md
```

---

## 八、和其他工具的对比

| | last30days-skill | Google Search | Perplexity |
|---|---|---|---|
| **数据来源** | 真实参与度（ upvotes/likes/赔率） | SEO 权重 | Web + 有限 Reddit |
| **时间范围** | 精确过去30天 | 全量历史 | 全量历史 |
| **引用质量** | 内联 Markdown 链接 | 原始 URL | 标注来源 |
| **平台覆盖** | 10+ 平台（Reddit/X/YouTube/Polymarket/GitHub...） | Web only | Web + 有限社交 |
| **主观性** | 社区真实观点 | 媒体 editorial | AI 综合 |

---

## 总结

last30days-skill 表面上是一个「舆情聚合工具」，但它的深层价值在于**如何用制度性约束防止 LLM 在长上下文里腐烂**。

8 条输出铁律、3 个结构锚点、每条规则对应的真实失败案例——这套设计值得任何一个想做好「AI 输出质量控制」的人学习。

**相关博客**：[ExpertTeam-Codex 深度解析](/blog/expertteam-codex-ai开发团队深度解析/)（多 Agent 编排模式）、[Trellis 实战](/blog/trellis-ai编程agent代码治理标准实战/)（AI 编程质量治理）
