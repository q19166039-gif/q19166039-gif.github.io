---
title: "Prowlarr 添加 Radarr 同步提示 URL 无效？直接写数据库搞定"
date: 2026-06-21 10:00:00 +0800
categories: [技术, 家庭服务器]
tags: [Prowlarr, Radarr, Sonarr, SQLite, 踩坑记录]
---

## 问题

在搭建 Jellyfin 全自动下载方案时，需要让 **Prowlarr**（索引器管理）同步索引到 **Radarr**（电影管理）和 **Sonarr**（剧集管理）。

Sonarr 通过 Prowlarr 的 Web UI 正常添加成功。但添加 Radarr 时，无论怎么填，Prowlarr 都报同一个错误：

> **Prowlarr URL is invalid, Radarr cannot connect to Prowlarr**

三个服务都跑在**同一台机器上**，网络连通性完全没问题。Sonarr 能正常添加，说明 Prowlarr 本身工作正常。

## 踩坑过程

### 尝试 1：换 URL 格式

一开始以为是 URL 格式问题，试了各种写法：

- `http://127.0.0.1:7878`
- `http://localhost:7878`
- `http://[内网IP]:7878`
- `http://[主机名]:7878`

全部报同样的错误。

### 尝试 2：检查 Radarr 自身连通性

确认 Radarr 本身正常运行：

```bash
curl http://127.0.0.1:7878/api/v3/system/status
```

返回正常，API 响应没问题。

### 尝试 3：检查 Prowlarr 日志

翻 Prowlarr 的日志，发现关键信息——Prowlarr 在添加应用时，**会反向验证 Radarr 能否连回 Prowlarr 自身**。如果 Radarr 连不回 Prowlarr 的 URL，就拒绝添加。

Sonarr 能成功是因为它的 API 验证逻辑更宽松，或者 Prowlarr 对 Sonarr 的验证方式不同。

## 解决方案：直接操作 SQLite 数据库

既然 API 验证过不去，那就绕过它——**直接把 Radarr 应用记录插入 Prowlarr 的 SQLite 数据库**。

### 步骤 1：找到数据库文件

Prowlarr 的数据库通常位于：

```bash
# 默认路径
/var/lib/prowlarr/prowlarr.db
```

### 步骤 2：查看现有应用结构

先看看 Sonarr 的记录长什么样，作为模板：

```bash
sqlite3 /var/lib/prowlarr/prowlarr.db ".schema Applications"
```

`Applications` 表的结构大致包含：

| 字段 | 说明 |
|------|------|
| `Id` | 自增 ID |
| `Name` | 应用名称（如 Radarr） |
| `Implementation` | 实现类名（如 Radarr） |
| `ConfigContract` | 配置契约名（如 RadarrSettings） |
| `Enable` | 是否启用（1=启用） |
| `SyncLevel` | 同步级别（1=仅添加, 2=添加并删除） |
| `Settings` | JSON 格式的配置 |

### 步骤 3：插入 Radarr 记录

参考 Sonarr 的 Settings JSON 格式，构造 Radarr 的配置：

```bash
sqlite3 /var/lib/prowlarr/prowlarr.db "
INSERT INTO Applications
  (Id, Name, Implementation, ConfigContract, Enable, SyncLevel, Settings)
VALUES
  (3, 'Radarr', 'Radarr', 'RadarrSettings', 1, 2,
   '{\"prowlarrUrl\":\"http://127.0.0.1:9696\",
     \"baseUrl\":\"http://127.0.0.1:7878\",
     \"apiKey\":\"<Radarr的API Key>\",
     \"syncCategories\":[2000,2010,2020,2030,2040,2050,2060]}');
```

**关键点：**
- `prowlarrUrl` 和 `baseUrl` 都用 `127.0.0.1`（不要用内网 IP）
- `syncCategories` 填入 Radarr 支持的分类 ID（2000-2060 是电影分类）
- `apiKey` 从 Radarr 的设置页面复制

### 步骤 4：重启 Prowlarr

插入后重启 Prowlarr 使配置生效：

```bash
systemctl restart prowlarr
```

### 步骤 5：验证

```bash
sqlite3 /var/lib/prowlarr/prowlarr.db "
SELECT Id, Name, Enable, SyncLevel FROM Applications;
"
```

预期输出：

```
2|Sonarr|1|1
3|Radarr|1|2
```

## 为什么 API 验证会失败？

事后分析，Prowlarr 的 API 验证逻辑是这样的：

1. 前端提交 Radarr 配置 → Prowlarr API 收到请求
2. Prowlarr 尝试用提交的 `baseUrl` 去访问 Radarr 的 API
3. 验证通过后，Prowlarr 告诉 Radarr "来，你连我试试"
4. Radarr 用收到的 `prowlarrUrl` 反向连接 Prowlarr
5. **如果 Radarr 连不回 Prowlarr → 拒绝**

问题出在第 4 步：如果填的是内网 IP，Radarr 可能因为某些网络配置（如防火墙、容器网络隔离）无法反向连接 Prowlarr。但用 `127.0.0.1` 就绕过了这个问题——因为所有服务都在同一台机器上。

**而直接写数据库则完全跳过了这个验证流程。**

## 总结

| 要点 | 说明 |
|------|------|
| **问题** | Prowlarr API 添加 Radarr 时报 "URL is invalid" |
| **根因** | Prowlarr 的反向连通性验证过于严格 |
| **解决方案** | 直接写 SQLite 数据库绕过 API 验证 |
| **关键配置** | 所有 URL 用 `127.0.0.1` 而不是内网 IP |
| **适用场景** | 所有服务运行在同一台机器上时 |

这个方法同样适用于其他 *arr 套件（Lidarr、Readarr 等）遇到类似问题时的排查思路——当 API 验证成为障碍时，数据库操作是一个有效的 bypass 手段。
