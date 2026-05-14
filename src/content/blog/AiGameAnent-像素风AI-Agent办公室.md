---
title: 'AiGameAnent：把 AI Agent 做成像素风办公室模拟游戏'
description: '深度解析 AiGameAnent 项目如何用 Phaser.js 游戏引擎打造等距视角的 AI Agent 协作工作室，多 Agent 分工开发 HTML5 小游戏'
pubDate: '2026-05-14 10:00:00'
tags: ["AI工具", "多Agent", "游戏开发", "Phaser", "TypeScript"]
categories: ["AI工具"]
---

## 前言

AI Agent 协作工具见过不少，但把 Agent 做成**像素风办公室模拟游戏**的，AiGameAnent 是第一个。

想象一下：你打开浏览器，看到一个等距视角的办公室，每个工位上坐着一个像素小人——那是你的 AI Agent。程序区的小人在写代码，美术区的小人在画图，会议室里三个总监在开立项会……

这篇文章带你深入解析这个项目的实现原理，看看它是怎么把 AI 工作流变成一个可视化的"经营游戏"的。

## 项目概览

**AiGameAnent**（又名 aiGameGongfang Studio）是一个面向 HTML5 页游和微信/抖音小游戏的多 Agent 工作流仓库。

- **GitHub**: [sconi789/AiGameAnent](https://github.com/sconi789/AiGameAnent)
- **Stars**: 32 | **Forks**: 9
- **语言**: TypeScript
- **许可**: MIT

核心特点：
1. **等距像素风办公室** — 用 Phaser 游戏引擎渲染，不是 DOM
2. **多 Agent 分工** — 6 个部门、19+ 专业角色
3. **本地推理支持** — 连接 Ollama / vLLM / LM Studio
4. **完整工作流** — 从立项会到代码交付，全流程可视化

## 技术栈

```
前端: Phaser 3 (游戏引擎) + Vite + TypeScript
后端: Node.js (studio-server)
共享: packages/shared (类型 + 事件定义)
```

核心依赖只有 **Phaser 3**，没有 React/Vue/Angular，整个 UI 都是游戏引擎渲染的。

## 等距视角是怎么实现的？

### 坐标转换

等距视角（Isometric）是 2D 游戏模拟 3D 感的经典技术。AiGameAnent 用的是标准的 2:1 等距比例：

```typescript
const ISO = {
  tileW: 64,  // 地砖宽度
  tileH: 32,  // 地砖高度（2:1 比例）
  originX: 140,
  originY: 140
};

// 网格坐标 → 屏幕坐标
function isoToScreen(gx: number, gy: number) {
  const x = (gx - gy) * (ISO.tileW / 2) + ISO.originX;
  const y = (gx + gy) * (ISO.tileH / 2) + ISO.originY;
  return { x, y };
}
```

这个公式把二维网格坐标转换成菱形排列的屏幕位置，形成经典的等距视角效果。

### 地板渲染

用双重循环遍历整个网格，每个格子放一个菱形地砖：

```typescript
for (let gx = 0; gx < gridW; gx++) {
  for (let gy = 0; gy < gridH; gy++) {
    const p = isoToScreen(gx, gy);
    const tile = this.add.image(p.x, p.y, "px_iso_floor")
      .setOrigin(0.5, 0);
    tile.setDepth(p.y - 2000);  // 按 Y 坐标排序深度
  }
}
```

注意 `setDepth(p.y - 2000)` 这个技巧——等距视角下，靠下的物体应该遮挡靠上的物体，用 Y 坐标作为深度值可以自然实现这个效果。

## 办公室布局：6 个部门分区

办公室按部门划分为 6 个区域，每个区域是一个矩形网格：

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   ┌──────────┐  ┌──────────────────┐  ┌──────────┐    │
│   │ 领导/制作 │  │                  │  │ 美术/音频 │    │
│   │  (8×4)   │  │   程序/工程       │  │  (10×8)  │    │
│   └──────────┘  │   (12×13)        │  ├──────────┤    │
│                 │                  │  │ 叙事/本地 │    │
│   ┌──────────┐  │                  │  │  (10×5)  │    │
│   │ 策划/设计 │  └──────────────────┘  ├──────────┤    │
│   │  (10×8)  │                        │ QA/发布  │    │
│   │          │  ┌────────┐            │  (10×7)  │    │
│   └──────────┘  │ 会议室  │            └──────────┘    │
│                 └────────┘                             │
│              ┌──────────────────┐                      │
│              │  公共服务 · 休闲   │                      │
│              └──────────────────┘                      │
└─────────────────────────────────────────────────────────┘
```

代码中的定义：

```typescript
const zones: ZoneDef[] = [
  { id: "leadership",  title: "领导/制作",       gx: 2,  gy: 2,  gw: 8,  gh: 4 },
  { id: "design",      title: "策划/设计",       gx: 2,  gy: 7,  gw: 10, gh: 8 },
  { id: "programming", title: "程序/工程",       gx: 13, gy: 2,  gw: 12, gh: 13 },
  { id: "art_audio",   title: "美术/音频",       gx: 27, gy: 2,  gw: 10, gh: 8 },
  { id: "narrative",   title: "叙事/本地化",     gx: 27, gy: 11, gw: 10, gh: 5 },
  { id: "qa_release",  title: "QA/发布/平台",    gx: 27, gy: 17, gw: 10, gh: 7 }
];
```

## 工位系统：每个 Agent 的"家"

每个 Agent 有一个工位，由以下元素组成：

```typescript
private spawnDesk(agentId: string, gx: number, gy: number, idx: number) {
  const p = isoToScreen(gx, gy);
  
  // 桌子
  const desk = this.add.image(p.x - 48, p.y + 46, "px_desk")
    .setOrigin(0, 1);
  
  // 头像
  const avatar = this.add.image(p.x - 42, p.y + 6, "px_avatar")
    .setOrigin(0, 0);
  
  // 状态图标（空闲/工作中/摸鱼）
  const statusIcon = this.add.image(p.x - 26, p.y + 4, "px_status_idle")
    .setOrigin(0.5, 0.5);
  
  // 名牌
  const label = this.add.text(p.x - 54, p.y + 82, agentLabel(agentId), {
    fontFamily: "monospace",
    fontSize: "12px",
    color: "#e7ecff"
  });
  
  // 深度排序：名牌 > 状态 > 头像 > 桌子
  desk.setDepth(deskY);
  avatar.setDepth(avatar.y + 999);
  statusIcon.setDepth(avatar.y + 1000);
  label.setDepth(avatar.y + 1001);
}
```

素材全是**像素风格精灵图**（`px_` 前缀），视觉上类似《Game Dev Story》那种经营游戏。

## Agent 行为动画

### 站立呼吸动画

Agent 站在工位上时有轻微的上下浮动，模拟"呼吸"效果：

```typescript
// idleBobTween — 站立时的呼吸动画
// 移动前必须停掉，否则会与位移 tween 抢 y 导致「瞬移」感
desk.idleBobTween = this.tweens.add({
  targets: avatar,
  y: avatar.y - 2,
  duration: 800,
  yoyo: true,
  repeat: -1
});
```

### 摸鱼系统

Agent 不是一直坐在工位上的，它们会随机去公共设施区域"摸鱼"：

```typescript
type Desk = {
  // 归属工位格（摸鱼区）；外出仅去公共设施，之后回此格
  homeGx: number;
  homeGy: number;
  // 到此时间应开始走回工位
  breakReturnAfterAt?: number;
  // 上次「出门摸鱼」出发时刻，用于拉长工位上发呆间隔
  lastBreakTripAt?: number;
};
```

### 寻路系统

有完整的 A* 寻路，支持路径可视化：

```typescript
// 寻路可视化：地面路径折线（忽略其他占格，仅静态障碍）
navPathGfx?: Phaser.GameObjects.Graphics;

// 寻路可视化：头顶目标说明
navHint?: Phaser.GameObjects.Text;
```

## UI 层：游戏内面板

虽然主体是 Phaser 渲染，但 UI 面板用的是传统 HTML/CSS，通过抽屉式设计弹出：

### 布局结构

```
┌─────────────────────────────────────────────────────────┐
│ [HUD 左栏]                           [菜单栏 右上]       │
│ ┌─────────────┐                      ┌─────────────┐    │
│ │ 秘书提示     │                      │ 设置 │ 策略 │    │
│ │ Agent 花名册  │                      └─────────────┘    │
│ └─────────────┘                                         │
│                                                         │
│                    [Phaser 画布]                         │
│                   (等距办公室)                            │
│                                                         │
│                              [底部菜单栏]                │
│                    ┌──────────────────────┐              │
│                    │ 招聘 │ 队列 │ 财务 │ 通知 │          │
│                    └──────────────────────┘              │
└─────────────────────────────────────────────────────────┘
```

### 抽屉面板

点击游戏内的物品或菜单按钮，会打开对应的管理面板：

- **招聘中心** — 雇佣/解雇 Agent，绑定模型
- **会议室** — 开立项会，生成章程
- **设置** — 配置本地/云端模型
- **策略** — 调整各部门工作模式
- **队列** — 查看任务队列
- **财务** — 统计 token 用量和成本

## 场景一：立项会怎么开？

这是最有意思的功能——AI Agent 在会议室里开"立项会"。

```typescript
// 立项会脚本（规则模式）
const script = [
  { delay: 0, speaker: "秘书", 
    message: "各位好，立项会现在开始。今天议题是「xxx」。" },
  { delay: 700, speaker: "秘书", 
    message: "议程：① 制作人定主线与里程碑 ② 技术评估 ③ 创意对齐" },
  { delay: 1600, speaker: "制作人", 
    message: "我想把「xxx」收成「最小可玩」：一条核心循环 + 两到三个里程碑。" },
  { delay: 2800, speaker: "技术总监", 
    message: "赞同。先做可本地打开的 HTML 试玩包即可。" },
  { delay: 4000, speaker: "创意总监", 
    message: "玩法上希望章程里写清一句「爽点」，再拆 3～7 条勾选式验收点。" },
];
```

如果有可用的 LLM，会调用模型生成更真实的会议对话；否则用规则脚本模拟。

## 场景二：任务派单

选一个 Agent，输入任务描述，点"入队"：

```typescript
enqueueBtn.onclick = async () => {
  const resp = await fetch(`${studioHttp}/api/queue/enqueue`, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ 
      agentId, 
      task, 
      priority, 
      providerId: providerSelect.value 
    })
  });
};
```

任务进入队列后，对应的 Agent 头顶会出现状态变化，工位上的小人开始"忙碌"。

## 场景三：模型路由

支持为每个 Agent 绑定不同的模型提供商：

- **本地文本** — Ollama 等本地模型
- **云端文本** — Moonshot、DeepSeek、MiniMax 等
- **豆包绘图** — 图像生成
- **AI 音乐** — 音频生成

```typescript
// 绑定 Agent 到 Provider
await fetch(`${studioHttp}/api/providers/bind`, {
  method: "POST",
  body: JSON.stringify({ agentId, providerId })
});
```

## 本地运行

```bash
# 克隆项目
git clone https://github.com/sconi789/AiGameAnent.git
cd AiGameAnent

# 安装依赖
npm install

# 启动（服务端 + Web 一条命令）
npm run dev

# 访问 http://127.0.0.1:8787
```

也可以分别启动前后端：

```bash
npm run dev:server  # 后端
npm run dev:web     # 前端 (Vite, port 5173)
```

## 配置模型

复制 `.env.example` 为 `.env`，填入你的 API 配置：

```bash
# 本地模型（Ollama）
LOCAL_BASE_URL=http://localhost:11434/v1
LOCAL_MODEL=qwen2.5:7b

# 云端模型
CLOUD_BASE_URL=https://api.moonshot.cn/v1
CLOUD_MODEL=moonshot-v1-8k
CLOUD_API_KEY=sk-xxx
```

## 适用场景

| 场景 | 推荐度 |
|-----|-------|
| 学习多 Agent 协作架构 | ⭐⭐⭐⭐⭐ |
| 快速原型 HTML5 小游戏 | ⭐⭐⭐⭐ |
| 本地 LLM 玩具项目 | ⭐⭐⭐⭐ |
| 生产级游戏开发 | ⭐⭐ |

## 总结

AiGameAnent 的核心创意在于：**用游戏化的方式可视化 AI 工作流**。

它不是最强大的多 Agent 框架，但可能是最有趣的。等距像素风办公室让抽象的 Agent 协作变得直观可见——你真的能看到"程序在写代码"、"美术在画图"、"总监在开会"。

如果你对多 Agent 协作感兴趣，或者想找一个有趣的本地 LLM 项目玩玩，AiGameAnent 值得一试。

---

**相关阅读**：
- [Oh My ClaudeCode Team 模式深度解析](/blog/Oh-My-ClaudeCode-Team模式深度解析)
- [Oh My ClaudeCode 19 个 Agent 角色全解析](/blog/Oh-My-ClaudeCode-19个Agent角色全解析)
- [Hermes Agent 调 Claude 子代理全流程实录](/blog/hermes-agent调claude子代理全流程实录)
