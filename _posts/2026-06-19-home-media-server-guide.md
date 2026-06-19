---
title: "家庭影音服务器搭建指南：Jellyfin + *arr 全自动下载方案"
date: 2026-06-19
categories: [技术, 家庭服务器]
tags: [Jellyfin, qBittorrent, Prowlarr, Radarr, Sonarr, 影音, PVE, LXC]
---

## 概述

家庭影音服务器的核心需求：**存片 → 搜片 → 下载 → 整理 → 播放**，全自动闭环。

本文记录基于 PVE LXC 容器搭建的完整方案，使用 Jellyfin 作为媒体服务器，配合 *arr 全家桶实现电影/剧集自动搜索、下载、整理。

## 架构总览

```
你搜片/添加想看
    ↓
Radarr(电影) / Sonarr(剧集)   ← 自动搜片、追更
    ↓
Prowlarr(索引器)              ← 聚合多个种子站
    ↓
qBittorrent(下载器)           ← 自动下载
    ↓
自动重命名、整理到媒体目录
    ↓
Jellyfin 自动扫描 → 直接播放
```

## 环境

| 项目 | 值 |
|------|------|
| 宿主机 | PVE (192.168.101.88) |
| Jellyfin LXC IP | 192.168.101.165 |
| 系统 | Debian 12, 2核/4GB |
| 存储 | 2.5T 挂载到 `/media` |
| 容器类型 | 非特权 (unprivileged) |

## 组件清单

| 组件 | 用途 | 端口 | 状态 |
|------|------|------|------|
| **Jellyfin** | 媒体服务器，流媒体播放 | `:8096` | ✅ |
| **qBittorrent EE** | BT 下载客户端，Web UI 管理 | `:8080` | ✅ |
| **Prowlarr** | 索引器管理器，聚合 tracker 搜索 | `:9696` | ✅ |
| **Radarr** | 电影自动管理 | `:7878` | ✅ |
| **Sonarr** | 剧集自动管理 | `:8989` | ✅ |

## 安装记录

### 基础准备

```bash
# 容器内安装基础工具
apt update && apt install -y wget curl sqlite3 unzip

# 创建 media 用户和目录结构
useradd -r -m -d /var/lib/media -s /bin/false media
mkdir -p /media/{downloads,movies,tv,music,photos}
chown -R media:media /media
```

### 安装 qBittorrent Enhanced Edition（静态编译版）

```bash
cd /opt
wget "https://github.com/c0re100/qBittorrent-Enhanced-Edition/releases/download/release-5.2.1.10/qbittorrent-enhanced-nox_x86_64-linux-musl_static.zip" -O qbittorrent.zip
unzip -o qbittorrent.zip -d qbittorrent
chmod +x qbittorrent/qbittorrent-nox
chown -R media:media qbittorrent
```

### 安装 Prowlarr

```bash
cd /opt
wget "https://prowlarr.servarr.com/v1/update/develop/updatefile?os=linux&runtime=netcore&arch=x64" -O prowlarr_update.json
# 从 JSON 中提取真实下载 URL
PROWLARR_URL=$(grep -oP 'https://github.com/Prowlarr/Prowlarr/releases/download/v[^"]+/Prowlarr.develop.[0-9.]+.linux-core-x64.tar.gz' prowlarr_update.json | head -1)
wget "$PROWLARR_URL" -O prowlarr.tar.gz
mkdir -p Prowlarr && tar xf prowlarr.tar.gz -C Prowlarr
chown -R media:media Prowlarr
```

### 安装 Radarr

```bash
cd /opt
wget "https://radarr.servarr.com/v1/update/develop/updatefile?os=linux&runtime=netcore&arch=x64" -O radarr_update.json
RADARR_URL=$(grep -oP 'https://github.com/Radarr/Radarr/releases/download/v[^"]+/Radarr.develop.[0-9.]+.linux-core-x64.tar.gz' radarr_update.json | head -1)
wget "$RADARR_URL" -O radarr.tar.gz
mkdir -p Radarr && tar xf radarr.tar.gz -C Radarr
chown -R media:media Radarr
```

### 安装 Sonarr

```bash
cd /opt
wget "https://sonarr.servarr.com/v1/update/develop/updatefile?os=linux&runtime=netcore&arch=x64" -O sonarr_update.json
SONARR_URL=$(grep -oP 'https://github.com/Sonarr/Sonarr/releases/download/v[^"]+/Sonarr.develop.[0-9.]+.linux-core-x64.tar.gz' sonarr_update.json | head -1)
wget "$SONARR_URL" -O sonarr.tar.gz
mkdir -p Sonarr && tar xf sonarr.tar.gz -C Sonarr
chown -R media:media Sonarr
```

> **注意**：*arr 的 tar.gz 包解压后会有三层嵌套（如 `/opt/Prowlarr/Prowlarr/Prowlarr`），systemd 服务文件的 `ExecStart` 路径要写对。

### Systemd 服务配置

创建四个服务文件到 `/etc/systemd/system/`：

**qbittorrent.service**
```ini
[Unit]
Description=qBittorrent Enhanced Edition (nox)
After=network.target

[Service]
User=media
Group=media
ExecStart=/opt/qbittorrent/qbittorrent-nox --webui-port=8080
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

**prowlarr.service**
```ini
[Unit]
Description=Prowlarr
After=network.target

[Service]
User=media
Group=media
ExecStart=/opt/Prowlarr/Prowlarr/Prowlarr -nobrowser -data=/var/lib/prowlarr
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**radarr.service** 和 **sonarr.service** 同理，注意二进制路径的三层嵌套。

```bash
# 启用并启动
systemctl daemon-reload
systemctl enable qbittorrent prowlarr radarr sonarr
systemctl start qbittorrent prowlarr radarr sonarr
```

## 非特权容器权限注意事项

PVE 非特权 LXC 容器中，容器内 uid/gid 在宿主机上会被映射（通常 +100000）。例如容器内 `media` 用户 uid=999，在宿主机上是 uid=100999。

如果遇到 `Access to the path '/var/lib/prowlarr/Sentry' is denied` 错误，从宿主机修复：

```bash
# 宿主机上执行
chown -R 100999:100996 /var/lib/lxc/101/rootfs/var/lib/prowlarr
chown -R 100999:100996 /var/lib/lxc/101/rootfs/var/lib/radarr
chown -R 100999:100996 /var/lib/lxc/101/rootfs/var/lib/sonarr
```

## 运维命令速查

### 服务管理

```bash
# 查看状态
systemctl status qbittorrent prowlarr radarr sonarr

# 重启
systemctl restart qbittorrent prowlarr radarr sonarr

# 查看日志
journalctl -u prowlarr --no-pager -n 20
journalctl -u radarr --no-pager -n 20
journalctl -u sonarr --no-pager -n 20
```

### 从 PVE 宿主机操作容器

```bash
# 执行命令
pct exec 101 -- systemctl status prowlarr

# 查看进程
pct exec 101 -- ps aux | grep -E "Prowlarr|Radarr|Sonarr|qbittorrent" | grep -v grep

# 查看端口
pct exec 101 -- ss -tlnp | grep -E "8080|9696|7878|8989"

# 推送文件到容器
pct push 101 /宿主机/文件 /容器内/路径

# 从容器拉取文件
pct pull 101 /容器内/文件 /宿主机/路径
```

### 直接 nsenter 进容器

```bash
# 获取容器 PID
CTPID=$(cat /var/lib/lxc/101/init)

# 进入容器命名空间执行命令
nsenter -t $CTPID -m -u -i -n -p -- ps aux
nsenter -t $CTPID -m -u -i -n -p -- ss -tlnp
```

## Web UI 配置步骤

### 1. qBittorrent

访问 `http://192.168.101.165:8080`
- 默认用户名：`admin`
- 默认密码：`adminadmin`
- 建议：设置 → Web UI → 修改密码

### 2. Prowlarr

访问 `http://192.168.101.165:9696`
- Settings → Indexers → 添加公开 tracker（推荐）：
  - 1337x
  - TorrentGalaxy
  - The Pirate Bay
  - YTS
  - EZTV
- Settings → Download Clients → 添加 qBittorrent：
  - Host: 127.0.0.1
  - Port: 8080
  - Username: admin
  - Password: adminadmin

### 3. Radarr

访问 `http://192.168.101.165:7878`
- Settings → Media Management：
  - 勾选 "Rename Movies"（自动重命名）
  - Movies Folder: `/media/movies`
- Settings → Download Clients → 添加 qBittorrent
- Settings → Indexers → 添加 Prowlarr（点 Prowlarr 按钮自动同步）
- Movies → Add New → 搜索电影加入

### 4. Sonarr

访问 `http://192.168.101.165:8989`
- Settings → Media Management：
  - 勾选 "Rename Episodes"
  - Series Folder: `/media/tv`
- Settings → Download Clients → 添加 qBittorrent
- Settings → Indexers → 添加 Prowlarr
- Series → Add New → 搜索剧集加入

### 5. Jellyfin

访问 `http://192.168.101.165:8096`
- 控制台 → 媒体库 → 添加媒体库：
  - 电影 → 文件夹 `/media/movies`
  - 剧集 → 文件夹 `/media/tv`
- 配置元数据刮削器（默认已配置）

## 工作流程

1. 你在 **Radarr** 添加想看的电影，或在 **Sonarr** 添加想追的剧
2. *arr 通过 **Prowlarr** 自动搜索各 tracker
3. 找到资源后推给 **qBittorrent** 下载
4. 下载完成 → 自动校验 → 自动重命名 → 整理到 `/media/movies/` 或 `/media/tv/`
5. **Jellyfin** 自动扫描到新内容，直接开播

## 存储结构

```
/media/
├── downloads/     ← qBittorrent 下载临时目录
├── movies/        ← 电影库（Jellyfin 扫描）
├── tv/            ← 剧集库（Jellyfin 扫描）
├── music/         ← 音乐
└── photos/        ← 照片
```

## 常见问题

### Q: *arr 启动报 "Is a directory"
解压 tar.gz 后目录结构是三层嵌套，systemd 的 `ExecStart` 路径要写完整的三层路径。

### Q: *arr 报 "Access to the path ... is denied"
非特权容器的 uid 映射问题。从 PVE 宿主机用正确的映射 uid 执行 `chown`。

### Q: qBittorrent 下载慢
- 检查 tracker 是否有效
- 在 qBittorrent 设置中添加更多 tracker
- 检查是否开启了 DHT、PEX、LSD

### Q: Jellyfin 不自动扫描新内容
- 检查媒体库设置中的 "实时监控" 是否开启
- 手动触发扫描：控制台 → 媒体库 → 扫描所有媒体库
