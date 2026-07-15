---
title: "量化交易系统 V4 使用说明"
date: 2026-07-15
categories: [技术, 交易]
tags: [OKX, 量化交易, Python, Demo]
---

## 概览

OKX USDT 永续合约量化交易系统，DeepSeek LLM 驱动分析策略，Demo 模拟盘运行。

**部署路径：** `/home/new/project/`
**Python 环境：** `/home/new/project/venv/`
**交易模式：** `demo`（模拟盘，携 `x-simulated-trading: 1`）
**监控币种：** BTC-USDT-SWAP, ETH-USDT-SWAP, DOGE-USDT-SWAP

---

## 常用命令

### 启动守护进程 (tmux 后台)

```bash
cd /home/new/project && source venv/bin/activate && \
tmux new-session -d -s trader 'python -u trader_daemon.py 2>&1 | tee /tmp/trader_daemon.log'
```

### 查看运行状态

```bash
# 查看 tmux 会话
tmux ls

# 查看 daemon 进程
ps aux | grep trader_daemon
```

### 查看实时日志

```bash
# tmux 实时窗口（附加到会话）
tmux attach -t trader

# 仅查看最新 N 行（不进入会话）
tmux capture-pane -t trader -p -S -20

# 日志文件实时 tail
tail -f /tmp/trader_daemon.log
```

### 停止守护进程

```bash
# 方式1：创建停止标记文件（推荐，优雅退出）
touch /tmp/trader_daemon.stop

# 方式2：直接杀进程
kill $(pgrep -f trader_daemon)

# 方式3：杀 tmux 会话
tmux kill-session -t trader
```

### 单次手动执行

```bash
cd /home/new/project && source venv/bin/activate && python run_trade.py
```

### 运行测试

```bash
# 全部测试
cd /home/new/project && source venv/bin/activate && python -m unittest discover tests -v

# 单个测试文件
python -m unittest tests/test_trading_mode.py -v
```

### 查看调度计划

启动日志中会打印调度器时间表：

| 任务 | 频率 |
|------|------|
| 技术指标更新 | 每 30s |
| LLM 完整交易流水线 | 每 180s |
| 价格卡死检查 | 每 30s |
| 仓位检查 + 状态同步 | 每 5s |
| 决策统计报告 | 每 1h |
| 影子账户快照 | 每 1h |
| 绩效分析报告 | 每 24h |

---

## 配置文件

`/home/new/project/.env`

| 变量 | 说明 |
|------|------|
| `TRADING_MODE` | shadow / demo / live |
| `TRADING_ACCOUNT_SCOPE` | 账户隔离标识 |
| `OKX_API_KEY` | OKX API Key |
| `OKX_SECRET_KEY` | OKX Secret |
| `OKX_PASSPHRASE` | OKX Passphrase |
| `DEEPSEEK_API_KEY` | DeepSeek API Key |
| `DEEPSEEK_MODEL` | deepseek-v4-pro |
| `LOG_LEVEL` | INFO / DEBUG / ERROR |

---

## 快速操作（一行命令）

```bash
# 启动（代码目录执行）
alias trade-start='cd /home/new/project && source venv/bin/activate && tmux new-session -d -s trader '\''python -u trader_daemon.py 2>&1 | tee /tmp/trader_daemon.log'\'

# 查看日志
alias trade-log='tmux capture-pane -t trader -p -S -20'

# 停止
alias trade-stop='touch /tmp/trader_daemon.stop'

# 状态
alias trade-ps='ps aux | grep trader_daemon | grep -v grep'
```

加到 `~/.zshrc` 后即可全目录使用。
