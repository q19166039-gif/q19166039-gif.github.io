---
title: "火山方舟 Coding Plan 使用备忘"
date: 2026-06-11 10:30:00 +0800
categories: [AI, 工具]
tags: [火山方舟, Coding Plan, API, 方舟]
---

最近在研究火山方舟的 Coding Plan，整理一下关键信息方便查阅。

## 调用地址

```
https://ark.cn-beijing.volces.com/api/coding/v3
```

详细配置参考：[方舟 API 文档](https://www.volcengine.com/docs/82379/2318283?lang=zh)

## 套餐限制

Coding Plan **不是按 token 计费**，而是按请求次数限制，维度包括：

- **5 小时**
- **周**
- **月**

具体限额看这里：[Coding Plan 限制说明](https://www.volcengine.com/docs/82379/2165245?lang=zh#628bdbe3)

## API Key 申请

直接访问方舟 API Key 管理页面：

> [https://console.volcengine.com/ark/region:ark+cn-beijing/apiKey](https://console.volcengine.com/ark/region:ark+cn-beijing/apiKey)

点击「创建 API Key」即可生成调用密钥。**注意妥善保管，不要泄露。**

## 用量查看

前往 Coding Plan 管理页面：

> [https://console.volcengine.com/ark/region:ark+cn-beijing/openManagement?LLM={}&OpenModelVisible=false&advancedActiveKey=subscribe](https://console.volcengine.com/ark/region:ark+cn-beijing/openManagement?LLM={}&OpenModelVisible=false&advancedActiveKey=subscribe)

---

## 快速汇总

| 项目 | 链接/值 |
|---|---|
| API 地址 | `https://ark.cn-beijing.volces.com/api/coding/v3` |
| API 文档 | [docs/82379/2318283](https://www.volcengine.com/docs/82379/2318283?lang=zh) |
| 套餐限制 | [docs/82379/2165245](https://www.volcengine.com/docs/82379/2165245?lang=zh#628bdbe3) |
| API Key | [console/ark/apiKey](https://console.volcengine.com/ark/region:ark+cn-beijing/apiKey) |
| 用量监控 | [console/ark/openManagement](https://console.volcengine.com/ark/region:ark+cn-beijing/openManagement?LLM={}&OpenModelVisible=false&advancedActiveKey=subscribe) |
