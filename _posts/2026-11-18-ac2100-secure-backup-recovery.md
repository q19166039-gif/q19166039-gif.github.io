---
title: "AC2100：影音绕行、健康检查与加密备份应急恢复"
date: 2026-11-18 01:55:00 +0800
layout: single
categories: [家庭网络]
tags: [OpenWrt, AC2100, sing-box, 备份]
---

本次为 Redmi AC2100 恢复影音服务器绕行、sing-box 健康检查和内存保护，并生成加密的配置备份。**没有升级固件、重启路由器、修改 DNS 或公开发布明文配置。**

## 备份下载与保管

[下载加密配置备份](/assets/files/ac2100-config-20261118.tar.gz.gpg)

文件 SHA-256：

```text
516931900887089de5d6d2ce8dde15fec39e49e2b36f427283d1499d07a7fb6c
```

加密使用 GnuPG AES256，已实际解密并与原始归档逐字节比对。解密口令不在网站或 Git 中；持有人须将口令另存离线密码管理器或安全 U 盘，不能只保留在同一台 Kali 上。没有口令不能恢复。

这是 **OpenWrt 配置备份，不是整机闪存镜像**。不含引导器、校准分区、完整固件、已安装软件包的二进制或家庭影音数据。恢复基线为 OpenWrt 25.12.6 / kernel 6.12.112 / sing-box 1.13.21；设备必须是相同型号，跨版本要先检查配置兼容性，不要盲目刷写。

## 为什么 CPU 会忙

巡检时 WAN 下载约 196–257 Mbps，CPU busy 约 77–83%，网络软中断约 28–29%。另一次存活连接流量差分中，局域网 .109 约 328 Mbps、影音 .165 仅约 0.47 Mbps，表明当时主要流量不是影音 BT。各采样并非同一时间，不能直接相加或当因果试验。

同时，影音绕行缺失，使 BT 的公网 IPv4 TCP 先进入 sing-box 再 direct 出站，每分钟约五六十条超时/拒绝日志。最终 direct 不等于绕过进程：额外的 socket、协议处理和垃圾回收会争抢资源，但不能据此断言所有断续都由 BT 引起。

## 本次恢复的机制

1. 在实际 `inet fw4 codex_singbox_prerouting` 链首，对 `192.168.1.165` 增加 `counter return`，早于 redirect。持久文件为 `/etc/nftables.d/20-singbox-redirect.nft`。仅绕过透明 TCP，防火墙和正常 NAT 不变；IPv6本就不匹配这条IPv4规则。整个影音主机的公网TCP直连，海外索引站可能受直连限制，后续应按需拆分服务，而不是把BT重新送进代理。
2. 包装器 `/usr/libexec/sing-box-protected` 设置 `GOMEMLIMIT=52428800`、`GOGC=50`、`oom_score_adj=500`，由 init 调用。50 MiB 是 Go 软限额，不保证 RSS 永远小于它；正的 OOM 优先级是内存紧张时优先牺牲代理，而非保证整机绝不 OOM。GOGC未照搬旧值20，避免过于频繁回收进一步加重CPU。
3. `/root/sing-box-healthcheck.sh` 每10分钟检查进程、三个监听端口、本地5353 DNS及两个HTTP候选探测。连续两次失败、距上次至少30分钟且当天不足3次，配置校验通过才重启sing-box；不连带重启dnsmasq，不按固定6小时无条件重启。HTTP成功不等于真实模型长连接稳定，仍需客户端验证。
4. 关键脚本、init、规则和cron已加入升级保留清单，`sysupgrade -l` 验证包含。软件包升级可能另行覆盖 init，升级后仍须验收，不把保留清单当永久保证。

## 人工恢复：还能访问管理页面

先用网线接 LAN 口，进入 `192.168.1.1`，保留一份故障现场备份；确认设备型号和软件版本，不直接点“升级固件”。

在自己的电脑上安装 GnuPG，下载上述文件并核对哈希，然后解密（交互输入口令，别写到命令行历史）：

```sh
sha256sum ac2100-config-20261118.tar.gz.gpg
gpg --output ac2100-config-20261118.tar.gz --decrypt ac2100-config-20261118.tar.gz.gpg
tar -tzf ac2100-config-20261118.tar.gz
```

进入 LuCI“系统 → 备份/升级”，选择**恢复配置备份**，上传解密的 `.tar.gz`，不是 `.gpg`，也不是当成固件刷写。根据界面提示重启。恢复会覆盖网络、Wi-Fi、代理和管理配置，电脑可能短暂断线；使用恢复后的原管理凭据，本文不提供密码。

## 人工恢复：SSH可用、LuCI不可用

先解密，再上传备份。较新OpenSSH默认SFTP，但该路由器可能没有SFTP服务，可用旧式SCP：

```sh
scp -O ac2100-config-20261118.tar.gz root@192.168.1.1:/tmp/
ssh root@192.168.1.1
```

在路由器确认归档后恢复。以下会覆盖现有配置并重启，**仅在确定要恢复时执行**：

```sh
tar -tzf /tmp/ac2100-config-20261118.tar.gz
sysupgrade -r /tmp/ac2100-config-20261118.tar.gz
reboot
```

`-r` 是恢复配置，不传入固件镜像。若目标是重新刷好的系统，要先准备相同型号兼容固件，并安装兼容的sing-box软件包；配置备份不会凭空恢复缺失的程序。不要为应急恢复盲装最新版、跨型号镜像或写mtd分区。

## 仅此次修复需要撤回

本次变更前的路由器私有目录：`/root/ac2100-before-20261118/`。修改前曾设置15分钟自动回滚，验收后已撤销。若目录仍完整，人工撤销本次变更可执行：

```sh
touch /root/ac2100-before-20261118/armed
sh /root/ac2100-before-20261118/rollback.sh
```

这会恢复变更前的init、透明链文件和升级保留清单，重建该透明链并重启sing-box，移除本次新加的两个脚本；不会清空整个防火墙。它撤销的是本次修复，不代表恢复为最佳配置，也不适合随意覆盖以后新的维护结果。刷机后该目录可能不在，应使用离机备份。

## 完全无法登录

先排除电源、网线和客户端地址；断开WAN后用网线LAN连接。若网络配置错误，按**该型号、该固件版本**的OpenWrt failsafe官方步骤处理。本文未现场演练物理按键时序或TFTP救砖，不提供未经验证的刷分区命令。`firstboot`/恢复出厂会清配置，不能作为第一步。配置备份不能修复损坏的引导器或校准分区。

## 恢复后逐项验收

路由器SSH：

```sh
fw4 check
sing-box check -c /etc/sing-box/config.json
nft list chain inet fw4 codex_singbox_prerouting
pid=$(pidof sing-box)
cat /proc/$pid/oom_score_adj
tr '\000' '\n' < /proc/$pid/environ | grep -E '^GO(GC|MEMLIMIT)='
sh /root/sing-box-healthcheck.sh
cat /tmp/sing-box-healthcheck.state
crontab -l
sysupgrade -l | grep -E 'sing-box-healthcheck|sing-box-protected|20-singbox-redirect|init.d/sing-box'
```

应有媒体来源return规则并逐渐增加计数、OOM值500、两个Go变量、检查退出码0及每10分钟cron。客户端再测试国内网络、Telegram、真实AI请求、Jellyfin播放和下载；观察同等流量下的CPU、RSS、FD与日志，不以一次200或瞬时CPU降低宣称长期稳定。

本次验证：规则命中、环境变量/OOM读回、健康检查连续通过，12项离线故障测试通过，Telegram302、ChatGPT200、Jellyfin健康接口200。未进行真实OOM、路由器重启或完整灾难恢复演练；未调整qB限速/强制做种，也未停止.109上的应用。
