---
title: "OpenWrt + sing-box 1.12 透明代理：踩坑记录与安全部署方法论"
date: 2026-07-05 10:00:00 +0800
categories: [技术, 网络]
tags: [OpenWrt, sing-box, 透明代理, 网络配置, 踩坑记录]
---

## 背景

家里的 OpenWrt 路由器上跑了 sing-box 做透明代理，某天突然发现代理"空转"——进程活着、端口听着，但流量根本不走代理。这就引出了将近两小时的排查和修复过程，记录一下踩的坑和总结出的**安全部署方法论**。

## 问题现象

- `ps` 能看到 sing-box 进程在跑
- `netstat` 确认 `0.0.0.0:12345` 在监听
- `sing-box check` 配置检查通过（只有弃用警告）
- 但路由器自身 `curl ifconfig.me` 返回的是**本地宽带公网 IP**，不是代理出口
- `nft list ruleset` 和 `iptables -t nat -S` 都看不到任何透明代理规则

**结论：sing-box 活了，但没有任何流量被导进去。**

## 坑一：sing-box 1.12 的两个破坏性变更

如果是从旧版本迁移过来的配置，sing-box 1.12 有两个坑会让你启动失败：

### 1. `redir` → `redirect` 重命名

1.12 之前：

```json
{ "type": "redir", "tag": "redir-in", "listen_port": 12345 }
```

1.12 之后必须写成：

```json
{ "type": "redirect", "tag": "redirect-in", "listen_port": 12345 }
```

不改的话报错：`FATAL: unknown inbound type: redir`

### 2. `type: dns` inbound 被移除

旧配置里专门有一个 DNS inbound 来处理 DNS 流量，1.12 完全移除了它。DNS 解析直接在 `dns` 配置段处理即可，不再需要单独的 inbound。

同时新增了一个**强制要求**：

```json
"route": {
  "default_mark": 1
}
```

这个 `default_mark: 1` 是防回环的关键——不加的话，路由器自身发起的所有新 TCP 连接都会卡死（下面会细说）。

## 坑二：规则集下载死锁（经典坑）

首次启动时，如果配置了远程规则集（如 `geosite-cn`、`geoip-cn`）：

```json
"rule_set": [
  {
    "tag": "geosite-cn",
    "type": "remote",
    "url": "https://cdn.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@sing/geo/geosite/geolocation-cn.srs",
    "download_detour": "proxy"
  }
]
```

你会遇到死锁：代理未通 → DNS 解析走代理 → DNS 不通 → 规则集下不来 → 代理永远不通。

**解决方案**：首次启动时把 `download_detour` 设为 `"direct"`，让规则集走直连下载。等代理通了之后再改回 `"proxy"`。

## 坑三：UCI 默认 enabled=0

OpenWrt 的 opkg 包安装后，`/etc/config/sing-box` 中 `option enabled '0'`（默认禁用）。这意味着：

```bash
/etc/init.d/sing-box start  # ✅ 返回 success，但进程根本不会启动！
```

**必须先执行：**

```bash
uci set sing-box.main.enabled='1'
uci set sing-box.main.user='root'   # TPROXY/TUN 需要 root 权限
uci commit sing-box
/etc/init.d/sing-box start
```

## 安全部署方法论（最值得分享的部分）

经过这次踩坑，我总结了一套**增量验证流程**，可以避免「加完规则 → 全家断网 → SSH 被锁在外面」的惨案。

### Phase 0：先验证代理出站本身

**不要**一上来就加 iptables/nftables 规则。先临时加一个 SOCKS5 入站，验证出站代理本身是否能通：

1. 准备一份**无规则集、无透明代理**的最小配置，只保留 SOCKS5 inbound + 出站 outbound
2. 传输到路由器，前台启动（方便看日志）
3. 用 `curl -x socks5h://127.0.0.1:10808 ifconfig.me` 验证出口 IP
4. 验证通过后停止测试进程，恢复原配置

这一步确认的是：**代理服务器本身可用，配置无误**。

### Phase 1：加 PREROUTING 链（仅 LAN 设备）

SOCKS 测试通过后，再加 iptables REDIRECT 规则，但**先只影响 LAN 设备流量**：

```bash
iptables -t nat -N SINGBOX_PRE
iptables -t nat -A SINGBOX_PRE -d <路由器IP> -j RETURN  # 放行本机
iptables -t nat -A SINGBOX_PRE -p tcp --dport 80 -j REDIRECT --to-port 12345
iptables -t nat -A SINGBOX_PRE -p tcp --dport 443 -j REDIRECT --to-port 12345
iptables -t nat -A PREROUTING -j SINGBOX_PRE
```

此时路由器的 SSH 仍然直连（不受影响），即使代理挂了你还能 SSH 进去修。

### Phase 2：加 OUTPUT 链（路由器自身流量）⚠️ 最危险的一步

**必须**先确认 `route.default_mark: 1` 配好了，再加 OUTPUT 链：

```bash
iptables -t nat -N SINGBOX_OUT
iptables -t nat -A SINGBOX_OUT -m mark --mark 1 -j RETURN  # 防回环
iptables -t nat -A SINGBOX_OUT -o lo -j RETURN
iptables -t nat -A SINGBOX_OUT -d <路由器IP> -j RETURN
iptables -t nat -A SINGBOX_OUT -p tcp --dport 80 -j REDIRECT --to-port 12345
iptables -t nat -A SINGBOX_OUT -p tcp --dport 443 -j REDIRECT --to-port 12345
iptables -t nat -A OUTPUT -j SINGBOX_OUT
```

**`-m mark --mark 1 -j RETURN` 这条是命根子。** 它的作用是：如果某个包已经被 sing-box 打上了 mark=1 标记，就放行它，不让它再被 REDIRECT 回 sing-box。不加这条会死循环——curl 请求 → OUTPUT REDIRECT 到 sing-box → sing-box 出站连接又触发 OUTPUT → 又 REDIRECT → ... → 连接耗尽。症状：路由器自身所有新 TCP 连接卡死，但 LAN 设备可能正常。

### 恢复流程

如果不慎断网：

```bash
# 从 PVE 控制台或带外管理口进入路由器
nft delete table inet singbox 2>/dev/null
ip rule del fwmark 1 table 100 2>/dev/null
/etc/init.d/sing-box stop
```

**预防**：修改 nftables 前先通过 PVE VNC 打开路由器的控制台窗口，万一 SSH 断了还有后门。

## 总结

| 要点 | 说明 |
|------|------|
| 1.12 迁移 | `redir` → `redirect`，移除 `dns` inbound |
| 必须加 `default_mark: 1` | 不加 OUTPUT 链会死循环 |
| 首次规则集下载 | 用 `"direct"` 绕过死锁 |
| UCI 默认禁用 | 装完记得 `uci set sing-box.main.enabled=1` |
| 增量部署 | SOCKS 验证 → PREROUTING → OUTPUT，逐阶段推进 |
| 防 SSH 被锁 | 改防火墙前先打开带外控制台 |

这套方法论不仅适用于 sing-box，任何在路由器上做透明代理/流量劫持的场景都通用。先验出站、再劫 LAN、最后劫本机——每一步验证通过再走下一步。
