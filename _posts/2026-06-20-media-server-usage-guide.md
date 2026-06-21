---
title: "家庭影音服务器使用指南：加片、下载、播放一条龙"
date: 2026-06-20
categories: [技术, 家庭服务器]
tags: [Jellyfin, qBittorrent, Prowlarr, Radarr, Sonarr, 使用指南]
---

## 一句话

打开 Radarr/Sonarr 搜片加入 → 自动下载 → Jellyfin 直接看。

## 服务地址

| 服务 | 地址 | 说明 |
|------|------|------|
| **Jellyfin** | `http://192.168.101.165:8096` | 播放器 |
| **Radarr** | `http://192.168.101.165:7878` | 电影管理 |
| **Sonarr** | `http://192.168.101.165:8989` | 剧集管理 |
| **Prowlarr** | `http://192.168.101.165:9696` | 索引器（不用管） |
| **qBittorrent** | `http://192.168.101.165:8080` | 下载器（不用管） |

**登录：** qBittorrent 用 `admin / zhouheart`，其他内网直连无需密码。

## 加电影（Radarr）

1. 浏览器打开 `http://192.168.101.165:7878`
2. 点顶部 **电影 → 添加新电影**
3. 搜索框输入电影名（中文英文都行）
4. 找到后点电影封面
5. 确认片名、年份，点 **添加电影**
6. 搞定，等着就行

Radarr 会自动通过 Prowlarr 搜索种子 → qBittorrent 下载 → 完成后自动重命名整理到 `/media/movies/` → Jellyfin 自动扫描识别。

## 加剧集（Sonarr）

1. 浏览器打开 `http://192.168.101.165:8989`
2. 点顶部 **系列 → 添加新系列**
3. 搜索剧名
4. 选好剧集，点 **添加系列**
5. 可以选要追的季数（默认全部）
6. 搞定

Sonarr 会自动追更，每出一集新的一集就自动下。

## 播放（Jellyfin）

1. 浏览器打开 `http://192.168.101.165:8096`
2. 点 **电影** 或 **剧集** 就能看到已入库的内容
3. 点封面直接播放
4. 支持手机、平板、电视（装 Jellyfin 客户端）

新加的电影/剧集下载完成后，Jellyfin 会自动扫描出现，不用手动刷新。

## 常见操作

### 查看下载进度
打开 qBittorrent `http://192.168.101.165:8080`，能看到正在下载的任务和速度。

### 手动触发 Jellyfin 扫库
Jellyfin → 控制台 → 媒体库 → 扫描所有媒体库（一般不需要，自动的）

### 查看 Radarr/Sonarr 下载状态
Radarr/Sonarr → 活动 → 队列，能看到正在下载和等待中的任务。

### 如果搜不到资源
Prowlarr 目前配了 7 个公开索引器（1337x, The Pirate Bay, YTS, LimeTorrents, TorrentGalaxy, EZTV, Nyaa），大部分电影剧集都能搜到。如果某个冷门资源搜不到，可以手动下载种子文件放到 qBittorrent 里下载，下载完移到对应目录就行。

## 存储结构

```
/media/
├── downloads/    ← 下载临时目录（不用管）
├── movies/       ← 电影库（Jellyfin 扫这里）
└── tv/           ← 剧集库（Jellyfin 扫这里）
```

## 一句话总结

**Radarr 加电影，Sonarr 加剧集，Jellyfin 看。** 其他都是自动的。
