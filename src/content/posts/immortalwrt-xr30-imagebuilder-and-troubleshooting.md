---
title: CMCC XR30 极简固件工程：官方 ImageBuilder 直出与 60 行 DTSO 链式注入
published: 2026-09-20
description: 摒弃 30GB 源码全量编译，基于 ImmortalWrt ImageBuilder 与增量 DTSO 实现零 Fork 周更固件；附 LocalSend 16.1KB 断流与 MTK PPE 死锁排查铁证。
tags: [OpenWrt, ImmortalWrt, MediaTek, Network, Firmware]
category: Router
draft: false
---

## 1. 架构选择：为什么拒绝源码 Fork？

传统全源码编译 OpenWrt 固件存在三大弊端：
1. 占用 30GB+ 磁盘空间，CI 全量构建耗时 1~2 小时；
2. 容易引入未经社区验证的私人补丁与隐蔽后门；
3. 与官方上游脱轨，升级维护成本极高。

**解法**：基于官方预编译 ImageBuilder，不 fork 源码、不全量重编，仅维护 62 行设备树增量 overlay（构建期 `fdtoverlay` 链式合成），其余 100% 保持与 ImmortalWrt 官方上游同步。

```
[官方 ImageBuilder 预编译内核/包仓库]
                 +
[62 行 mt7981b-cmcc-xr30-nand.dtso 增量补丁]  ─── fdtoverlay 链式注入 ───> 纯净产物 (3 分钟出包)
```

---

## 2. 引导链与双轨版本发布

- **引导链**：基于 `RSxiaoyu/bl-mt798x-xr30`，使用 Ubootmod（ATF SP2 + U-Boot 20250711），去 NMBM、支持 UBI 磨损均衡并自带 Web Failsafe 故障恢复控制台。
- **固件双通道**：
  - **`snapshot` 分支**：Linux 6.18 内核，每周日全自动 CI 构建，跟进最新驱动与内核特性；
  - **`25.12` 分支**：Linux 6.12 内核，追求长期稳定。

---

## 3. 网络排障实录：LocalSend 16.1KB 卡死与 MTK PPE 死锁

### 现象
局域网内手机与电脑互传文件时，LocalSend 固定卡死在 **16.1 KB** 或 **1.58 MB**，传输速率瞬间归零。

### 内核铁证抓取
通过 SSH 进入路由器后台，读取联发科 MT7981B 硬件加速引擎状态表：

```sh
cat /sys/kernel/debug/ppe0/entries | grep 53317
```

**抓取到的现场数据**：
```text
0144c BND IPv4 5T orig=192.168.1.175:34618->192.168.1.2:53317 ... bytes=1586104
0144d BND IPv4 5T orig=192.168.1.2:53317->192.168.1.175:34618 ... packets=67 bytes=4324
```

### 根因还原
事故由两层问题叠加导致：
1. **透明代理误劫持内网公网 IPv6**：  
   运营商下发的公网 IPv6（`2409:...`）被路由器上的代理插件误判为“境外公网流量”，暴力走 TPROXY 劫持进 Sing-box；而上游机场节点普遍无 IPv6 出站支持，握手后在第一个 TLS Record 上限（恰好为 **16 KB / 16,384 字节**）处直接死锁。
2. **MTK PPE / WED 硬件流控桥接缺陷**：  
   连接一旦持续产生大流量，Linux 内核判定为高速长连接并标记为 `BND`，交由硬件芯片卸载。MT7981B 在处理“有线 LAN 口”与“无线 Wi-Fi”之间的二层网桥大包时，硬件队列存在 TCP 序列号与 ACK 错位 Bug，导致滑动窗口堵死（卡死在 1.58 MB）。

### 极简原生解法
不编写任何底层复杂的 iptables 放行规则，直接通过原生开关处理：

```sh
# 1. 代理核心关闭 IPv6 代理
# 局域网传输、Moonlight/Sunshine 串流 100% 直通二层网桥，零劫持、零死锁

# 2. 保留 MTK PPE 硬件流控加速给 WAN 口
uci set firewall.@defaults[0].flow_offloading='1'
uci set firewall.@defaults[0].flow_offloading_hw='1'
uci commit firewall
/etc/init.d/firewall restart
```

**效果**：外网下载与千兆测速全由 PPE 硬件接管，CPU 占用率稳定在 **0%~3%**；内网设备间互传彻底脱离路由流控表，千兆无线内网跑满，死锁完全根除。
