---
title: '【Harness Engineering】Harness Engineering 的本质：三大支柱与社区考古'
description: '从赛博史馆考古到三大支柱（Three Pillars）拆解，深入理解 Harness Engineering 的本质与 OAI 生产级实践。'
pubDate: '2026-05-04'
tags: ['AI', 'Harness Engineering', 'Agent', 'Context Engineering', 'IMPACT', '架构']
categories: ['AI开发']
---

Harness engineering 最早由 Mitchell Hashimoto 在 [My AI Adoption Journey – Mitchell Hashimoto](https://mitchellh.com/writing/my-ai-adoption-journey) 取名，紧接着 OAI ~~(蹭热度还得是你)~~ 在 2 月 11 日通过 Ryan Lopopolo [Harness engineering: leveraging Codex in an agent-first world | OpenAI](https://openai.com/index/harness-engineering/) 用一个完整的生产案例成功让这个词出圈。

![image|690x344, 50%](upload://jNEB5FopFLZrimXMOtdKW0GERjv.png)
![image|690x148, 50%](upload://qfY8q1rSkh6FuujvLLpaVVkbuAh.png)

> 一言以蔽之：模型之外皆工程

不过，经过赛博史馆的考古捏，发现 `swyx` 在 2025 年 3 月就有相关理论提出了，当时叫做 `"agent engineering" + IMPACT framework`。

> [!info] **IMPACT** *framework*
> **I**ntent: *我们想做什么*，这里需要我们作为用户清晰表达，并且有一定的方法办法可以用来验证 ~~表达不清楚也可以让opus 4.6替我们表达，这样gpt可以稳稳接住并且补一刀。~~     
> **M**emory: *agent需要记住什么*，跨`session`的`memory`，一些常见的workflow也可以涵盖其中 ~~这样就不用化身产品，对agent吆五喝六了。~~                
> **P**lanning：*规划&拆分*，需要能精细规划，并且可以根据实际情况及时调整。                   
> **A**uthority：什么时候自主决定，什么时候用户介入。                        
> **C**ontrol Flow： *LLM 驱动的控制流*，~~LLM-in-the-loop，也是可以随着项目演进可以自行进化的。~~
> **T**ools: *agent能调用一套东西*，好的吧所有别的都屈居于`tools`了

呃🤔 好像所谓 `Harness Engineering` 又是一次硬造新词的表达方式了？这个不就是之前写过的 `context engineering` (context window 里到底应该塞哪些 token) [Effective context engineering for AI agents \ Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) + `action space/environment` + `feedback loop`？

![image|261x193, 75%](upload://7BEuXSShtl1v28xQNa7nUvDy11r.png)

于是，还是三分而立（`Three Pillars`）更容易理解：

* **I：context engineering (内容工程)**：把目标、约束、相关知识塞进 context。
* **II：action space ≈ agent environment (操作空间)**：agent 能在怎样的环境里大展身手：tools、sandbox、permissions、repo 结构、本地文档布局，etc。          
* **III：feedback loops (闭环反馈)**：传感器，测试，log，etc。

---

这样说来呢

* [【2026/04/06更新 issue-driven-workflow】Codex / CC 化身issues工单规划/测试/重写的勤劳牛🐎](https://linux.do/t/topic/1355015)
* [【四个SKILL梦幻联动】一套做PPT的skills。内置了战略咨询模版 + 用到了一些独特的方法](https://linux.do/t/topic/1827145)
* [【基本完结】三句话我让Codex SKILLs 给我生成了一篇LaTex Review - SKILLs果然有替代很多MCP功能的潜质](https://linux.do/t/topic/1369252)

本质上就是 harness engineering 的早期实践了，每个项目里面都包含了 `Three Pillars` 的内容。

---

我们再回到 OAI 的那篇文章：

> Building software still demands discipline, but the discipline shows up more in the scaffolding rather than the code.

> [!tip] I. context engineering (内容工程)
> i. **AGENTS.md** 当作地图 -> 保持到 100 行以内 (其实就是 progressive disclosure，可以直接引用 `docs/` 里面类似于 `design-docs / exec-plans / product-specs` 酱的内容，~~OAI用了 `*-llms.txt`这样的后缀，不过感觉大家可能日常写md多一点吧~~)
> ii. 疯狂 `plan/` 根据任务大小有不同的方法设计 plan
> iii. `Agent legibility is the goal`，任务需要的内容必须要对 agent 可见

> [!success] II. action space (操作空间)  
> i. `Agent` 灵活调用各种工具(gh、本地脚本、内嵌 skills)，解放用户双手
> ii. 擅用 `worktree` [多 agent 不会相冲 + 独立并且可观察 `observability`]，大型项目严格的分层结构 (不同层之间使用 api 沟通，看起来是为了防止 debug 的时候乱改改一串)
> iii. 使用稳定的技术 (原文说的是`无聊`)。确保 api 稳定，基础模型已经熟练运用的技术，方便排列组合 (看起来是宁愿让 agent 自己造轮子，也不要用不透明/不稳定的第三方库)

> [!example] III. feedback loops (闭环反馈)             
> i. `Ralph Wiggum Loop` 这个风挺大的，召唤 agent 小伙伴来 review 酱
> ii. 高度自定义的 linter -> `结构化日志 + 命名规范 + 文件大小限 + etc.` ~~也算是大厂标配吧，不过 linter 里面注入了给 agent 的下一步方案~~ 小车快跑，`diff` 要小，`pr` 出了就尽快跑，好了就 merge 不好就重修。
> iii. 高强度自动修正现有文档，总结归纳，改进 `context engineering`

---

梦到哪里写哪里 
🐱 🐈

![image|225x224, 50%](upload://s8ZQdvpQEED5OQRGgvRKP6bGDUT.png)
