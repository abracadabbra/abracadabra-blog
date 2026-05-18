---
title: 'GoldAgent：量化 + LLM 混合驱动的黄金价格分析系统'
description: '让数据说话，让 AI 辩论，让决策有据可依。一个融合技术分析、时序预测与多 Agent 辩论的黄金价格分析系统实战项目。'
pubDate: '2026-05-18'
tags: ['AI', 'Agent', '量化交易', 'LLM', 'Python', 'FastAPI', '黄金', '多Agent辩论']
categories: ['AI开发', '量化金融']
heroImage: '/blog/gold-agent-hero.jpg'
---

> 让数据说话，让 AI 辩论，让决策有据可依

## 项目背景

最近在研究 AI Agent 的实际应用场景时，发现金融领域特别适合 Agent 化改造。传统量化交易依赖硬编码规则，而 LLM 虽然擅长推理但容易产生幻觉。于是我想：**能不能把两者结合起来，让量化数据约束 LLM 的推理边界，同时让 LLM 的多角度分析弥补纯量化策略的盲区？**

这就是 GoldAgent 的初衷——一个**量化 + LLM 混合驱动**的黄金价格分析系统。

## 系统架构总览

```
┌──────────────────────────────────────────────────────────────┐
│                        前端仪表盘                             │
│   金价走势 · 技术指标 · 预测区间 · 辩论结果 · 回测报告        │
│                    Next.js + lightweight-charts               │
└──────────────┬───────────────────────────────────┬───────────┘
               │ REST / WebSocket                  │
┌──────────────▼───────────────────────────────────▼───────────┐
│                     FastAPI 后端                              │
│                                                              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐ │
│  │  数据采集    │ │  量化分析    │ │    LLM 辩论引擎          │ │
│  │             │ │             │ │                          │ │
│  │ · akshare   │ │ · pandas-ta │ │ · 🟢 看多方 (GPT-4.1)    │ │
│  │ · yfinance  │ │ · Prophet   │ │ · 🔴 看空方 (Claude)     │ │
│  │ · FRED API  │ │ · backtrader│ │ · 🔍 审计员 (GPT-4.1m)   │ │
│  │ · RSS 新闻  │ │ · 信号生成  │ │ · ⚖️ 仲裁官 (GPT-4.1)   │ │
│  └──────┬──────┘ └──────┬──────┘ └────────────┬────────────┘ │
│         │               │                     │              │
│  ┌──────▼───────────────▼─────────────────────▼────────────┐ │
│  │                    共享数据层                              │ │
│  │   PostgreSQL + Redis + 本地 Parquet 缓存             │ │
│  └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

## 核心技术亮点

### 1. 多数据源融合采集

黄金价格受多重因素影响，单一数据源远远不够。系统整合了**三大类数据源**：

```python
# 数据源架构
数据采集层
├── 金价数据
│   ├── akshare    → 上海金交所 Au99.99 现货金（国内基准价）
│   ├── yfinance   → XAUUSD 伦敦金 / COMEX 黄金期货
│   └── yfinance   → GLD ETF（全球最大黄金 ETF）
│
├── 宏观指标（影响金价的核心变量）
│   ├── yfinance   → 美元指数 DX-Y.NYB
│   ├── yfinance   → 10年期美债收益率 ^TNX
│   ├── yfinance   → VIX 恐慌指数
│   ├── FRED API   → CPI、联邦基金利率、TIPS 收益率
│   └── FRED API   → M2 货币供应量
│
└── 情绪数据
    ├── RSS        → Google News 'gold price'
    ├── RSS        → Reuters 黄金新闻
    └── RSS        → Kitco 黄金 RSS
```

**为什么选 akshare + yfinance 组合？**
- akshare 覆盖国内黄金市场（上海金交所、沪金期货），数据权威
- yfinance 覆盖国际市场（伦敦金、COMEX、ETF），更新及时
- 两者互补，形成完整的国内外金价视图

### 2. 智能两级缓存

金融数据 API 调用频繁，直接请求容易被限流。设计了 **Redis 热缓存 + Parquet 冷缓存** 的两级策略：

```python
# 缓存策略伪代码
async def get_data(key: str) -> DataFrame:
    # Level 1: Redis 热缓存（TTL 5分钟）
    cached = await redis.get(key)
    if cached:
        return deserialize(cached)
    
    # Level 2: Parquet 本地缓存（按年月分区）
    parquet_path = f"data/cache/{key}/{year_month}.parquet"
    if os.path.exists(parquet_path):
        df = pd.read_parquet(parquet_path)
        await redis.set(key, serialize(df), ex=300)
        return df
    
    # Level 3: 远程 API 拉取
    df = await fetch_from_api(key)
    await redis.set(key, serialize(df), ex=300)
    df.to_parquet(parquet_path)
    return df
```

**Parquet 格式的优势：**
- 列式存储，压缩率高（比 CSV 小 5-10 倍）
- 支持按列查询，读取特定指标时不需要加载全部数据
- 保留数据类型，不需要每次解析

### 3. 200+ 技术指标计算

使用 pandas-ta 库计算全套技术指标，覆盖趋势、振荡、波动率、成交量四大类：

```python
def compute_indicators(df: pd.DataFrame) -> dict:
    """输入 OHLCV DataFrame，输出全部技术指标"""
    return {
        # 趋势指标
        "ma": {
            "ma5":   ta.sma(df["close"], length=5),
            "ma20":  ta.sma(df["close"], length=20),
            "ma60":  ta.sma(df["close"], length=60),
            "ema12": ta.ema(df["close"], length=12),
            "ema26": ta.ema(df["close"], length=26),
        },
        
        # 振荡指标
        "oscillator": {
            "rsi14":  ta.rsi(df["close"], length=14),
            "macd":   ta.macd(df["close"]),  # MACD + Signal + Hist
            "stoch":  ta.stoch(df["high"], df["low"], df["close"]),
        },
        
        # 波动率
        "volatility": {
            "bbands":  ta.bbands(df["close"], length=20),
            "atr14":   ta.atr(df["high"], df["low"], df["close"], length=14),
        },
        
        # 趋势强度
        "trend": {
            "adx":     ta.adx(df["high"], df["low"], df["close"]),
            "supertrend": ta.supertrend(df["high"], df["low"], df["close"]),
        },
    }
```

**信号生成规则：**

| 指标 | 看多得分 | 看空得分 | 最大分值 |
|------|----------|----------|----------|
| MA5 vs MA20 | +20 | -20 | ±20 |
| MA20 vs MA60 | +10 | -10 | ±10 |
| RSI(14) < 30 | +15 | — | ±15 |
| RSI(14) > 70 | — | -15 | ±15 |
| MACD 金叉 | +15 | — | ±15 |
| MACD 死叉 | — | -15 | ±15 |
| 布林下轨 | +10 | — | ±10 |
| 布林上轨 | — | -10 | ±10 |
| Supertrend 多 | +20 | — | ±20 |
| Supertrend 空 | — | -20 | ±20 |
| ADX > 25 | 方向加分 | 方向减分 | ±10 |

**信号分类：** ≥50 强烈看多，≥20 看多，≤-20 看空，≤-50 强烈看空

### 4. Prophet 时序预测 + 外部回归因子

单纯的金价历史数据预测效果有限，因为黄金价格受宏观因素影响很大。系统使用 Facebook 的 Prophet 模型，并**引入外部回归因子**提升预测准确性：

```python
from prophet import Prophet

def predict_gold_price(df: pd.DataFrame, days: int = 7) -> dict:
    """
    输入: 历史金价 DataFrame (ds, y) + 外部变量
    输出: 未来 N 天价格区间 (yhat, yhat_lower, yhat_upper)
    """
    model = Prophet(
        daily_seasonality=True,
        yearly_seasonality=True,
        changepoint_prior_scale=0.05,  # 控制趋势灵活度
    )
    
    # 添加外部变量作为回归因子
    model.add_regressor("usd_index")      # 美元指数
    model.add_regressor("us_10y_yield")   # 美债收益率
    model.add_regressor("vix")            # VIX 恐慌指数
    
    model.fit(df)
    
    future = model.make_future_dataframe(periods=days)
    forecast = model.predict(future)
    
    return {
        "forecast": forecast[["ds", "yhat", "yhat_lower", "yhat_upper"]].tail(days),
        "trend": forecast["trend"].iloc[-1],
        "changepoints": model.changepoints.tolist(),
    }
```

**为什么选择 Prophet？**
- 自带季节性检测（日、周、年周期）
- 支持添加外部回归因子（美元指数、美债、VIX）
- 对缺失值和异常值鲁棒
- 提供不确定性区间（yhat_lower, yhat_upper）

### 5. 多 Agent 辩论架构（核心创新）

这是系统最有意思的部分。借鉴了学术界 MoE（Mixture of Experts）的思想，设计了 **4 个 Agent 各司其职** 的辩论架构：

```
用户提问 / 定时触发
        │
        ▼
┌─ Step 1: 数据采集 ─────────────────────┐
│  DataService 拉取最新金价 + 宏观 + 新闻  │
└──────────────┬─────────────────────────┘
               │
               ▼
┌─ Step 2: 量化信号 ─────────────────────┐
│  QuantService 计算指标 + 预测 + 信号     │
└──────────────┬─────────────────────────┘
               │
               ▼
┌─ Step 3: 辩论 Round 1 ─────────────────┐
│  🟢 看多方: 基于数据构建看多论点 (GPT-4.1)    │
│  🔴 看空方: 基于数据构建看空论点 (Claude)      │
└──────────────┬─────────────────────────┘
               │
               ▼
┌─ Step 4: 数据审计 ─────────────────────┐
│  🔍 审计员: 验证双方数据准确性 (GPT-4.1-mini) │
└──────────────┬─────────────────────────┘
               │
               ▼
┌─ Step 5: 仲裁裁决 ─────────────────────┐
│  ⚖️ 仲裁官: 综合所有输入输出最终判断 (GPT-4.1) │
└──────────────┬─────────────────────────┘
               │
               ▼
         结构化 JSON 输出
```

**每个 Agent 的配置：**

| Agent | 角色 | 模型 | 温度 | 职责 |
|-------|------|------|------|------|
| 🟢 看多方 | Bull Advocate | GPT-4.1 | 0.7 | 构建看多论点，引用数据支撑 |
| 🔴 看空方 | Bear Challenger | Claude Sonnet 4 | 0.7 | 质疑反驳，寻找反面证据 |
| 🔍 审计员 | Data Auditor | GPT-4.1-mini | 0.2 | 验证数据准确性，标记幻觉 |
| ⚖️ 仲裁官 | Chief Arbitrator | GPT-4.1 | 0.4 | 综合裁决，输出趋势判断 |

**为什么用不同模型？**
- 看多方用 GPT-4.1：擅长数据引用和逻辑构建
- 看空方用 Claude：Claude 更擅长批判性思维和质疑
- 审计员用 GPT-4.1-mini：成本低，专注于事实核查
- 仲裁官用 GPT-4.1：综合能力强，适合最终裁决

**结构化输出示例：**

```json
// 看多方输出
{
  "stance": "bullish",
  "confidence": 72,
  "arguments": [
    {"point": "RSI 未超买", "evidence": "RSI=58.3", "strength": "medium"},
    {"point": "美元走弱", "evidence": "DXY 102.3→101.8", "strength": "strong"}
  ],
  "price_target": {"low": 3220, "high": 3300, "currency": "USD/oz"},
  "key_risk": "美债收益率上行可能压制金价"
}

// 仲裁官输出
{
  "verdict": "bullish",
  "confidence": 65,
  "price_range": {"low": 3220, "high": 3300},
  "time_horizon": "1w",
  "key_reasons": ["技术面偏多", "央行持续购金", "地缘风险支撑"],
  "risk_warnings": ["美债收益率上行", "CPI 超预期"],
  "final_advice": "震荡偏多，建议轻仓做多，止损 3200"
}
```

**辩论成本：** 单次约 $0.02-0.05（4 次 LLM 调用）

### 6. 策略回测引擎

使用 backtrader 框架实现策略回测，内置三种经典策略：

```python
class GoldenCrossStrategy(bt.Strategy):
    """MA 金叉死叉 + RSI 过滤 + ATR 止损"""
    params = (
        ("fast_ma", 20),
        ("slow_ma", 60),
        ("rsi_threshold", 40),
        ("atr_stop_mult", 2.0),
    )

    def next(self):
        # 金叉 + RSI 未超买 → 买入
        if (self.ma_fast[0] > self.ma_slow[0] and
            self.ma_fast[-1] <= self.ma_slow[-1] and
            self.rsi[0] > self.p.rsi_threshold):
            self.buy()

        # 死叉 → 卖出
        elif (self.ma_fast[0] < self.ma_slow[0] and
              self.ma_fast[-1] >= self.ma_slow[-1]):
            self.close()
```

**回测指标：**
- Sharpe Ratio（夏普比率）
- 最大回撤（Max Drawdown）
- 胜率（Win Rate）
- 总交易次数
- 资金曲线

## 技术栈总结

| 层级 | 技术 | 用途 | 选型理由 |
|------|------|------|----------|
| 后端框架 | FastAPI | Web 服务 | 异步高性能，自动 OpenAPI 文档 |
| 数据采集 | akshare + yfinance | 金价数据 | 国内外数据源互补 |
| 宏观数据 | FRED API | 经济指标 | 美联储官方数据，免费权威 |
| 技术指标 | pandas-ta | 指标计算 | 纯 Python，200+ 指标 |
| 时序预测 | Prophet | 价格预测 | 自带季节性，支持外部回归 |
| 回测框架 | backtrader | 策略验证 | 成熟稳定，文档完善 |
| LLM 调用 | OpenAI SDK | 多 Agent | 兼容所有主流模型端点 |
| 数据库 | PostgreSQL | 持久化 | 时序数据 + JSON 支持 |
| 缓存 | Redis | 热数据 | 内存数据库，pub/sub 支持 |
| 前端 | Next.js + lightweight-charts | 可视化 | TradingView 开源图表库 |
| 部署 | Docker Compose | 容器化 | 一键部署 |

## 项目结构

```
gold-agent/
├── src/gold_agent/
│   ├── config.py              # Pydantic Settings 配置管理
│   ├── main.py                # FastAPI 入口 + 生命周期
│   │
│   ├── data/                  # 数据采集层
│   │   ├── gold_price.py      # 金价数据 (akshare + yfinance)
│   │   ├── macro.py           # 宏观数据 (FRED + yfinance)
│   │   ├── news.py            # 新闻情绪 (RSS + 关键词)
│   │   └── cache.py           # 两级缓存 (Redis + Parquet)
│   │
│   ├── quant/                 # 量化分析层
│   │   ├── indicators.py      # 技术指标 (pandas-ta)
│   │   ├── predictor.py       # Prophet 时序预测
│   │   ├── signals.py         # 多指标信号生成
│   │   └── backtest/
│   │       └── engine.py      # backtrader 回测引擎
│   │
│   ├── debate/                # LLM 辩论层
│   │   ├── agents.py          # 4 Agent 配置
│   │   ├── engine.py          # 辩论流程编排
│   │   ├── llm.py             # OpenAI 兼容调用封装
│   │   └── prompts.py         # 提示词模板
│   │
│   ├── api/                   # API 路由
│   │   ├── analysis.py        # 分析接口
│   │   ├── debate.py          # 辩论接口
│   │   ├── backtest.py        # 回测接口
│   │   └── websocket.py       # WebSocket 实时推送
│   │
│   └── db/
│       └── models.py          # SQLAlchemy 9 张表模型
│
├── tests/                     # pytest 单元测试
├── frontend/                  # Next.js 前端 (规划中)
├── Dockerfile                 # 容器镜像
├── docker-compose.yml         # 编排配置
└── pyproject.toml             # 项目元数据
```

## API 接口一览

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/analysis/gold` | GET | 获取金价数据 |
| `/api/analysis/indicators` | GET | 技术指标计算 |
| `/api/analysis/signal` | GET | 交易信号生成 |
| `/api/analysis/predict` | GET | Prophet 时序预测 |
| `/api/analysis/macro` | GET | 宏观数据 |
| `/api/analysis/news` | GET | 新闻情绪分析 |
| `/api/debate/run` | POST | 运行完整 4 Agent 辩论 |
| `/api/debate/quick` | GET | 快速分析 |
| `/api/backtest/run` | GET | 运行策略回测 |
| `/ws/{client_id}` | WS | WebSocket 实时推送 |

## 实际应用场景

1. **每日晨报生成**：定时触发数据采集 + 量化分析 + LLM 辩论，生成结构化的黄金市场晨报
2. **实时信号监控**：WebSocket 推送金价变动和交易信号
3. **策略验证**：回测不同参数的交易策略，验证历史表现
4. **多角度分析**：通过辩论机制避免单一视角的偏见

## 踩坑记录

### 1. akshare 接口不稳定
akshare 的某些接口在国内访问时偶尔会超时，解决方案：
- 增加重试机制（3 次重试，指数退避）
- Redis 缓存兜底，即使 API 超时也能返回缓存数据

### 2. Prophet 添加回归因子的坑
Prophet 的 `add_regressor` 要求训练数据和预测数据都包含该列，否则会报错。解决方案：
- 在预测前检查未来数据框是否包含所有回归因子
- 缺失的回归因子用最近 N 天的均值填充

### 3. 多 Agent 辩论的 prompt 设计
最初设计的 prompt 太宽泛，导致 Agent 输出不一致。解决方案：
- 使用结构化 JSON schema 约束输出格式
- 在 prompt 中明确要求引用具体数据（如 "RSI=58.3"）
- 设置较低的 temperature（0.2-0.7）减少随机性

### 4. backtrader 数据格式要求
backtrader 要求 DataFrame 必须有 datetime index，且列名必须是 open/high/low/close/volume。解决方案：
- 统一数据预处理函数，确保格式一致
- 使用 `bt.feeds.PandasData` 自定义数据源

## 未来规划

1. **前端仪表盘**：Next.js + lightweight-charts 实现可视化
2. **更多数据源**：加入 Reddit 情绪、CFTC 持仓报告
3. **强化学习**：用 RL 优化辩论权重和信号阈值
4. **多品种支持**：扩展到白银、铂金等贵金属
5. **实盘对接**：接入券商 API 实现自动化交易

## 总结

GoldAgent 的核心创新在于**量化与 LLM 的深度融合**：
- 量化数据约束 LLM 的推理边界，减少幻觉
- LLM 的多角度分析弥补纯量化策略的盲区
- 多 Agent 辩论机制避免单一视角的偏见
- 结构化输出让结果可解释、可验证

这个项目让我对 AI Agent 的实际应用有了更深的理解。Agent 不是万能的，但在特定领域（如金融分析）结合领域知识和数据，能发挥出远超传统方法的价值。

**项目地址：** [https://github.com/abracadabbra/gold-agent](https://github.com/abracadabbra/gold-agent)

---

*如果觉得有用，欢迎点个 Star ⭐️*
