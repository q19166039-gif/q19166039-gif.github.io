---
layout: post
title: "OpenWrt 防火墙实战：关闭 SSH 端口转发与风险端口扫描"
date: 2026-07-19 10:00:00 +0800
excerpt: "记录 OpenWrt 关闭 SSH 端口转发、清理 nftables 残留规则并检查 WAN 暴露面的完整流程。"
categories: [network, security]
tags: [openwrt, firewall, nftables, ssh, security]
---

## 背景

路由器的 WAN 端口映射一直是内网服务暴露给公网的主要途径。最近检查一台 OpenWrt 路由器时发现，**Kali 工作站的 SSH (22 端口) 通过路由器端口映射暴露到了公网**——显然这是需要立即修复的安全风险。

本文将记录如何完整关闭端口转发、清理残留规则，以及如何系统性地扫描 OpenWrt 上还有哪些端口暴露在 WAN 侧。

---

## 第一步：诊断现状

首先登录路由器，查看当前的防火墙规则全貌：

```bash
# 查看 nftables 完整规则集
nft list ruleset

# 查看 iptables NAT 规则
iptables -t nat -L -n

# 查看 UCI 防火墙配置中的端口转发
uci show firewall | grep redirect

# 查看侦听端口
netstat -tlnp
```

通过以上命令可以清晰看到：
- **UCI redirect 规则**：WAN 端口 22 被 DNAT 转发到内网 Kali 的 22 端口
- **nftables dstnat 链**：两条 DNAT 规则（pppoe-wan 和 wan 接口）将入站 22 端口指向内网机器
- **nftables forward_wan 链**：一条放行规则，允许 WAN→内网的 22 端口流量

## 第二步：通过 UCI 删除端口转发

OpenWrt 的端口转发通常由 UCI 管理，优先通过 UCI 清理：

```bash
# 查看已有的 redirect 规则编号
uci show firewall | grep redirect

# 删除第 0 条 redirect 规则
uci delete firewall.@redirect[0]

# 提交更改
uci commit firewall

# 重载防火墙
fw4 reload
```

> **为什么先删 UCI？** UCI 是 OpenWrt 的声明式配置层，删除 UCI 条目后 `fw4 reload` 会自动重新生成 nftables 规则。但如果有手动添加的 nftables 规则，UCI 无法覆盖，需要手动清理。

## 第三步：手动清理残留的 nftables 规则

`fw4 reload` 后，如果 UCI 配置已经清理干净，nftables 中的 DNAT 规则也应该消失。但有些情况下（特别是之前通过 `nft` 命令直接添加的规则）会有残留，需要手动删除。

先查看规则的 handle 号（`-a` 参数显示 handle）：

```bash
# 查看带 handle 号的规则
nft -a list chain inet fw4 dstnat
nft -a list chain inet fw4 forward_wan
```

根据 handle 号逐一删除：

```bash
# 删除 dstnat 链中的 SSH DNAT 规则
nft delete rule inet fw4 dstnat handle 355

# 删除 forward_wan 链中的 SSH 放行规则
nft delete rule inet fw4 forward_wan handle 358
```

**验证清理结果：**

```bash
# 确认 dstnat 中不再有 dport 22 的规则
nft -a list chain inet fw4 dstnat | grep 'dport 22' || echo "已清除"

# 确认 forward_wan 中不再有 dport 22 的规则
nft -a list chain inet fw4 forward_wan | grep 'dport 22' || echo "已清除"
```

## 第四步：扫描风险端口

关闭 22 端口后，还需要系统性地检查 **WAN 侧还开放了哪些端口**。OpenWrt 的防火墙架构中，入站流量经过 `input_wan` 链（发往路由器自身）和 `forward_wan` 链（发往内网），加上 `dstnat` 链的端口映射。

### 检查路由器自身（input_wan）的放行规则

```bash
nft list chain inet fw4 input_wan
```

OpenWrt 默认配置中，`input_wan` 链默认策略为 **drop**，仅放行以下必要协议：

| 协议 | 端口 | 说明 | 是否必要 |
|:----:|:----:|:-----|:--------:|
| UDP | 68 | DHCP Renew | ✅ 必要 |
| ICMP | — | Ping（外网可 ping） | ⚠️ 可禁 |
| IGMP | — | 组播 | ✅ 必要 |
| UDP | 546 | DHCPv6 | ✅ 必要 |
| ICMPv6 | — | IPv6 ICMP | ✅ 必要 |

### 检查端口映射（dstnat）

```bash
# 查看所有 DNAT 映射
nft list chain inet fw4 dstnat | grep 'dnat' || echo "无 DNAT 映射"
```

### 检查转发规则（forward_wan）

```bash
nft list chain inet fw4 forward_wan
```

## 第五步：理解 OpenWrt 防火墙链结构

搞懂 nftables 的链模型对排查和加固非常重要。OpenWrt 21.02+ 使用 `fw4`（基于 nftables）替代了旧的 `fw3`（基于 iptables）。`fw4` 的核心表 `inet fw4` 包含以下关键链：

```
           ┌──────────────────┐
WAN ─────→│   input_wan      │──→ 路由器自身服务 (drop by default)
           └──────────────────┘
           │
           ↓
           ┌──────────────────┐
           │   forward_wan    │──→ 内网转发 (需要放行规则)
           └──────────────────┘
           │
           ↓
           ┌──────────────────┐
           │   dstnat         │──→ DNAT 端口映射 (prerouting)
           └──────────────────┘
```

- **input_wan**：控制 WAN 侧能否访问路由器自身的服务（SSH、Web 管理页面等）。默认 drop。
- **forward_wan**：控制 WAN 侧流量能否转发到内网机器。
- **dstnat**：端口映射，将 WAN 侧端口的流量 DNAT 到内网特定机器。

## 总结

完整的安全加固流程：

1. **诊断** → 查看 UCI 配置和 nftables 规则，识别暴露面
2. **UCI 清理** → `uci delete` + `uci commit` + `fw4 reload`
3. **nftables 兜底** → 如果 UCI 未覆盖，手动 `nft delete rule` 清理残留
4. **验证** → 确认所有暴露规则已清除
5. **风险扫描** → 系统性检查 input_wan / forward_wan / dstnat 链的放行规则

最终效果：**WAN 侧 SSH 完全关闭**，外网不再能通过路由器映射访问内网 SSH 服务。路由器自身除 DHCP/Ping 等必要协议外，无额外端口开放。

> **额外建议：** 如果对安全性要求更高，可以考虑在 `input_wan` 链中限制 ICMP echo（禁止外网 ping）以及限制 LAN 侧管理端口的访问来源范围。
