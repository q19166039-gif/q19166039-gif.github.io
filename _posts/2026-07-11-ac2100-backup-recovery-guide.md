---
layout: post
title: "Redmi AC2100 OpenWrt 备份与恢复指南（含 Breed 刷机恢复）"
date: 2026-07-11 03:00:00 +0800
categories: [network, router]
tags: [ac2100, openwrt, sing-box, breed, backup]
---

## 备份文件说明

备份文件：`ac2100-backup-20260711.tar.gz`（大小约 3.9MB）

### 备份内容

| 文件 | 大小 | 用途 |
|------|:----:|:-----|
| `kernel.bin` | 4.0 MB | OpenWrt 内核，**Breed 可直接刷入 kernel 分区** |
| `factory.bin` | 256 KB | WiFi 校准数据 + MAC 地址，**Breed 可直接刷入 EEPROM 分区** |
| `bootloader.bin` | 512 KB | 当前 Bootloader（Breed），恢复用 |
| `configs/sing-box/config.json` | — | sing-box 完整配置（含 proxy-dns, sniff, 白名单规则） |
| `configs/sing-box/config.json.bak` | — | sing-box 旧配置备份 |
| `configs/dhcp` | — | dnsmasq DHCP/DNS 配置 |
| `configs/firewall` | — | 防火墙配置 |
| `configs/firewall.user` | — | 自定义防火墙规则 |
| `configs/sing-box.nft` | — | nftables 透明代理重定向规则 |
| `configs/etc-config/` | — | OpenWrt 所有 UCI 配置（network, wireless, system 等） |
| `firmware-info.txt` | — | 内核版本、OpenWrt 版本、已安装包列表 |

## 通过 Breed 恢复（完整刷机）

适用于：**路由器变砖、无法进入 OpenWrt、需要从 Breed 重新刷入系统**

### 前置条件

1. 路由器已在 Breed Web 恢复控制台
2. PC 有线连接路由器 LAN 口，IP 设为 `192.168.1.x`（如 192.168.1.100）
3. 浏览器访问 `192.168.1.1` 进入 Breed

### 恢复步骤

#### 第一步：刷入 kernel

- Breed → **固件更新** → **刷入 kernel 分区**
- 选择 `kernel.bin`（4MB）
- 点击 **上传** → **更新**

#### 第二步：刷入 factory（EEPROM）

- Breed → **固件更新** → **刷入 EEPROM 分区**
- 选择 `factory.bin`（256KB）
- 点击 **上传** → **更新**

> ⚠️ **重要**：factory 分区包含你的 Wi-Fi MAC 地址和射频校准数据，**务必保留**。如果刷别人的固件导致 Wi-Fi 信号弱或 MAC 变化，就是没刷这个分区。

#### 第三步：重启并上传完整 sysupgrade 固件

- Breed → **重启**
- 启动后进入 OpenWrt
- 进入 **系统 → 备份/升级 → 刷写新的固件**
- 上传 OpenWrt 23.05.5 完整 sysupgrade 固件

**从备份恢复配置**：
- 系统 → 备份/升级 → 上传备份
- 选择备份文件中的 `configs/` 目录文件逐一恢复
- 或者通过 SCP 手动恢复 `/etc/config/` 下的配置文件

### 如果 kernel.bin 也无法启动

需要重新下载官方 OpenWrt 固件：

```
固件: openwrt-23.05.5-ramips-mt7621-xiaomi_redmi-router-ac2100-squashfs-sysupgrade.bin
下载: https://downloads.openwrt.org/releases/23.05.5/targets/ramips/mt7621/
```

刷入方式：
- Breed → **固件更新** → **刷入 kernel 分区** → 选择官方 kernel
- 或 Breed → **固件更新** → **固件** → 选择完整 sysupgrade 固件（Breed 会自动将其写入正确分区）

## 配置恢复步骤（OpenWrt 仍可启动的情况）

如果 OpenWrt 系统正常，只是需要恢复配置：

```bash
# 备份文件已解压到路由器
cd /tmp
scp user@your-pc:/path/to/ac2100-backup-20260711.tar.gz ./

# 解压
tar xzf ac2100-backup-20260711.tar.gz
cd ac2100-backup/configs

# 恢复 sing-box
cp -r sing-box /etc/
cp sing-box.nft /usr/share/nftables.d/chain-post/dstnat/

# 恢复防火墙
cp dhcp /etc/config/
cp firewall /etc/config/
cp firewall.user /etc/

# 重启服务
fw4 reload
/etc/init.d/sing-box restart
/etc/init.d/dnsmasq restart
```

## 当前系统信息

- **OpenWrt**: 23.05.5 (r24106-10cc5fcd00)
- **Kernel**: 5.15.x
- **Sing-box**: 1.11.15 (mipsle, VMess+WS 代理)
- **代理模式**: 透明代理（nftables redirect TCP → sing-box:7892）
- **DNS 方案**: 
  - 代理域名（Google/YouTube/GitHub 等）：8.8.8.8 走荷兰 VPS 解析
  - 国内域名：223.5.5.5 (AliDNS)
- **VPS**: 185.239.70.197:8080 (VMess+WebSocket, Host: www.bing.com)
- **内存占用**: 可用约 51MB / 128MB
- **端口**: DNS 53 (dnsmasq) / 透明代理 7892 (redirect) / SOCKS5 7893 (mixed)

### 代理白名单

Google / YouTube / GitHub / Telegram / Netflix / Twitter/X / ChatGPT / Grok / HuggingFace / Pornhub / XVideos

## 分区布局（MT7621 128MB SPI Flash）

| 分区 | 大小 | 说明 |
|:----|:----:|:------|
| Bootloader (mtd0) | 512KB | Breed |
| Config (mtd1) | 256KB | U-boot env |
| Bdata (mtd2) | 256KB | Boot data |
| factory (mtd3) | 256KB | **MAC+WiFi校准** ⚠️ |
| crash (mtd4) | 256KB | Crash dump |
| crash_syslog (mtd5) | 256KB | Crash log |
| reserved0 (mtd6) | 256KB | 保留 |
| kernel_stock (mtd7) | 4MB | 出厂内核备份 |
| kernel (mtd8) | **4MB** | **当前 OpenWrt 内核** |
| ubi (mtd9) | ~117MB | 根文件系统 + overlay |

> Breed 可以刷入的分区：kernel（mtd8）、factory/EEPROM（mtd3）、Bootloader（mtd0）、以及整个固件（kernel + rootfs 打包的 sysupgrade 固件）
