---
title: "Hermes-Agent 简要中文使用手册"
date: 2026-06-17
categories: [技术, Hermes-Agent]
tags: [AI, Hermes, 教程]
---

# Hermes-Agent 简要中文使用手册

> 澄清：Hermes-Agent 是 **Nous Research** 开发的开源AI代理项目，并非吴恩达发布。本文整理核心用法方便快速查阅。

---

## 什么是 Hermes-Agent

Hermes-Agent 是一个**自进化AI代理**，核心特点：
- ✅ **闭环学习**：从经验中自动创建技能，在使用中不断改进
- ✅ **跨平台**：支持 Telegram/Discord/Slack/CLI 多种入口
- ✅ **任意模型**：支持 OpenRouter/OpenAI/Anthropic/小米MiMo等300+模型，一键切换
- ✅ **定时任务**：内置cron调度，支持自动推送结果到消息平台
- ✅ **随处运行**：$5VPS就能跑，也支持Serverless按需运行
- ✅ **委派并行**：生成多个子代理同时处理任务

---

## 快速安装

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

安装完成后加载环境：
```bash
source ~/.bashrc  # 或 source ~/.zshrc
```

**支持环境**：Linux/macOS/WSL2/Android(Termux)，Windows需要WSL2。

---

## 常用命令

| 命令 | 作用 |
|------|------|
| `hermes` | 启动交互式CLI，开始对话 |
| `hermes model` | 选择LLM提供商和模型 |
| `hermes tools` | 配置启用的工具 |
| `hermes config set` | 设置单个配置项 |
| `hermes setup` | 运行完整设置向导（一次性配置） |
| `hermes gateway` | 启动消息网关（Telegram/Discord等） |
| `hermes update` | 更新到最新版本 |
| `hermes doctor` | 诊断配置问题 |
| `hermes claw migrate` | 从OpenClaw迁移配置 |

---

## 常用斜杠命令（CLI和消息平台通用）

| 命令 | 作用 |
|------|------|
| `/new` / `/reset` | 开始新对话 |
| `/model [provider:model]` | 更换模型 |
| `/personality [name]` | 设置人格 |
| `/retry` / `/undo` | 重试/撤销上一轮 |
| `/compress` | 压缩上下文 |
| `/usage` | 查看用量统计 |
| `/skills` | 浏览已安装技能 |
| `/stop` | 中断当前工作 |

---

## Nous Portal 一站式服务

如果你不想收集一堆API Key，可以使用Nous Portal：
一个订阅覆盖300+模型 + 网页搜索/图像生成/TTS等工具。

安装时直接：
```bash
hermes setup --portal
```

---

## 核心功能介绍

### 1. 技能系统
Hermes会在复杂任务完成后**自动创建技能**，下次遇到同类任务直接复用。
- 手动管理：`skill_manage` 工具可以创建/修改/删除技能
- 技能存储在 `~/.hermes/skills/`

### 2. 记忆系统
- **用户画像**：存储用户偏好和个人信息
- **持久记忆**：存储环境信息/最佳实践，跨会话保留
- **会话搜索**：支持全文搜索历史对话，找回之前的工作

### 3. 定时任务（Cron）
用自然语言创建定时自动化任务：
```python
# 示例：每天9点发送日报到微信
cronjob(
    action='create',
    schedule='0 9 * * *',
    prompt='整理今天的网络日志，生成简要日报发送到微信',
    deliver='weixin'
)
```

### 4. 消息网关
可以让Hermes运行在云端VPS，你通过微信/Telegram随时随地和它对话：
```bash
hermes gateway setup  # 配置平台
hermes gateway start  # 启动网关
```

支持平台：Telegram/Discord/Slack/WhatsApp/Signal/微信（通过社区桥接）/Home Assistant

---

## 配置文件

主配置文件位置：`~/.hermes/cli-config.yaml`

主要配置项：
- `model`: 默认模型提供商和型号
- `enabled_toolsets`: 启用的工具集
- `gateway`: 消息网关配置
- `security`: 安全设置（命令审批等）

---

## 常见问题

**Q: 可以使用本地模型吗？**  
A: 可以，只要提供OpenAI兼容的API端点即可配置。

**Q: 最低配置要求？**  
A: 只需1GB内存就能运行，$5/month的VPS足够。

**Q: 怎么从OpenClaw迁移？**  
A: 安装后执行 `hermes claw migrate` 自动迁移所有配置/记忆/技能。

---

## 官方资源

- 官方文档：https://hermes-agent.nousresearch.com/docs/
- GitHub：https://github.com/NousResearch/Hermes-Agent
- 技能中心：https://agentskills.io
- Discord社区：https://discord.gg/NousResearch

---

*本文整理于 Hermes-Agent 官方 README.zh-CN.md*
