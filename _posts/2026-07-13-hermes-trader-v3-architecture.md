---
title: "Hermes Trader V3 — AI 自动交易系统全链路架构"
date: 2026-07-13
categories: [技术, 量化交易]
tags: [Python, OKX, DeepSeek, 交易系统, 架构设计]
---

## 项目概述

Hermes Trader 是一个基于 Python 的 7×24 小时自动加密货币交易系统。当前版本 V3 已完成全链路集成，所有核心模块均已接入主交易流水线。

**核心原则：**
- LLM 只负责分析和策略建议，**所有风控和安全由程序实现**
- Signal Validator 拥有**最高否决权**，LLM 不可绕过
- 连续亏损 ≥ 3 笔自动进入冷却，断路器 5 次失败自动熔断

---

## 架构总览

![架构图](/architecture-diagram.html)

### 16 步交易流水线

```
Market Data → Indicators → News Pipeline → Priority Scheduler
→ Health Manager → State Machine → Portfolio Manager
→ AI Gateway → Strategy Planner (DeepSeek)
→ ⚠️ Signal Validator (最高否决权)
→ Risk Engine → Execution Permission
→ Execution Engine → StateMachine Update → Replay → Memory → 推送
```

---

## 核心模块

### 📊 市场数据层 (Market Data)

通过 OKX WebSocket 获取实时行情数据，每 30 秒更新技术指标缓存。

| 数据 | 来源 |
|------|------|
| 实时价格 | WebSocket Ticker |
| K 线数据 | WebSocket Candle (1m/5m/15m/1H/4H) |
| 持仓量 (OI) | WebSocket Open Interest |
| 资金费率 | WebSocket Funding Rate |
| 盘口深度 | WebSocket Orderbook (5档) |

### 📰 新闻流水线 (News Pipeline)

8 个新闻源异步采集，经分类→去重→评分后选择性送 LLM。

**新闻源：**
- 交易所公告：OKX、Binance、Bybit
- 新闻媒体：CoinDesk、CoinTelegraph、The Block、Decrypt、Blockworks

**评分阈值：**

| 分数 | 处理方式 |
|------|---------|
| 0~15 | ❌ 丢弃 |
| 16~30 | 📝 记录日志 |
| 31~35 | 💾 存库备用 |
| 36~45 | 🤖 送 DeepSeek 分析 |
| 46+ | 🚨 紧急，立即触发 |

### ⚡ 优先级事件调度器 (Priority Event Scheduler)

| 优先级 | 事件类型 | 说明 |
|--------|---------|------|
| 100 | 安全事件 (Hack/Exploit) | 最高优先 |
| 95 | 交易所上币公告 | |
| 90 | ETF 相关 | |
| 85 | 监管/SEC | |
| 70 | 巨鲸异动 | |
| 50 | 资金费率 > \|0.008\| | |
| 40 | ATR 波动率 > 均值 × 2.5 | |
| 30 | OI 15 分钟变化 > 12% | |
| 20 | 普通新闻 | |

### 🏥 健康管理器 (Health Manager)

监控 DeepSeek、OKX、数据库、WebSocket。健康分控制交易状态：

| 健康分 | 状态 |
|--------|------|
| 100 | ✅ 正常 |
| ≥80 | ⚠️ 警告 |
| ≥60 | 🚫 禁止新开仓 |
| ≥40 | 🔒 仅允许平仓 |
| <40 | ⛔ 系统停止 |

### 🔁 状态机 (State Machine)

```
INIT → NORMAL → READY → OPENING → LONG/SHORT → EXITING → NORMAL
                          ↓ (连续亏损 ≥ 3 笔)
                      COOLDOWN (30 分钟) → NORMAL
```

- LLM **不可修改**状态
- 非法状态转移自动拒绝
- 冷却/熔断到期自动恢复

### ⚠️ Signal Validator（最高否决权）

纯程序规则验证 LLM 策略，LLM 不可绕过：

| 规则 | 条件 |
|------|------|
| EMA 排列 | LONG 需 EMA20 > EMA60，SHORT 需 EMA20 < EMA60 |
| RSI 边界 | LONG ≤ 75，SHORT ≥ 25；≥80/≤20 直接否决 |
| ATR 过滤器 | 当前 ATR > 20 根均值 × 3 时 HOLD |
| 新闻冲突 | 新闻看跌 + LLM 做多 → confidence 降权/否决 |

### 🛡️ 风控引擎 (Risk Engine)

输出统一 `TradePermission` 对象，Execution 仅消费此对象：

- 单币种最大仓位 \$5,000
- 每日最大亏损 \$500
- 连续亏损 ≥ 3 笔停止开仓
- 单笔最大亏损 ≤ 2%
- 最小盈亏比 ≥ 1.5
- 最大杠杆 3x
- 资金费率 > |0.01| 禁止开仓

### ⚙️ 执行引擎 (Execution Engine)

- **指数退避重试：** 1s → 2s → 4s → 8s → 16s（最多 5 次）
- **断路器：** 连续 5 次 API 失败 → 熔断 5 分钟
- **幂等控制：** 同一币种同方向已有挂单时跳过重复
- **动态仓位：** 按风险金额(1%) × 止损距离 × 杠杆 自动计算张数

### 💾 回放系统 (Replay)

每次分析（含 HOLD）完整记录：
- Market / Indicators / News Snapshot
- Prompt 版本 & Hash
- LLM 输出 & Token 用量
- Validator / Risk / Execution 结果
- State Machine & Portfolio 快照

支持按 Session ID 查询和完整交易链回放。

### 🔌 推送通知

- **微信：** 每小时状态报告 + 实时交易信号
- **信号文件：** `/tmp/trade_signal.txt` 供 cron 读取

---

## 技术栈

| 组件 | 技术 |
|------|------|
| 运行环境 | Python 3.13+ |
| 交易所 API | OKX v5 REST + WebSocket |
| LLM | DeepSeek V4 (OpenAI Compatible) |
| 数据库 | SQLite (aiosqlite) |
| 异步框架 | asyncio / aiohttp |
| 信号推送 | Hermes Gateway (WeChat) |
| 调度 | 内置 Scheduler + cron |

---

## 项目结构

```
trader_daemon.py       # 主守护进程
config/settings.py     # 配置（API Key 从 .env 读取）
market/               # 行情模块（WS + K线 + 指标）
news/                 # 新闻系统（采集 + 分类 + 评分 + 流水线）
agents/               # AI Agent（市场分析 + 新闻分析 + 策略规划）
risk/                 # 风控（引擎 + Signal Validator）
execution/            # 执行（引擎 + 断路器 + 仓位计算 + 权限）
positions/            # 持仓管理
portfolio/            # 投资组合管理
core/                 # 核心（状态机 + 事件调度 + 回放 + 健康 + 看门狗）
memory/               # 持久化存储
llm/                  # AI Gateway
logger/               # 日志
daily_review/         # 日报
prompts/              # LLM 提示词
```

---

## 运行

```bash
# 1. 配置 .env（API Key 等）
cp .env.example .env
# 编辑 .env 填入你的 Key

# 2. 安装依赖
pip install -r requirements.txt

# 3. 启动
python3 trader_daemon.py

# 4. 停止
touch /tmp/trader_daemon.stop
```

---

## 安全说明

- 所有 API Key 通过 `.env` 文件读取，**不出现在源码中**
- `.env.example` 为模板，可安全提交到 Git
- 风控逻辑全部由程序实现，LLM 无权绕过
- Signal Validator 拥有最高否决权
- 断路器确保 API 故障时自动熔断

---

> **注意：** 本项目当前在 OKX 模拟盘运行，不涉及真实资金。实盘需自行承担风险。
