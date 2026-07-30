---
layout: single
title: "构建个人 AI 量化交易系统：从行情采集到日报统计全链路"
date: 2026-07-13 10:00:00 +0800
categories: [trading, ai]
tags: [okx, deepseek, ai-agent, crypto, quantitative-trading]
---

过去几周我用 Python + DeepSeek AI 搭了一套纯个人用的加密货币量化交易系统，14 个模块覆盖了从行情接入、技术分析、AI 决策到风控执行、日志归档的完整闭环。今天来分享一下它的架构设计和可扩展之处。

## 整体架构

系统采用 **管道-过滤器** 架构，数据单向流动，每个模块职责单一：

```
行情采集 → 指标计算 + 新闻抓取 → AI 分析 → 策略规划 → 风控检查 → 执行引擎 → 持仓管理 → 监控告警 → 日志归档 → 日报统计
```

其中 AI 分析部分调用 DeepSeek API（OpenAI 兼容格式），其余所有模块纯 Python 实现，不依赖任何第三方 ML 框架。

## 14 个阶段的功能概览

### 第一阶段：实时行情（WebSocket）
通过 OKX 公共 WebSocket 接入永续合约的实时 ticker、成交、盘口深度（5档）、资金费率和持仓量数据。程序启动后自动建立连接并维持心跳保活，断线自动重连。

### 第二阶段：K线管理器
同时通过 WebSocket 实时推送 + REST API 历史补全获取 K 线数据。支持 `1m/5m/15m/1H/4H` 五个周期，内存缓存最近 500 根，去重、排序、截断自动处理。

### 第三阶段：技术指标
纯 Python 实现，无需 TA-Lib 等 C 扩展库。涵盖：
- 均线族：EMA(20/60/120)、SMA
- 动量类：RSI(14)、MACD(12/26/9)
- 波动率：ATR(14)、布林带(20/2)
- 强度：ADX(14)
- 成交量：Volume MA(20)、VWAP

### 第四阶段：新闻抓取
异步并发抓取 CoinTelegraph RSS、OKX 官方公告，提取标题、时间、摘要并统一结构化。内置简单的重要性分级规则，支持高/中/低三级标记。

### 第五阶段：市场分析 AI Agent
将 K 线数据和技术指标打包为 Prompt，调用 DeepSeek 进行结构化分析。输出严格的 JSON schema：

```json
{"trend": "bullish|bearish|sideways", "strength": 0.0-1.0, "confidence": 0.0-1.0, "reason": [...]}
```

### 第六阶段：新闻分析 AI Agent
类似地，对新闻标题进行情感/方向分析，输出 impact/direction/confidence/duration 四字段。

### 第七阶段：策略规划 Agent
整合市场分析 + 新闻分析 + 当前持仓，由 DeepSeek 综合判断输出交易计划，包含 entry/stop_loss/take_profit/action。

### 第八阶段：风控引擎（纯规则）
**整个系统中唯一不可被绕过、不可被 AI 修改的模块。** 检查项包括：
- 单币种最大仓位上限
- 每日最大亏损上限
- 连续亏损笔数
- 单笔亏损比例
- 最小盈亏比 1.5
- 资金费率上限
- 异常波动率过滤
- 高重要性新闻自动暂停交易

### 第九阶段：持仓管理器
内存管理当前持仓，支持从 OKX REST API 同步真实仓位，自动计算未实现盈亏，提供移动止损/止盈更新接口。

### 第十阶段：执行引擎
所有下单/平仓/修改操作强制经过风控引擎检查，通过后再调用 OKX API。内置 Demo 模式，可在不连接真实交易所的情况下完整测试。

### 第十一阶段：监控告警
周期性轮询持仓状态、行情连接、资金费率、爆仓风险，异常实时写入独立日志文件。

### 第十二阶段：集中日志
统一 JSONL 格式日志模块，按日归档，异步安全写入。覆盖 prompt、LLM 输出、交易信号、执行结果、错误、API 调用 6 种事件。

### 第十三阶段：记忆模块
SQLite 存储历史分析记录、交易计划和执行结果，供后续查询参考。

### 第十四阶段：日报统计
每日自动统计胜率、总盈亏、年化 Sharpe、最大回撤、平均盈亏比、预估手续费和资金费率成本，写入日志和数据库。

## 关键配置说明

只需修改项目根目录下一个配置文件即可调整大部分行为：

```
# 交易所 API 凭证
OKX_API_KEY / OKX_SECRET_KEY / OKX_PASSPHRASE

# 交易模式（Demo/实盘）
OKX_DEMO = True   → 模拟交易，不产生真实订单
OKX_DEMO = False  → 真实交易，下单到 OKX

# AI 模型配置
DEEPSEEK_API_KEY   → 你的 DeepSeek API Key
DEEPSEEK_BASE_URL  → API 端点（兼容 OpenAI 格式的均可替换）
DEEPSEEK_MODEL     → 模型名称（可换 deepseek-chat 或其他）
```

## 可扩展之处

### 1. 增加交易币种
在配置文件中修改 `SYMBOLS` 列表即可，系统自动订阅行情、K 线、计算指标、纳入分析。

```
SYMBOLS = ["BTC-USDT-SWAP", "ETH-USDT-SWAP", "SOL-USDT-SWAP", "DOGE-USDT-SWAP"]
```

### 2. 增加 K 线周期
修改 `KLINE_INTERVALS` 数组，支持所有 OKX 提供的周期如 `30m`、`2H`、`1D` 等。

### 3. 增加新闻来源
配置文件中的 `NEWS_SOURCES` 字典任意扩展，支持 API 和 RSS 两种类型：

```python
NEWS_SOURCES = {
    "my_custom_source": {
        "type": "rss",
        "url": "https://your-source.com/rss"
    }
}
```

### 4. 替换 AI 模型
LLM 配置使用 OpenAI 兼容格式，可替换为任何标准 API：
- DeepSeek（默认）
- OpenAI GPT 系列
- Anthropic Claude（通过代理）
- 本地部署的 Ollama / vLLM
- 任何兼容 `/v1/chat/completions` 的端点

只需修改 `LLM_BASE_URL` 和 `LLM_API_KEY`。

### 5. 调整风控参数
所有风控阈值集中在 `RISK_CONFIG` 字典，可根据个人风险偏好自由调整。

### 6. 扩展分析内容
三个 AI Agent 的 Prompt 文件（`prompts/` 目录下）可自由编辑，调整分析维度和输出格式。

### 7. 对接其他交易所
执行引擎封装了标准 API 签名和请求逻辑，替换 `OKX_BASE_URL` 和签名方式即可对接 Binance、Bybit 等。

## 运行方式

```bash
# 设置 API Key
export DEEPSEEK_API_KEY="sk-xxx"

# 启动完整系统
cd project && python main.py
```

## 写在最后

这套系统不是为了高频交易或自动化炒币而设计的——它更像一个 **结构化决策辅助工具**：AI 提供分析建议，规则引擎兜底风控，人类最终决定是否执行。每个模块都可以独立使用，比如单独跑行情监控、或只跑新闻分析。

完整的 14 个模块代码量不到 5000 行 Python，全部开源托管在 GitHub。如果你对某个模块的实现细节感兴趣，欢迎在评论区交流。
