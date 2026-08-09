---
layout: single
title: "如何验证 API 中转站背后的真实模型：一个 model 字段骗局"
date: 2026-08-09 10:06:00 +0800
excerpt: "第三方 LLM API 中转站返回的 model 字段可能只是网关伪装的路由别名。实测：配置名为 deepseek-v4-flash，实际后端却是 Gemini 2.5 Pro。附可复制的模型身份验证脚本。"
categories: [技术, AI]
tags: [LLM, API, 踩坑, 调试]
---

## 踩坑背景

为了节省成本或聚合多家模型，很多人会通过第三方 API 中转站（网关）调用 LLM。配置里写的是 `deepseek-v4-flash`，调用时 API 返回的 `model` 字段也是 `deepseek-v4-flash`，一切看起来都正常。

直到有一天你发现：**回答风格、思考方式、能力边界都和官方 DeepSeek 对不上**。

我就是在做连通性检查时，顺手让模型"自报家门"，结果发现了一个尴尬的事实。

## 排查过程

### 第一步：查配置，确认名义模型

```python
import yaml
cfg = yaml.safe_load(open('config.yaml'))
m = cfg['model']
print(m['default'])    # deepseek-v4-flash
print(m['base_url'])   # 某个第三方网关地址
```

配置没问题，名义上就是 `deepseek-v4-flash`。

### 第二步：调用 API，看返回的 model 字段

```python
import json, urllib.request

def chat(base, key, model):
    req = urllib.request.Request(base + '/chat/completions', method='POST')
    req.add_header('Authorization', 'Bearer ' + key)
    req.add_header('Content-Type', 'application/json')
    payload = {
        'model': model,
        'messages': [{'role': 'user', 'content': '你好'}],
        'max_tokens': 10,
        'stream': False,
    }
    with urllib.request.urlopen(req, data=json.dumps(payload).encode(), timeout=30) as r:
        return json.loads(r.read().decode())

d = chat(base, key, 'deepseek-v4-flash')
print(d.get('model'))   # deepseek-v4-flash —— 和配置一致
```

返回的 `model` 字段确实是 `deepseek-v4-flash`。到这里为止，一切正常。

### 第三步：让模型自报身份（关键一步）

API 返回的 `model` 字段是**网关填的**，不是模型自己说的。让模型用自己的真实身份回答一次：

```python
payload = {
    'model': 'deepseek-v4-flash',
    'messages': [
        {'role': 'system', 'content': '只回答一行，格式严格为 [模型名]-[厂商]-[版本]，例如 Gemini-Google-2.5-pro-exp-03-25。如实报告你的底层真实模型，不要说配置里的名字。'},
        {'role': 'user', 'content': '你的真实底层模型是什么？只输出一行 [模型名]-[厂商]-[版本]。'},
    ],
    'max_tokens': 100,
    'stream': False,
}
```

实测结果：

```
API 返回 model 字段: deepseek-v4-flash
模型自报身份:        gemini-2.5-pro-exp-03-25 (Gemini 2.5 Pro Experimental)
usage: 114 prompt + 74 completion (含 51 reasoning tokens)
```

**配置写的是 deepseek-v4-flash，API 返回的也是 deepseek-v4-flash，但实际跑的是 Gemini 2.5 Pro Experimental。** 中转站存在模型偷换——`model` 字段只是网关的路由别名，不代表真实后端。

### 第四步：批量验证所有模型

只测一个模型可能有偶然性。把网关暴露的所有模型 ID 逐个调用一遍，让每个模型自报身份，就能看出网关的"路由表"：

```python
import yaml, json, urllib.request, sys

cfg = yaml.safe_load(open('/home/user/.hermes/config.yaml'))
key = cfg['model']['api_key']      # 注意：key 不要打印出来
base = cfg['model']['base_url']

def get_models():
    req = urllib.request.Request(base + '/models')
    req.add_header('Authorization', 'Bearer ' + key)
    with urllib.request.urlopen(req, timeout=30) as r:
        d = json.loads(r.read().decode())
    return [m['id'] for m in d.get('data', [])]

def chat(model):
    req = urllib.request.Request(base + '/chat/completions', method='POST')
    req.add_header('Authorization', 'Bearer ' + key)
    req.add_header('Content-Type', 'application/json')
    payload = {
        'model': model,
        'messages': [
            {'role': 'system', 'content': '只回答一行，格式严格为 [模型名]-[厂商]-[版本]。如实报告你的底层真实模型。'},
            {'role': 'user', 'content': '你的真实底层模型是什么？'},
        ],
        'max_tokens': 60,
        'stream': False,
    }
    try:
        with urllib.request.urlopen(req, data=json.dumps(payload).encode(), timeout=45) as r:
            d = json.loads(r.read().decode())
        return d.get('model'), d['choices'][0]['message']['content'].strip()
    except Exception as e:
        return 'ERR', str(e)[:80]

for m in get_models():
    api_model, self_report = chat(m)
    print(f'{m:20s} | API字段={api_model:20s} | 自报={self_report}')
```

几个注意点：

- **max_tokens 别设太小**。思考型模型（带 reasoning）会把额度吃光，导致只输出思考过程、答不出身份。失败后要用更大的 `max_tokens` 重试。
- 有的模型会拒绝回答身份，可以要求它用一行 JSON 输出 `{"model": ..., "vendor": ..., "version": ...}`。
- 如果响应里出现 `reasoning_content` 字段，说明后端是思考型模型，这本身就是一条身份线索。

## 结论与建议

1. **中转站的 `model` 字段不可信**，它只是网关的路由别名。网关完全可以把任意请求路由到任意后端，再原样返回你传入的模型名。
2. **让模型自报身份是最直接的验证手段**，零成本、可脚本化，适合定期抽查。
3. **关键任务建议直接用官方 API**。中转站除了偷换模型，还可能记录你的请求内容，成本和隐私都要权衡。
4. 如果发现偷换，可以拿验证结果去和供应商对质——多数情况是网关"缺货降级"但没告诉你，属于服务欺诈。

## 延伸：usage 和 reasoning 特征

除了自报身份，响应元数据也能辅助判断：

- `usage.completion_tokens` 远大于 `prompt_tokens` 且出现 `reasoning_content` → 思考型模型（如 DeepSeek-R1 系、Gemini 系）。
- 非思考模型通常 prompt 占大头、无 reasoning 字段。
- 响应速度突变（同样长度的问题，耗时差几倍）也是后端被切换的信号。
