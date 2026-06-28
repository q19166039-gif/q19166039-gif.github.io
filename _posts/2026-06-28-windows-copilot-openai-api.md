---
title: "免费白嫖 GPT-4？Windows Copilot API 逆向工程项目体验"
date: 2026-06-28 10:00:00 +0800
categories: [技术, 开源项目]
tags: [Copilot, OpenAI, API, 逆向工程, LLM, Python]
---

## 背景

前两天在 GitHub 上看到一个有意思的项目 —— [Windows-Copilot-API](https://github.com/sums001/Windows-Copilot-API)，把 Microsoft Copilot 的免费网页版逆向工程为一个兼容 OpenAI API 格式的本地服务。简单说就是：**免 API Key、免付费，用自己的 Microsoft 账号就能调 GPT-4 级别的模型**。

项目上线不到一周就 600+ Star、216 Fork，说明需求确实不小。

## 它干了什么？

原理不复杂：项目用 Playwright 打开浏览器让你登录一次 Microsoft 账号，拿到 Cloudflare clearance 和 Copilot 会话 token，然后通过 WebSocket 协议和 copilot.microsoft.com 通信，把请求/响应翻译成 OpenAI 的 `/v1/chat/completions` 格式。

工作流：

```
你的应用 → localhost:8000/v1 → Copilot WebSocket → Microsoft 服务器
```

## 两种使用方式

### 方式一：Python 库直接调用

```python
from copilot import CopilotClient

client = CopilotClient()

# 单轮对话
reply = client.chat("用一句话解释量子计算")
print(reply.text)

# 多轮对话（传 conversation_id）
reply2 = client.chat("再说详细点", reply.conversation_id)

# 流式输出
for chunk in client.stream("讲个笑话"):
    print(chunk, end="", flush=True)
```

### 方式二：启动 OpenAI 兼容服务器

```bash
# 安装依赖
pip install -r requirements.txt
playwright install chromium

# 登录一次（会打开浏览器）
python -m copilot login

# 启动服务
python app.py
# → 监听 http://127.0.0.1:8000
```

然后任何 OpenAI SDK 都能直接用：

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="unused"  # SDK 要求传但不校验
)

resp = client.chat.completions.create(
    model="copilot",
    messages=[{"role": "user", "content": "你好！"}]
)
print(resp.choices[0].message.content)
```

甚至 curl 也能调：

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "你好！"}]}'
```

## 踩坑记录

### 1. Cloudflare 验证是个门槛

Copilot 的聊天接口在 Cloudflare 后面，需要 `cf_clearance` cookie。项目在登录时自动处理了，但 clearance 有效期大概半小时就过期。

- **有图形界面的机器**：项目会自动弹出浏览器让你点验证，点完自动继续
- **纯命令行的 VPS/服务器**：没法弹浏览器，clearance 过期后服务端会返回 503，需要手动在本地登录一次再传 session 文件上去

### 2. 并发限制

Copilot 的 WebSocket 不支持同一个账号同时发起多个对话。所以服务端用了线程锁（`threading.Lock()`）串行化请求：

| 并发数 | 结果 | 总耗时 |
|-------|------|--------|
| 1 | ✅ 全部成功 | 3.7s |
| 2 | ✅ 全部成功 | 4.6s |
| 4 | ✅ 全部成功 | 8.3s |
| 8 | ❌ 有 502 失败 | 13.3s |

**结论**：这适合个人使用，不适合高并发场景。同时发太多请求 Copilot 会返回 502。

### 3. 内置限流

项目默认限速 12 RPM（每分钟请求数），burst 4。超过了会返回 429 + Retry-After。可以用环境变量调整：

```bash
RATE_LIMIT_RPM=20 RATE_LIMIT_BURST=5 python app.py
```

### 4. 上下文窗口有限

Consumer 版 Copilot 的上下文窗口大概只有 4K 字符，所以传超长 system prompt 或多轮对话会被截断。项目实现了一种按需截断策略：只保留最后一条用户消息，system prompt 太长时从前面砍。

## Docker 部署

项目提供了 Dockerfile 和 docker-compose.yml，但有个坑：**Docker 容器里没法弹浏览器完成 Cloudflare 验证**，所以需要先在宿主机上跑 `python -m copilot login` 登录一次，session 目录绑定挂载到容器里：

```yaml
volumes:
  - ./session:/app/session
```

## 性能基准测试

项目作者用 GPQA Diamond（198 道研究生级别题目）测了一下，得分 **40.9%**，属于 GPT-4 家族水平，不是 o1/o3 推理模型级别的。日常对话、写代码、翻译完全够用。

## 值得用吗？

**适合场景：**
- 想白嫖免费 LLM API 的个人开发者
- 需要本地测试 OpenAI 兼容接口但不想花钱
- 网络环境能直连 Copilot（不需要额外翻墙）

**不适合场景：**
- 生产环境/高并发
- 需要长上下文窗口
- 需要 o1/o3 级别推理能力
- 服务器在数据中心（Cloudflare 验证很难过）

## 总结

这个项目思路挺巧的，把免费的 Copilot 网页服务包装成了标准的 OpenAI API。对于个人开发者来说，省下了每个月几十刀的 API 费用，用来跑跑脚本、做个聊天机器人完全够用。但不是银弹 —— 并发、上下文、Cloudflare 验证都是硬限制。

项目 MIT 协议开源，代码量不大（核心就几个 Python 文件），适合读一读学习逆向工程 WebSocket 协议的思路。
